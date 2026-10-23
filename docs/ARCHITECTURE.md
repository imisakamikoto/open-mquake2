# 架构设计

## 分层

```text
┌─────────────────────────────────────────────────────┐
│ Flutter UI（Riverpod + flutter_map，Leaflet 式图层） │
├─────────────────────────────────────────────────────┤
│ Dart Services（HTTP / WebSocket / 音频 / TTS / 设置）│
├───────────────────── dart:ffi 桥接 ───────────────────┤
│ Rust 核心（geo / map / seismic / geojson）           │
└─────────────────────────────────────────────────────┘
```

## 为什么这样分层

旧版把所有逻辑塞进浏览器端 JavaScript，存在两个问题：

1. **单文件巨石**：`index.js` 约 8300 行，`getTravelTime`、
   `iclWaveDraw`、`SnetRedraw`、`fetchSnetImageFromMsil` 等函数被重复定义，
   修改一处容易漏掉另一处。
2. **计算与渲染耦合**：震相走时、震中定位、等值线、坐标投影都混在 Leaflet
   回调里，无法单独测试，也难以在移动端复用。

地图本身保留 Leaflet 的模型（`flutter_map` 是 Leaflet 的 Dart 实现），
因为 1.0 的地图表现力完全够用、也便于逐项对齐；被替换掉的只是"把计算塞进
渲染回调"这件事——所有数值计算都搬到 Rust，渲染只负责画。

新架构把「纯计算」和「有副作用的 I/O」彻底分开：

- Rust 核心是无状态、无 I/O 的函数库，输入输出都是可序列化的 JSON。这样
  震相走时、震度换算、震中定位、渲染指令生成都可以用 `cargo test` 覆盖。
- Dart 层负责所有 I/O，使用成熟的 `http`、`web_socket_channel`、`just_audio`
  等包，把并发、重连、缓存这些易变逻辑留在 UI 语言里。

## 桥接约定

Rust 编译为动态库，通过 `dart:ffi` 暴露少量 C ABI 函数：

```rust
pub extern "C" fn mquake_travel_time(
    core: *mut c_void, depth_km: f64, distance_km: f64, wave: c_int,
) -> f64

pub extern "C" fn mquake_wave_radius(      // 波面半径（米）
    core: *mut c_void, origin_ms: i64, now_ms: i64,
    depth_km: f64, wave: c_int,
) -> f64

pub extern "C" fn mquake_forecast_jma(     // 行政区最大震度
    json: *const u8, len: usize,
) -> *mut c_char
```

好处：桥接面只包含基础类型，不需要维护跨语言结构体镜像；模型变更只影响
两侧各自的 JSON 模型。有状态数据（如解析后的走时表）由 `Core` 句柄持有，
Dart 侧创建一次后复用。

## 领域模型

所有模型都定义在 `rust/src/model/`，并在 Dart 侧有对应等价物：

- `EarthquakeEvent`：震中、震级、深度、烈度、机构、报数等。
- `EewEvent`：紧急地震速报，含 PLUM 推定、取消报。
- `TsunamiAlert`：海啸警报。
- `Station`：各类测站（K-NET / KiK-net / S-net / CENC / Palert / KMA…）。

