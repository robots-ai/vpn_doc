# 微风网络 (wfnetwork) 架构与开源模块扫描报告

> **分析对象**：本机已安装的"微风网络"代理客户端（DisplayName "微风网络"，版本 3.0.0，build `2026071103`）
> **安装路径**：`C:\Users\zhangle_china\AppData\Local\Programs\wfnetwork\`
> **数据路径**：`C:\Users\zhangle_china\AppData\Roaming\wfnetwork\wfnetwork\`
> **分析日期**：2026-10-30
> **分析方式**：只读静态分析 —— 目录枚举、二进制可打印字符串抽取、Go/Rust 依赖还原、Flutter `NOTICES` 许可证清单解压、本地 SQLite schema 读取。全程未修改任何软件文件。

---

## 1. 总体架构（四层）

```
┌─────────────────────────────────────────────────────────────────┐
│  层1  界面层   Flutter (Dart AOT)                                  │
│  wfnetwork.exe (壳 469KB) + data/app.so (13MB Dart 快照)          │
│  状态管理 flutter_riverpod · 主题 dynamic_color (Material You)     │
├─────────────────────────────────────────────────────────────────┤
│  层2  系统辅助  Rust 服务（自研封装）                              │
│  wfnetworkHelperService.exe (827KB) — warp/tokio HTTP 服务         │
│  暴露 ping / start / stop / logs 接口，供 UI 执行提权操作          │
├─────────────────────────────────────────────────────────────────┤
│  层3  代理内核  mihomo (Clash.Meta / MetaCubeX)   [开源]           │
│  wfnetworkCore.exe (34.9MB, Go 1.25, sing 库)                     │
│  mixed-port 7892 · DNS :1053 · TUN 设备名 "wfnetwork"             │
├─────────────────────────────────────────────────────────────────┤
│  层4  后端 + 数据                                                  │
│  api02.weifengapi.com (账号/订阅/订单)                             │
│  configoss.weifengapi.com (阿里云 OSS，配置下发)                  │
│  本地：config.yaml(加密) · database.sqlite · shared_preferences    │
└─────────────────────────────────────────────────────────────────┘
```

**一句话总结**：标准 mihomo 内核 + Rust 提权辅助服务 + Flutter 商业订阅客户端 + 自有后端（`weifengapi.com`）。

---

## 2. 组件详情

### 2.1 代理内核 —— mihomo（开源，未魔改）

- 主程序 `wfnetworkCore.exe` 即标准 **mihomo (Clash.Meta / MetaCubeX)**，仅重命名。
- 编译信息：Go 1.25.0，构建主模块为 `devel`，版本号未以明文字符串固化（ldflags 注入），无法静态读出精确小版本。
- 关键运行参数（从应用配置还原）：
  - `mixed-port: 7892`，`mode: rule`，`log-level: error`，`ipv6: true`
  - `find-process-mode: always`，`tcp-concurrent: true`，`unified-delay: true`
  - `dns`: `enable: true`，监听 `0.0.0.0:1053`
  - `tun`: 设备名 `wfnetwork`，`stack: mixed`，`dns-hijack: any:53`

### 2.2 系统辅助服务 —— Rust（自研封装，借开源 crate）

`wfnetworkHelperService.exe` 是 **Rust** 编写的 Windows 服务，用 warp 起一个本地 HTTP 服务，暴露 `ping / start / stop / logs` 接口，让非提权的 Flutter UI 完成需要管理员权限的操作（系统代理、TUN、内核启停）。

### 2.3 界面 —— Flutter（自研 UI，大量使用开源包）

`wfnetwork.exe` 是 Flutter 壳，业务逻辑编译进 `data/app.so`（Dart AOT 快照）。

### 2.4 后端与数据

- **订阅下发**：`https://api02.weifengapi.com/api/v1/client/subscribe?token=<redacted>`
- **配置下发**：`http://configoss.weifengapi.com/oss/ConFigOss.json`（阿里云 OSS）
- **更新/下载**：`168downloads.oss-ap-northeast-2.aliyuncs.com`（阿里云 OSS，首尔区域）
- **本地存储**：
  - `config.yaml` —— 生成的 mihomo 配置，**加密/混淆存储**（base64 形式的密文，非明文 YAML）
  - `profiles/999999.yaml` —— 订阅 profile，**加密存储**（二进制密文）
  - `database.sqlite` —— 订阅/规则/脚本本地库
  - `shared_preferences.json` —— 账号、登录态、应用设置
  - `GEOIP.dat / GEOIP.metadb / GEOSITE.dat / ASN.mmdb` —— 运行时路由数据

---

## 3. 功能点 → 模块映射

| 功能点 | 实现方式 | 依赖模块/库 |
|---|---|---|
| 代理内核 | mihomo 原版（重命名） | mihomo (MetaCubeX) + sing 库 |
| 界面框架 | Flutter 桌面 | Flutter + flutter_riverpod |
| 动态取色主题 | Material You | dynamic_color |
| 账号登录/注册/找回 | 邮箱+密码 → JWT | 后端 `/passport/auth/*`，令牌存 shared_preferences |
| 订阅管理 | 拉取加密 profile | 后端 `/api/v1/client/subscribe` + drift/sqflite |
| 分流规则/脚本 | 规则/脚本/映射 + 本地编辑 | database.sqlite + re_editor |
| 路由数据 | 规则引擎 | GEOIP/GEOSITE.dat + ASN.mmdb |
| TUN 虚拟网卡 | 虚拟网卡模式 | mihomo tun（设备名 "wfnetwork"） |
| 系统代理 | 全局代理 + 绕过名单 | 自研 proxy_plugin.dll |
| DNS | 劫持/自定义 DNS (:1053) | mihomo dns + overrideDns |
| 付款/套餐 | 订单+支付+激活 | dio + flutter_inappwebview |
| 优惠券/邀请 | 折扣 + 邀请返利 | 后端 `/user/coupon/*`、`/user/invite/*` |
| 在线客服 | Crisp 聊天窗 | crisp_chat |
| 二维码 | 订阅/支付二维码 | qr_flutter |
| 流量统计 | 上传/下载/配额面板 | 后端 subscription_info |
| 连通性测试 | 延迟测试 | testUrl `cp.cloudflare.com/generate_204` |
| 自启动 | 开机自启 | launch_at_startup |
| 崩溃上报 | crashlytics | firebase（默认关闭） |
| 配置加密 | 本地 config/profile 加密 | pointycastle (AES) |
| JS 执行 | 订阅解码/脚本 | flutter_js + quickjs |
| 内嵌网页/WebView | 支付页、网页登录 | flutter_inappwebview (WebView2) |
| 网络请求 | 与后端通信 | dio |
| 图片缓存 | 节点国旗/头像 | cached_network_image / octo_image |

---

## 4. 开源模块清单

### 4.1 内核层（Go / mihomo 生态）

**主程序**：`github.com/metacubex/mihomo`（Clash.Meta）。

**MetaCubeX 维护的 fork 库（占绝对主导）**：
`chi`(HTTP 路由)、`tls`/`utls`(TLS 指纹)、`quic-go`、`sing-tun`/`sing-quic`/`sing-vmess`/`sing-mux`、`smux`/`yamux`(多路复用)、`bbolt`(存储)、`blake3`/`mlkem`/`hpke`/`ascon`(密码学)、`kcp-go`、`qpack`、`fswatch`、`randv2`、`tfo-go` 等。

**SagerNet sing 家族**：协议实现（sing-box 同源）。

**其它三方**：`miekg/dns`、`klauspost/compress`、`gobwas/ws`(WebSocket)、`vmihailenco/msgpack`、`andybalholm/brotli`、`oschwald/geoip2`(MaxMind)、`enfein/mieru`(mieru 协议)、`oasisprotocol/curve25519`、`sirupsen/logrus`、`insomniacslk/dhcp`、`golang.org/x/*` 等。

### 4.2 辅助服务（Rust）

从二进制字符串还原的 crate 清单：
**tokio · warp · hyper · h2 · http · serde_json · anyhow · bytes · mio · httparse · futures · indexmap · hashbrown · once_cell · parking_lot · mime · slab · generic-array**

### 4.3 界面层（Dart/Flutter 应用层包）

`dio` `flutter_riverpod` `dynamic_color` `drift`/`drift_flutter` `sqflite`/`sqlite3` `pointycastle` `flutter_inappwebview` `desktop_webview_window` `flutter_js` `crisp_chat` `qr_flutter` `re_editor` `file_picker` `image_picker` `cached_network_image` `octo_image` `flutter_cache_manager` `flutter_html` `flutter_svg` `launch_at_startup` `screen_retriever` `tray_manager` `window_manager` `win32`/`win32_registry` `connectivity_plus` `isolate_contactor`/`isolate_manager` `synchronized` `uuid` `shortid` `yaml` 等。

### 4.4 内置原生库 / 数据资产

- **Flutter 引擎 / Skia 依赖**：Skia、ICU、brotli、harfbuzz、freetype2、libpng、libjpeg-turbo、libwebp、libjxl、rapidjson、re2、angle、vulkan 等
- **SQLite**（本地数据库）、**WebView2**、**QuickJS**、**MSVC 运行库**
- **路由数据**：GEOIP/GEOSITE.dat（MetaCubeX 规则集）、ASN.mmdb（MaxMind）

### 4.5 疑似自研（NOTICES 许可证清单中无对应项）

- `proxy_plugin.dll` —— 系统代理设置
- `window_ext_plugin.dll` —— 窗口扩展
- `wfnetworkHelperService.exe` 业务逻辑（hub/windows 服务封装）
- 账号/订阅/订单整套后端协议，以及本地 AES 配置加密方案

---

## 5. 付款模块专项

**订单接口**（从 `app.so` 还原）：

| 接口 | 作用 |
|---|---|
| `/user/order/checkout` | 创建订单（传 planId / orderAmount / coupon） |
| `/user/order/getPaymentMethod` | 获取可用支付方式 |
| `/user/order/check` | 查询支付状态（配合 `checkpaymentstatus`） |
| `/user/order/detail` / `fetch` / `cancel` | 订单详情 / 列表 / 取消 |
| `/user/plan/fetch` / `fetch2` | 拉取套餐 |
| `/user/coupon/check` | 校验优惠券 |

**支付流程**：
选套餐 → 创建订单 → 返回 `paymentUrl` → 用 `flutter_inappwebview`(WebView2) 打开网页收银台 → 轮询支付状态 → 成功自动激活（"Order completed, service opened"）。

**计费要素**：
- 套餐周期：月付 / 半年付 / 两年付 / 三年付 / 一次性 / 试用（`tryOutPlanId`）
- 价格字段：`month_price`、`half_year_price`、`two_year_price`、`three_year_price2`、`onetime_price`、`originalPrice`、`finalPrice`
- **Apple Pay**（`applePayAPIEnabled`）
- 优惠券抵扣（`Coupon deduction`、`coupon_id` / `coupon_code`）
- 邀请返利（`/user/invite/*`、`inviteUserId`、`surplus_order_ids`）
- 30 分钟未支付自动取消订单

---

## 6. 后端 API 清单

**认证**：`/passport/auth/login`、`/passport/auth/register`、`/passport/auth/forget`
**用户**：`/user/info`
**邀请**：`/user/invite/fetch`、`/user/invite/details`、`/user/invite/save`
**订单**：`/user/order/checkout`、`/user/order/check`、`/user/order/detail`、`/user/order/fetch`、`/user/order/cancel`、`/user/order/getPaymentMethod`、`/user/order/save2`
**套餐**：`/user/plan/fetch`、`/user/plan/fetch2`
**优惠券**：`/user/coupon/check`
**配置**：`/comm/config`、`configoss.weifengapi.com/oss/ConFigOss.json`

---

## 7. 本地存储

### `database.sqlite`（drift/sqflite）

| 表 | 说明 |
|---|---|
| `profiles` | 订阅（id、label、url、auto_update、subscription_info 流量配额、selected_map） |
| `rules` | 分流规则 |
| `scripts` | 脚本 |
| `profile_rule_mapping` | 订阅-规则-场景映射 |

### `shared_preferences.json`（应用键）

`flutter.config`(应用设置 JSON)、`flutter.auth_token`(JWT)、`flutter.email`、`flutter.password`(⚠️明文)、`flutter.autoLogin`、`flutter.rememberAccount`、`flutter.hasLoggedInBefore`、`flutter.version`。

### `flutter.config` 关键配置

- `dashboardWidgets`：outboundMode / trafficUsage / systemProxyButton / tunButton
- `networkProps`：systemProxy、bypassDomain 绕过名单、routeMode "config"
- `vpnProps`：TUN 开关、ipv6、访问控制（accessControlProps）
- `patchClashConfig`：下发 mihomo 的配置补丁（mixed-port 7892 等）
- `testUrl`：`http://cp.cloudflare.com/generate_204`

---

## 8. 结论

- **内核**：未魔改的标准 mihomo（MetaCubeX），"换皮"为 `wfnetworkCore.exe`。
- **界面**：Flutter 自研，但大量使用成熟开源 Flutter 包。
- **辅助**：Rust 自研提权服务（借 warp/tokio 等开源库）。
- **商业模式**：订阅制（账号 JWT + 加密订阅下发 + 套餐/优惠券/邀请/ApplePay 订单支付），后端 `weifengapi.com`（阿里云 OSS 分发配置与安装包）。

---

## 9. 隐私 / 安全观察（仅记录，未利用）

1. `shared_preferences.json` 中 **登录密码以明文存储**（`flutter.password`），JWT token 同文件保存。
2. 订阅 profile 与生成的 mihomo 配置均**本地加密/混淆**（`pointycastle` AES），目的之一是阻止用户直接提取节点列表或迁移到其它客户端。
3. 本机同时存在多个代理客户端（微风 7892 / Clash Plus 7877 / Clash Verge / Clash for Windows），系统代理与环境变量端口不一致时会导致代理失效。

---

## 10. VPS 节点分析（实测）

> 实测方式：解析订阅得到节点清单；由于节点域名在公网 DNS 均为 NXDOMAIN，改用「运行中内核的活跃 TCP 连接」还原真实 IP，再对每个 IP 做 ASN 供应商查询 + TCP 连接延迟（RTT）实测。日期 2026-10-30，当时微风内核（PID 11488）正在运行。

### 10.1 节点协议与线路

- 订阅共 **56 个节点**，协议统一为 **VLESS + Reality + xtls-rprx-vision**（Xray 抗封锁配置），`fp=chrome`，SNI 伪装 `python.org` / `www.mozilla.org` / `vuejs.org`。
- 3 个入口域名：
  - `asfhwsrgseg.weifengchat.vip` —— 48 节点，端口 44101–44510（约分 5 组）
  - `dg-drect.iz2ze58f9krop9tgbc.org` —— "东莞直连"，4 节点，端口 11101–11104
  - `jp-drect.iz2ze58f9krop9tgbc.org` —— "日本直连"，4 节点，端口 31101–31104
- 上述域名在公网 DNS **均返回 NXDOMAIN**（阿里/腾讯/Google 公共 DNS 都查不到），属于私用解析（HTTPDNS 或经代理解析），对外隐藏真实 IP。

### 10.2 实际 VPS IP（从内核活跃连接还原）

| IP | 角色 | 供应商 (ASN) | 位置 | 实测 RTT |
|---|---|---|---|---|
| 18.139.83.97 | 主中转入口（对应节点端口 44305/44307） | AWS · AS16509 Amazon | 新加坡 | 108–117 ms |
| 13.250.99.251 | 中转备用（SYN_SENT 未连通） | AWS · AS16509 | 新加坡 | 超时(>5s) |
| 8.152.7.203 | 国内节点（端口 9301/9402/9403） | 阿里云 · AS37963 | 北京 | 27–42 ms |
| 60.204.2.5 | 辅助连接 | 华为云 · AS55990 | 上海 | — |
| 106.38.254.86 | 辅助连接 | 中国电信 IDC · AS23724 | 北京 | — |
| 183.47.108.60 | 辅助连接 | 中国电信骨干 · AS4134 | 广州 | — |
| 1.12.12.12 | 腾讯云 DNS (anycast) | 腾讯 · AS132203 | 广州 | — |
| 223.6.6.6 | 阿里公共 DNS (DoH) | 阿里 · AS45102 | 杭州 | 23 ms |

### 10.3 供应商汇总与架构推断

- **海外中转出口：Amazon AWS（新加坡 ap-southeast-1）** —— 主出口 `18.139.83.97`，本机直连 RTT 约 108–117 ms。
- **国内侧**：阿里云（北京）、华为云（上海）、中国电信（北京/广州）、腾讯云（DNS）—— 用于中转入口 / 直连节点 / DoH 解析。
- **架构推断**：`本机 → AWS 新加坡中转 → 目标网站`，即"新加坡中转"型出口，流量在 AWS 上落地；"东莞直连 / 日本直连"为备选线路。

### 10.4 品牌与后端域名（ConFigOss.json 下发配置，base64 解码）

- 品牌名："微风网络"，上游域名 `breeze-net.net`（`host_source: https://vup.breeze-net.net`）
- API 后端（4 个）：`api01 / api02 / sapi01 / sapi02 .weifengapi.com/api/v1`
- 更新下载：`dl.breezenetaff.com`（windows / macos / android / linux），最新版本 **4.1.0**（本机为 3.0.0）
- 节点域名：`weifengchat.vip`（主）、`iz2ze58f9krop9tgbc.org`（直连）

### 10.5 实测下行带宽（2026-10-30，经代理固定文件法）

> 测速方式：`curl` 走本地混合端口 `127.0.0.1:7892`，下载固定大小文件计时。测速会记入订阅已用流量，本次累计约 126 MB（约占 200 GB 配额的 0.06%）。

| 源 | 文件大小 | 用时 | 平均下行 | 换算 |
|---|---|---|---|---|
| Cloudflare（新加坡同区，`__down?bytes=25000000`） | 25.0 MB | 3.29 s | 7.60 MB/s | ≈ 60.8 Mbps |
| OVH（`proof.ovh.net/files/100Mb.dat`） | 104.9 MB | 11.96 s | 8.77 MB/s | ≈ 70.2 Mbps |

- **真实持续下行 ≈ 70 Mbps（约 8.8 MB/s）**，以 100 MB 长跑为准；Cloudflare 25 MB 同区交叉验证 ≈ 61 Mbps，两者落在 60–70 Mbps 区间。
- **本次出口 IP：`140.235.140.109`**，新加坡，ASN **401243 · Komi LLC**（ipinfo + Cloudflare `cf-meta-ip` 双重确认）。
- 与 10.2 节记录的 AWS 新加坡 `18.139.83.97` 不同 —— 订阅出口在切换，属多出口/动态节点，非固定单 IP。
- 附注：Cloudflare `__down` 对 `bytes ≥ 100000000` 返回 403（仅允许较小分片）；`speed.hetzner.de` TLS 握手失败（exit 35），故本次未采用这两个源。
