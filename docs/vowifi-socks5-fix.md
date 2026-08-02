# VoWiFi SOCKS5 代理 + SIM Auth 修复记录

> 修复日期：2026-08-02
> 测试设备：Qualcomm SDX55 蜂窝模组
> 测试卡：UK MVNO (PLMN 23433 / EE 网络)
> 代理：sing-box SOCKS5 (127.0.0.1:1080)，出口英国

---

## 问题现象

SimAdmin VoWiFi 连接始终卡在 `sim_auth_logical_channel_failed`，无法进入 IKE 阶段。
即使 SIM Auth 偶尔通过，IKE 握手也因为没有走代理而超时（中国直连英国 ePDG UDP 500/4500 不通）。

---

## 根因分析

### 问题一：QMI UIM Open Logical Channel 使用 AID 前缀失败

**现象**：`degraded_reason: sim_auth_logical_channel_failed`

**排查过程**：

1. 通过 `qmicli -p --uim-get-card-status` 确认 SIM 卡状态正常，USIM 应用 ready
2. 手动测试发现：
   - 7 字节 AID 前缀 `A0000000871002` → **失败** (`SimFileNotFound`)
   - 完整 16 字节 AID `A0000000871002FF44FF128900000100` → **成功**
3. 结论：该 Qualcomm 基带固件不支持 QMI UIM Open Logical Channel 的 AID 前缀匹配

**根因**：代码硬编码 `USIM_AID_PREFIX`（7字节）调用 `open_logical_channel`，在不支持前缀匹配的固件上必然失败。

### 问题二：IKE UDP 流量未经过 SOCKS5 代理

**现象**：即使 SIM Auth 通过，IKE_SA_INIT 发出后无响应（4s 超时）

**排查过程**：

1. 对比 另一个已正常工作的 VoWiFi 实现（同设备部署）日志：
   ```
   socks5: 代理传输层初始化成功 proxy=127.0.0.1:1080 relay=<local_relay> remote=<epdg_ip>:500
   ```
2. SimAdmin 源码 `run_live_ike_with_destination` 直接绑定本地 UDP socket 发包，完全不检查代理配置
3. 配置中的 `proxy_mode: socks5_udp_associate` 仅在 ePDG DNS 解析时使用，IKE/ESP 流量直连

**根因**：SOCKS5 代理支持只实现了 DNS 层，IKE 和 ESP 数据面缺失代理转发。

---

## 修复方案

### 修复一：动态获取完整 USIM AID (`qmi_uim.rs`)

```
修改文件：backend/src/connectivity/modems/softstack/vowifi/qmi_uim.rs
新增行数：+284
```

1. 添加 `QMI_UIM_GET_CARD_STATUS` (0x002F) 消息构建和解析
2. 添加 `resolve_full_aid_via_qmicli()` 函数：
   - 调用 `qmicli -d /dev/wwan0qmi0 -p --uim-get-card-status`
   - 解析输出中 "Application ID:" 后面的完整 HEX AID
   - 进程级 `OnceLock` 缓存，避免重复调用
3. 修改 `verify_usim_application_via_proxy_reason()` 和 `execute_usim_authenticate_via_proxy_reason()`：
   - 先调用 `resolve_full_aid_via_qmicli` 获取完整 AID
   - 使用完整 AID 调用 `open_logical_channel`

### 修复二：IKE/EAP-AKA 全链路 SOCKS5 代理 (`live.rs`)

```
修改文件：backend/src/connectivity/modems/softstack/vowifi/live.rs
新增行数：+445
```

1. 重写 `run_live_ike_with_destination()`：
   - 检查 `line_overrides(line_id).proxy` 是否配置了 SOCKS5
   - 有代理时使用 `Socks5UdpClient::connect()` 建立 UDP ASSOCIATE relay
   - IKE_SA_INIT 通过 relay 发送/接收
   - NAT-T 帧封装（前置 4 字节零标记）

2. 新增 `run_live_ike_eap_aka_via_socks5()`：
   - 完整 EAP-AKA 多轮交互（Challenge → SIM Auth → Response → Success）
   - 支持 EAP Identity、Notification 等可选轮次
   - 最终 IKE_AUTH 完成 + Child SA 安装
   - 返回带有 `EspTransport::Socks5` 的 session

3. 新增辅助函数：
   - `socks5_send()` — NAT-T 帧封装 + SOCKS5 发送
   - `socks5_recv_with_retransmit()` — 超时重传 + NAT-T 帧解封

### 修复三：ESP 数据面 SOCKS5 传输 (`tun_gateway.rs`)

```
修改文件：backend/src/connectivity/modems/softstack/vowifi/tun_gateway.rs
新增行数：+71
```

1. 新增 `EspTransport` 枚举：
   ```rust
   pub enum EspTransport {
       Direct(UdpSocketDatagramTransport),
       Socks5 { client: Arc<Socks5UdpClient>, remote: SocketAddr },
   }
   ```
2. 实现 `send_esp_nat_t_metadata()` 和 `recv_nat_t_raw_metadata()` 方法
3. `TunGatewayConfig.transport` 类型改为 `EspTransport`
4. TUN Gateway 透明使用，不关心底层是直连还是代理

---

## 测试验证

### 测试时间线

| 时间 (UTC) | 事件 |
|------------|------|
| 19:14:52 | 首次 IKE_SA_INIT via SOCKS5 成功 |
| 19:14:53 | IKE_SA_INIT response parsed successfully |
| 19:14:54 | USIM Authentication succeeded (auts=false) |
| 19:14:54 | Sending EAP-AKA response |
| 19:14:54 | Received EapSuccess |
| 19:14:54 | IKE session fully established via SOCKS5 proxy! |
| 19:23:52 | (第二轮) IKE session fully established |
| 19:49:06 | IMS REGISTER → 401 (AKAv1-MD5 challenge) |
| 19:49:07 | IMS REGISTER authenticated → 200 OK |
| 19:49:08 | IMS REGISTER ready, TTL=300s |

### 最终状态

```
Phase: voice_ready
  identity_ready: True
  sim_auth_ready: True
  profile_matched: True (gb_ee_23433)
  epdg_ready: True
  ike_ready: True
  child_sa_ready: True
  esp_ready: True
  ims_registered: True
  sms_ready: True
  voice_ready: True
Degraded: None
```

### 关键日志确认

```
IKE session fully established via SOCKS5 proxy!
Successfully established IKE session with destination=<epdg_ip>:4500
IMS REGISTER authenticated response: 200 OK
IMS REGISTER ready cache updated line_id="<line_id>" profile_id="gb_ee_23433" ttl_secs=300
```

---

## 不影响的功能

- 直连 VoWiFi（无代理场景）：代码检测无代理配置时走原有直连路径，行为不变
- VoLTE：完全独立的代码路径，不受影响
- 其他 SIM 管理功能：无改动

## 外部依赖

- **无新增 Cargo 依赖**（Cargo.toml 未修改）
- **唯一的外部命令调用**：`qmicli`（libqmi 套件，设备已预装，SimAdmin 原本就依赖）
