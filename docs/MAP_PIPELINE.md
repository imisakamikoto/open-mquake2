# 地图实现（与 mquake1.0 对齐）

地图改用 **Leaflet 式地图模型**：Flutter 侧用 `flutter_map`（Leaflet 的 Dart
实现），因此可以直接沿用 1.0 里的 Leaflet 概念与参数；Rust 侧只负责计算
（走时/波半径、JMA 震度、地名反查），不再生成渲染指令。

对应关系：

| mquake1.0（Leaflet） | mquake2（flutter_map） |
| --- | --- |
| `L.map('map', {...})` | `FlutterMap(options: MapOptions(...))` |
| `L.geoJSON(data, {style})` | `PolygonLayer` / `PolylineLayer` |
| `L.circle([lat,lon], {radius})` | `CircleMarker(useRadiusInMeter: true)` |
| `L.circleMarker([lat,lon], {radius: 3})` | `CircleMarker(radius: 3)` |
| `L.marker([lat,lon], {icon: divIcon})` | `Marker(child: ...)` |
| `map.setView / fitBounds / setRadius` | `MapController.move / fitCamera` + 每秒重建半径 |
| `map.createPane(name) + zIndex` | `FlutterMap.children` 的先后顺序 |
| `layer.bindPopup / bindTooltip` | `onTap` / `LayerHitNotifier` → 自绘弹窗 |

## 相机与交互（对齐 1.0 的 `L.map` 参数）

- 初始中心 `[38.6329231, 138.4916350]`、缩放 6、`minZoom 3`、`maxZoom 10`；
- `worldCopyJump / continuousWorld` → 相机不设限，中心经度超出 ±180° 时折回
  （`map_view.dart::_onPositionChanged`），实现左右无缝环绕；
- 拖拽、滚轮缩放、双击放大、左上角缩放控件（Leaflet 默认控件样式）；
- `preferCanvas: true` → flutter_map 本身即 canvas 渲染。

## 图层顺序（对齐 1.0 的 pane z-index）

| 顺序 | 图层 | 1.0 pane | z-index |
| --- | --- | --- | --- |
| 1 | 海洋（整块背景色，**纯黑 `#000000`**） | baseGeoPane 底色矩形 | 200 |
| 2 | 世界陆地 **灰 `#3C3C3C`** + 岸线/国界 **暗灰 `#5A5A5A`**（宽 1） | baseGeoPane + SEKAI.geo.json | 200 |
| 3 | 中国断层 `#8D6F64`（**最细 0.8**）；板块边界与台风警戒线已从渲染栈移除 | baseGeoPane1 | 205 |
| 4 | 行政区震度分布（暗灰描边 0.5 + 震度色填充） | baseGeoPane2 | 209 |
| 5 | 震中 + P/S 波圈（quakePane） | quakePane | 350 |
| 6 | 海啸岸段（宽 5、闪烁） + 测站圆点与震度图标 | overlayPane | 400 |

## 关键文件

- `map/map_view.dart`：地图主体。相机参数、图层顺序、缩放控件、弹窗、
  世界环绕、视口与 `viewportProvider` 的双向同步、回到震中。
- `map/layers/static_layers.dart`：静态要素（陆地 / 板块 / 断层 / 台风线 /
  海啸岸段 / 行政区），一次性加载 + 按缩放级别做几何简化（LOD）缓存。
- `map/layers/quake_layers.dart`：震中图标（`img/Source-Copy.png`）、推定
  震源橙色空心圆、P/S 波圈（半径来自 Rust `mquake_wave_radius`）、按 vsync
  逐帧扩张；自动取景取「P 波前沿 ∪ 预计震度 ≥1 区域」的联合包围盒。
- `map/layers/shake_detection_layers.dart`：揺れ検知站群的闪烁经纬度格网。
  格宽按站群跨度自动选 0.5° / 1° / 2° / 5°，格线对齐整度，不叠加标题、
  坐标文字或测站独立环；官方 EEW 生效时隐藏。新站群出现或范围扩大 18%
  以上时，镜头平滑聚焦到格网范围。
- `map/layers/station_layers.dart`：测站圆点（半径 3、`calclocalshindocolor`
  区间中点色、不透明度 0.95）+ 22px 震度图标（等价 `shindopng/*.svg`，
  最多前 800 个）+ 「预警时仅显示震度图标」。
- `map/layers/tsunami_layers.dart`：按等级着色（`tsunamiColors` /
  `chinaTsunamiColors`）并 800ms 闪烁，仅在生效时出现（`tsunami_gate.dart`）。
- `map/layers/admin_intensity_layers.dart`：行政区震度分布，算法与 1.0 的
  `adminIntensity.worker.js` 完全一致（震中在多边形内取震中点，否则取边界
  最近点；有效半径 ×1.8 外记 0；多事件按行政区取最大）。
- `map/layers/tile_config.dart`：可选底图（在线 XYZ 瓦片），默认关闭。
- `map/map_style.dart`：所有配色的唯一来源，逐条对应 1.0 的色表。
- `map/geo_utils.dart`：GeoJSON 解析、距离、最近点、点在多边形内、RDP 简化。
- `map/map_auto_view.dart`：自动视角保护（用户操作 1.5s 内不抢视角、同一
  目标 10s 节流），相机移动用 Ticker 插值，对应 1.0 的
  `setViewSafely / fitBoundsSafely`。

## 性能

- 陆地（143k 顶点）与断层（48k 顶点）按缩放级别用 RDP 简化后缓存，
  容差 ≈0.6 像素（0.0008°–0.003°）；
- 行政区（45 万顶点、1119 个区）只在有生效预警时绘制，且逐级简化 +
  视口包围盒裁剪 + 数量上限 900；
- P/S 波由 `Ticker` 按显示器刷新率驱动，只通过 `ValueNotifier` 重绘波圈
  `CustomPainter`，不重建底图 / 测站 / 城市名；
- 每个图层用单独的 `RepaintBoundary` 隔离，波圈动画期间静态图层复用 GPU 纹理；
- Windows 启动时指定高性能 GPU，渲染器使用引擎默认稳定路径；
- 低功耗模式只拉长数据轮询间隔，不降低波面帧率。

## Vs30 / 地盘增幅率（Arv）

- 数据源：`global_vs30.grd`（30″ 全球 Vs30 网格）与
  `app/assets/Resource/neweewgeo.json`（行政区多边形）；
- 每个行政区按多边形内网格做面积加权平均，得到代表性 Vs30（同时保留
  `vs30_max` 供对照），按
  `Arv600 = clamp((600 / Vs30)^(2/3), 0.5, 2.5)` 转换；
- Vs600 → Vs400 工程基盘使用标准给出的 `1.31` 倍；地表增幅率按
  `Arv400 = Arv600 / 1.31` 计算；
- 工程基盘 PGA 使用司・翠川(1999) PGA 衰减式后除以 1.4；PGV 使用
  司・翠川(1999) Vs600 衰减式；地表 PGV 为
  `PGV400 × (Arv600 / 1.31)`；计测震度使用
  `2.68 + 1.72 log10(PGV_surface)`；
- 应用读取 `app/assets/Resource/admin_vs30_arv.json`，在行政区震度预估时把
  对应的 `Arv` 传给 Rust/Dart 的 JMA 震度公式；
- 可读的完整清单在 `docs/admin_vs30_arv.csv`；
- 重新生成：

  ```powershell
  $env:PYTHONPATH = "tools/pydeps"
  python tools/build_vs30_arv.py
  ```

## 已弃用（不再使用）

- Dart：`map_painter.dart`、`scene_builder.dart`、`RenderCommand` /
  `MapScene` / `SceneLayer` / `LayerStyle`、`bridge.dart` 里的
  `renderFrame / projectPoints / contour` 绑定；
- Rust：`src/map/render.rs`、`src/map/spatial.rs`、`src/map/simplify.rs`
  以及 `api.rs` 中对应的 C ABI 入口（源码保留以便回退，DLL 未变）。

## 尚未移植的 1.0 地图特性

- K-NET 摇晃检出的闪烁矩形（`knetShakeLayer` / `createShakeRectangle`）
  与「按网格自动聚焦」（`tryAutoFocus`）；
- 震度等值线（`drawIntensityContour` / `drawCencIntensity`）与 JMA 区域
  预报图层的自动缩放；
- 台湾 / Wolfx / Palert / Raspberry Shake 测站图层（1.0 的 `twStationLayer`
  等），以及台风路径（`drawTyphoon` 的历史路径与预报点）；
- 海啸音效（`sounds/tsunami_*.wav`）。

