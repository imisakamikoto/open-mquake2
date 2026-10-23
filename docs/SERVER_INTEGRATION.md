# 服务端内置方案

旧版前端依赖部署的 Node 服务（`all.js` 的 `/getapi` 与 `/ws`，以及
`cencking.js`、`joho.js`、`globalquake.js` 等子服务）。新版把这些能力直接
做进应用，**不再需要部署服务器**：

```text
旧：Flutter ──> https://abc-1.misakabosai.qzz.io/getapi（部署的 joho.js，多机构列表）
              └> http://154.9.252.189:3000/getapi（部署的 cencking.js，CENC 速报）
              └> ws://allapi.misakabosai.qzz.io/ws

新：Flutter ──> AggregationService（应用内）
                   ├─ Rust 计算（震度 / 地名反查 / 走时）
                   └─ 直连上游数据源
```

## 地震列表 —— 对应 `joho.js` 的 `/getapi`

1.0 右侧的「主事件 + 子事件 + 机构名称」列表来自 `joho.js`（部署在
`abc-1.misakabosai.qzz.io`），并不是来自 `all.js`。新版把这 8 个机构 + FSSN
全部内置（`app/lib/src/services/aggregation/joho_sources.dart`），每 3 秒全量
刷新一次（与 joho 的 `setInterval(fetchAndUnifyData, 3000)` 一致）：

| 机构（`Source`） | 上游 | EventID | 过滤 |
| --- | --- | --- | --- |
| `USGS` | `earthquake.usgs.gov/.../summary/4.5_week.geojson` | `properties.code` | 取前 100 条 |
| `JMA` | `www.jma.go.jp/bosai/quake/data/list.json` | `eid` | 只留 `ttl === 震源・震度情報` |
| `GeoNet` | `api.geonet.org.nz/quake?MMI=1` | `publicID` | M≥3.5 |
| `CWA` | `api.core.exptech.dev/api/v2/eq/report` | `id` | 取前 50 条 |
| `TMD` | `earthquake.tmd.go.th/feed/rss_tmd.xml` | link 末段 | 取前 50 条 |
| `CENC` | `api.dizhensubao.getui.com/api.htm`（POST CEIC） | `eqid` | 近 15 天，取前 100 条 |
| `BJDZJ` | `bjdzj.gov.cn/eportal/ui?...dzsb-front` | `cataId` | — |
| `GXDZJ` | `gxdzj.gov.cn/dzjGxdzMappage.jspx`（HTML 表格） | `GXDZJ_时间_经度_纬度` | 取前 10 行 |
| `FSSN` | `wss://ws.fanstudio.tech/fssn`（备用 `.hk`） | `id` | 缓存 50 条 |

合并规则照抄 joho.js：

1. `deduplicateCencData`：EventID 含 `CENC` / `CD.` / `AU.` 的记录按
   「经纬度 ≤0.5°、时间差 ≤10 分钟、深度差 ≤10km」聚类，组内保留人工测定
   （`Info === 'M'` 或不含 `[A]`）的最新一条；
2. `removeDuplicates`：全体按「地名 + 经纬度两位小数 + 发震时间到分钟」去重；
3. 按发震时间倒序。

唯一的有意偏差：joho 在任一机构请求失败时会**整轮跳过**（用上一轮完整缓存），
新版改成**按机构**保留上一次成功结果，避免一个被墙的机构让整个列表停更。

## 预警（EEW）—— 对应 `all.js` 的 `/ws`

| 来源 | 方式 | 对应旧文件 |
| --- | --- | --- |
| 成都高新减灾研究所（chinaeew） | HTTP | `all.js parseApi2Data` |
| ExpTech trem | HTTP | `all.js fetchTremData` |
| 中国地震台网（CENC 微信多账号） | HTTP POST，31 个账号轮询注册 | `cencking.js` |
| 日本气象厅紧急地震速报（Wolfx 转发） | WebSocket `wss://ws-api.wolfx.jp/jma_eew` | 替代原先的 DMDATA 接入 |
| 韩国气象厅（KMA） | WebSocket | `all.js connectKmaWebSocket` |

这些事件进入预警缓存，与列表里「最近 1 分钟」的事件一起决定预警条 / 震波圈
（对应 1.0 `iclEvents` 的 60 秒自动取消逻辑）。

## Rust 侧新增计算

- 震度：`max_intensity_wechat`（CENC 公式）、`max_shindo_jma` / `shindo_at_point`
  （PGV600→700→IINSTR→震度文本）、`fmt_shindo_to_text`。
- 地名反查：`geocode.rs` 的 point-in-polygon + 统一地名（与 `all.js unifyHypoCenter`
  逻辑一致），地名库由 `app/data/geodata/` 加载。

## 仍需外部程序/凭据的部分（可选）

- `globalquake.js` 依赖外部 `gq2ws`（`ws://localhost:2857`），未内置；
  GlobalQuake 来源默认关闭。
- JMA 紧急地震速报原先走 DMDATA 官方 WebSocket（需要 Client ID/Secret 申请与
  有效期），现已改为 Wolfx 转发的 `wss://ws-api.wolfx.jp/jma_eew`，免凭据。
  客户端实现见 `lib/src/services/aggregation/wolfx_eew.dart`；收到 `heartbeat`
  会回 `ping` 保活，`isTraining` 报文被忽略。
- `niedst/`（K-NET/S-net 图片）、`NM/NMEFC`（海洋预报）是图层数据服务，
  可后续作为额外图层接入。

## 数据文件

`app/data/geodata/` 已从 `Misaka-server-all-main` 拷入：

```text
jp.geojson              ← 震央地名.geojson
cenc_regions.geojson    ← cenc_regions.geojson
province_regions.geojson← province_regions.geojson
counties/*.json         ← counties/counties/*.json（2884 个县）
cenc_apps.json          ← 从 cencking.js 提取的 31 个 CENC 账号配置
```

