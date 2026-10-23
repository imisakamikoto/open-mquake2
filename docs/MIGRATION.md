# 原版功能映射

| 原版（index.js） | 新版实现 |
| --- | --- |
| `parseTravelTimeData` / `getTravelTime` / `getWaveRadius` | `seismic/travel_time.rs`（双线性插值 + 二分反算，不再重复定义） |
| `calcMaxInt`（index.html） | `seismic/intensity.rs::max_intensity` |
| `pgaToShindo` | `seismic/intensity.rs::pga_to_shindo` |
| `calclocalshindocolor` / `calclocalpgacolor` | `color.rs`（连续色带，替代长 if-else） |
| `intColor` / `intColorJma` | `color.rs::INTENSITY_COLORS` / `JMA_COLORS` |
| Leaflet 地图 | `app/lib/src/map/`（flutter_map，图层/配色/交互逐项对齐 1.0，见 [docs/MAP_PIPELINE.md](MAP_PIPELINE.md)） |
| `iclDataGet`（WS `ws://allapi.misakabosai.qzz.io/ws`） | `services/eew_service.dart` |
| `getData`（`https://abc-1.misakabosai.qzz.io/getapi`） | `services/data_service.dart` |
| `connectEewWebSocket`（p2pquake） | `services/eew_service.dart` |
| `all.js /getapi` 与 `/ws`（部署服务端） | `services/aggregation/`（应用内聚合，无需部署） |
| `cencking.js`（CENC 微信多账号） | `services/aggregation/cenc_source.dart` |
| `joho.js` / `all.js` 震度与地名 | `rust/src/seismic/intensity.rs` + `rust/src/geocode.rs` |
| `playSoundForEvent` | `services/audio_service.dart` |
| `initTts` / `playEewTts` | 可选扩展（`flutter_tts`，本骨架暂未启用） |
| 各测站图层（K-NET/S-net/CENC/Palert/KMA/TW/Wolfx/RS） | `services/data_service.dart` + 统一 `StationLayer` |
| `drawIntensityContour` / `drawCencIntensity` | `map/contour.rs` + 烈度图层 |
| `updateTsunamiLayer` / `updateChinaTsunamiLayer` | `services/data_service.dart` + 海啸图层 |
| `fetchTyphoons` / `drawTyphoon` | `services/data_service.dart` + 台风图层 |
| `estimateEpicenterForTriggers`（网格搜索） | `seismic/epicenter.rs` |

## 地图功能对照（1.0 → 2.0）

| 1.0 地图元素 / 行为 | 2.0 实现 | 状态 |
| --- | --- | --- |
| 世界陆地 + 海岸线（`SEKAI.geo.json`，`#373e34` / `#d7def8`） | `static_layers.dart` + `PolygonLayer` | ✅ |
| 海洋底色（`MAP_OCEAN_COLOR`）与世界环绕 | 地图背景色 + 相机经度折回 | ✅ |
| 板块边界 / 中国断层 / 台风警戒线 | `static_layers.dart`（同配色、同线宽） | ✅ |
| 震中图标 `Source-Copy.png`、推定震源空心圆 | `quake_layers.dart` | ✅ |
| P/S 波圈（走时表半径、每秒扩张、`fitWaveBounds`） | `quake_layers.dart` + Rust `mquake_wave_radius` | ✅ |
| 新事件自动聚焦（`fitBoundsByEventOnly` / `setViewSafely`） | `quake_layers.dart` + `map_auto_view.dart` | ✅ |
| 测站圆点 + `shindopng` 震度图标 + 站点弹窗 | `station_layers.dart` + `MapView` 弹窗 | ✅ |
| 「预警时仅显示震度图标」（`hideStationOnEew`） | `station_layers.dart` | ✅ |
| 行政区震度分布（`adminIntensity.worker.js`） | `admin_intensity_layers.dart`（同算法，Rust/Dart 双实现） | ✅ |
| 海啸岸段按等级着色 + 闪烁 | `tsunami_layers.dart` | ✅ |
| 陆地 / 断层弹窗（`bindPopup`） | `LayerHitNotifier` + `MapView` 自绘弹窗 | ✅ |
| 缩放控件、双击放大、滚轮缩放、回到震中 | `map_view.dart` | ✅ |
| K-NET 摇晃闪烁矩形、`tryAutoFocus` 网格聚焦 | — | ⬜ 未移植 |
| 震度等值线、JMA 区域预报图层自动缩放 | — | ⬜ 未移植 |
| 台湾 / Wolfx / Palert / RS 测站图层、台风路径 | — | ⬜ 未移植 |
| 海啸音效（`sounds/tsunami_*.wav`） | — | ⬜ 未移植 |

## 数据源（保持不变，集中配置）

```text
地震速报 WS     ws://allapi.misakabosai.qzz.io/ws
地震列表        https://abc-1.misakabosai.qzz.io/getapi
p2pquake WS     https://api.p2pquake.net/v2/ws
海啸            https://tsunami.misakabosai.qzz.io/api/tsunami-alerts
K-NET           https://nied.misakabosai.qzz.io/api/knet-data
中国烈度        http://cencint.misakabosai.qzz.io/api/earthquake/intensity-contour
...（其余见 app/lib/src/config.dart）
```

