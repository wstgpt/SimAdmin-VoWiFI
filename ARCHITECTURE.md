# SimAdmin-Enhance 项目架构分析

> 分支：AI-Generate | 版本：v1.1.3 | 分析时间：2026-08-02

---

## 1. 项目概述

SimAdmin-Enhance 是一套面向 Debian 蜂窝 CPE / 随身 WiFi / 软路由设备的 **SIM/eSIM 中枢管理系统**。核心亮点为原生实现 WiFi Calling (VoWiFi)，具备完整的 IKEv2/IPsec 全链路能力，无第三方程序依赖。

**技术栈一览：**
| 层级 | 技术选型 |
|------|----------|
| 后端 | Rust + Axum + Tokio + zbus (D-Bus) |
| 前端 | React 19 + TypeScript + Vite 7 + Material UI 7 |
| 数据库 | SQLite (rusqlite, bundled) |
| 部署 | 单二进制同进程托管 SPA + systemd |
| 构建 | pnpm (前端) + cargo (后端) + GitHub Actions CI/CD |

---

## 2. 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        浏览器 (React SPA)                        │
│  Dashboard │ SIM │ eSIM │ Network │ SMS │ Phone │ OTA │ ...     │
└────────────────────────────────┬────────────────────────────────┘
                                 │ HTTP/REST API
┌────────────────────────────────┼────────────────────────────────┐
│                        Axum HTTP Server                          │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ API 层   │  │ Auth 中间件│  │ SPA 静态服务│  │ CORS / Limits│  │
│  └─────┬────┘  └──────────┘  └────────────┘  └──────────────┘  │
│        │                                                        │
│  ┌─────▼─────────────────────────────────────────────────────┐  │
│  │                  Services (业务逻辑层)                      │  │
│  │  Orchestrator │ Trunk │ Messaging │ Automation │ Notify    │  │
│  │  System │ Network │ LineRegistry                           │  │
│  └─────┬─────────────────────────────────────────────────────┘  │
│        │                                                        │
│  ┌─────▼─────────────────────────────────────────────────────┐  │
│  │              Connectivity (IMS 连接层)                      │  │
│  │  ┌────────────────────┐   ┌──────────────────────────┐    │  │
│  │  │   Core (共享核心)    │   │  Modems / Softstack      │    │  │
│  │  │ SIP Frame │ AKA    │   │  ┌─────────┐ ┌────────┐  │    │  │
│  │  │ SMS Codec │ Voice  │   │  │ VoLTE   │ │ VoWiFi │  │    │  │
│  │  │ Register  │        │   │  └─────────┘ └────────┘  │    │  │
│  │  └────────────────────┘   └──────────────────────────┘    │  │
│  └─────┬─────────────────────────────────────────────────────┘  │
│        │                                                        │
│  ┌─────▼─────────────────────────────────────────────────────┐  │
│  │                Hardware (硬件抽象层)                        │  │
│  │  Cellular (ModemManager/QMI/AT) │ SIM (eSIM/lpac)         │  │
│  └─────┬─────────────────────────────────────────────────────┘  │
│        │                                                        │
│  ┌─────▼─────────────────────────────────────────────────────┐  │
│  │                Platform (平台基础设施)                      │  │
│  │  ConfigManager │ Database (SQLite) │ Utils                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
   ┌──────────────┐
   │ Linux Kernel │  D-Bus / ModemManager / NetworkManager
   │ + 蜂窝基带   │  QMI / AT 串口 / ip xfrm / iptables
   └──────────────┘
```

---

## 3. 后端架构详解

### 3.1 分层结构

后端代码位于 `backend/src/`，按功能领域分为 5 个顶级模块：

```
backend/src/
├── main.rs            # 程序入口：加载配置、初始化子系统、装配路由
├── state.rs           # 全局 AppState（共享状态）
├── api/               # Web 层：HTTP 路由、请求/响应模型、认证
├── services/          # 业务逻辑层
├── connectivity/      # IMS 连接层（协议核心 + 接入实现）
├── hardware/          # 硬件抽象层
└── platform/          # 基础设施（配置、数据库、工具）
```

### 3.2 API 层 (`api/`)

| 文件 | 职责 |
|------|------|
| `handlers.rs` (296KB) | 所有 `/api/*` 路由处理函数 |
| `models.rs` | 请求/响应 DTO |
| `auth.rs` | 密码/会话认证 + auth 中间件 |

- 使用 Axum 框架，路由注册在 `main.rs`
- 支持密码保护、会话管理、空闲超时自动登出

### 3.3 Services 层 (`services/`)

应用级业务逻辑，位于 Connectivity 和 Hardware 之上：

| 子模块 | 职责 |
|--------|------|
| `orchestrator/` | 多路径 SMS/语音路由决策、MT 监听选举、跨传输去重 |
| `trunk/` | SIP Trunk 网关，将已建立的连接腿暴露为 SIP 中继线 |
| `messaging/` | SMS 接收/转发 + 验证码提取 |
| `automation/` | 定时/触发任务调度器（重启、发短信、拨号、数据消耗等） |
| `notify/` | 多通道通知推送 + 发送队列 |
| `system/` | OTA 更新、系统事件、设备状态监控 |
| `network/` | DDNS 动态域名管理 + iptables 防火墙 |
| `line_registry/` | 每 modem/SIM 运行时注册表，绑定物理线路到各 Runtime |

### 3.4 Connectivity 层 (`connectivity/`)

核心设计理念：**共享核心 + 可插拔接入腿**。

#### Core（共享核心）
所有接入方式公用的协议实现，只编写和测试一次：
- `sip_frame.rs` — SIP 报文组帧/解析 (RFC 3261)
- `sip_message.rs` — SIP 消息结构
- `digest_aka.rs` — Digest-AKA 鉴权 (RFC 3310/4169)
- `sms_codec.rs` — SMS PDU 编解码 (67KB，含完整 3GPP 实现)
- `voice.rs` — 语音通话核心逻辑
- `register.rs` — IMS 注册流程
- `access.rs` — 接入腿抽象接口
- `context.rs` — 连接上下文

#### Modems / Softstack（接入腿实现）

**VoWiFi 接入腿** (`vowifi/`)：
WiFi → ePDG → IMS，用户态全实现，~50 个源文件：
- IKEv2 协议栈：`ike.rs`, `ike_state.rs` (124KB), `ike_codec.rs`, `ike_keys.rs`, `ike_dh.rs`, `ike_eap.rs`, `ike_encrypted.rs`, `ike_payloads.rs`, `ike_retransmit.rs`, `ike_events.rs`, `ike_identity.rs`
- EAP-AKA 鉴权：`eap_aka.rs`, `aka.rs`, `qmi_uim.rs`
- 数据面：`dataplane.rs` (60KB), `tun_gateway.rs`, `transport.rs`, `socks5.rs`
- ePDG 发现：`epdg.rs`
- IMS 信令：`ims.rs`
- 运行时管理：`runtime.rs`, `live.rs` (274KB), `executor.rs`, `flow.rs`
- 配置管理：`profiles.rs`, `profile_store.rs`, `profile_record.rs`, `profile_import.rs`
- 诊断/稳定性：`diagnostics.rs`, `stability.rs`, `soak.rs`, `restore.rs`

**VoLTE 接入腿** (`volte/`)：
LTE 基带 → IMS APN bearer → IMS，内核 IPsec：
- `bearer.rs` — IMS 承载建立
- `pcscf.rs` — P-CSCF 发现与连接
- `ipsec.rs` — 内核 `ip xfrm` IPsec 配置
- `sip.rs` — SIP 信令处理
- `digest_aka.rs` — VoLTE 专用 AKA
- `sms.rs` — VoLTE SMS
- `voice.rs` — VoLTE 语音
- `rtp_relay.rs` — RTP 中继
- `runtime.rs`, `live.rs` (146KB) — 运行时生命周期
- `native_bearer.rs` — 原生承载管理
- `vilte.rs` — ViLTE (视频通话) 支持
- `data_slot.rs`, `identity.rs`, `channel.rs`, `plan.rs`, `readiness.rs`, `errors.rs`

### 3.5 Hardware 层 (`hardware/`)

| 子模块 | 职责 |
|--------|------|
| `cellular/` | 蜂窝调制解调器控制：ModemManager D-Bus、QMI、AT 指令、小区锁定、串口通信 |
| `sim/` | eSIM/eUICC 配置管理（通过 `lpac` 工具） |

### 3.6 Platform 层 (`platform/`)

| 文件 | 职责 |
|------|------|
| `config.rs` (171KB) | 配置管理器 — 所有运行参数的读写 |
| `db.rs` (135KB) | SQLite 数据库层 — 短信、通话、事件、通知等持久化 |
| `utils.rs` (44KB) | 通用工具函数 |

### 3.7 全局状态 (`state.rs`)

`AppState` 结构体持有所有共享资源，通过 Axum `FromRef` 注入到 handler：
- D-Bus 连接 (`zbus::Connection`)
- 数据库实例 (`Database`)
- 配置管理器 (`ConfigManager`)
- 通知发送器、系统事件发射器
- DDNS 管理器、eSIM 管理器
- VoWiFi/VoLTE Runtime
- SMS 重同步句柄
- 小区锁定状态、活跃通话记录
- Line Runtime 注册表

---

## 4. 前端架构详解

### 4.1 技术栈

- **框架**：React 19 + TypeScript 5.9
- **构建工具**：Vite 7 + pnpm
- **UI 库**：Material UI 7 (MUI) + MUI X Charts/DataGrid
- **路由**：React Router DOM 7
- **状态/数据获取**：TanStack React Query 5 + SWR
- **代码分割**：React.lazy + Suspense (路由级)

### 4.2 目录结构

```
frontend/src/
├── main.tsx              # 应用入口
├── App.tsx               # 路由配置 + 认证保护壳
├── index.css             # 全局样式
├── api/                  # API 层
│   ├── contracts.ts      # API 类型契约 (60KB)
│   ├── current.ts        # API 调用函数 (38KB)
│   └── types.ts          # 通用类型
├── contexts/             # React Context
│   ├── ThemeContext.tsx   # 深色/浅色主题切换
│   └── RefreshContext.tsx # 刷新触发
├── components/           # 共享组件
│   ├── Layout/           # 布局框架（侧边栏、顶栏、主布局）
│   ├── ErrorSnackbar.tsx
│   ├── DateRangePicker.tsx
│   ├── ModemLineSelector.tsx
│   └── PasswordStrengthHint.tsx
├── pages/                # 页面组件（路由级代码分割）
│   ├── Dashboard/        # 仪表盘（组件化拆分 + hooks）
│   ├── SimCard.tsx       # SIM 卡管理
│   ├── EsimManager.tsx   # eSIM 管理 (85KB)
│   ├── DeviceNetwork.tsx # 设备网络 (64KB)
│   ├── SMS.tsx           # 短信管理 (49KB)
│   ├── Phone.tsx         # 电话
│   ├── Configuration.tsx # 系统配置 (32KB)
│   ├── OtaUpdate.tsx     # OTA 更新 (39KB)
│   ├── Login.tsx         # 登录
│   ├── NotificationCenter.tsx # 通知中心
│   ├── AutomationCenter.tsx   # 自动化中心
│   ├── VowifiDiagnostics.tsx  # VoWiFi 诊断 (84KB)
│   ├── sim/              # SIM 子组件
│   ├── phone/            # 电话子组件
│   ├── sms/              # 短信子组件
│   ├── notifications/    # 通知子组件
│   ├── automation/       # 自动化子组件
│   └── device-network/   # 网络子组件
├── lib/                  # 工具库
│   ├── queryClient.ts    # React Query 客户端
│   └── passwordPolicy.ts # 密码策略
└── utils/                # 工具函数
    ├── carriers.ts       # 运营商信息
    ├── theme.ts          # 主题工具
    ├── ip.ts             # IP 工具
    └── modemErrors.ts    # Modem 错误码
```

### 4.3 路由设计

```
/login              → Login（公开）
/                   → Dashboard（受保护）
/sim                → SimCard / eSIM（tab 切换）
/device-network     → DeviceNetwork
/sms                → SMS
/phone              → Phone
/notifications      → NotificationCenter
/automation         → AutomationCenter
/config             → Configuration
/config/security    → Configuration（安全设置）
/ota                → OtaUpdate
```

所有受保护路由通过 `ProtectedShell` 组件包裹，未认证时重定向到 `/login`，支持空闲超时自动登出。

### 4.4 Dashboard 组件化设计

Dashboard 页面采用组件化 + hooks 拆分模式：
- `hooks/useDashboardData.ts` — 数据获取逻辑
- `components/` — 独立展示组件：
  - `StatusOverview` — 状态概览
  - `DeviceInfoCard` — 设备信息
  - `SimCardInfo` — SIM 卡信息
  - `ConnectionStatus` — 连接状态
  - `NetworkSpeed` — 网速监控
  - `SystemResources` — 系统资源
  - `TemperatureMonitor` — 温度监控
  - `CellInfo` — 小区信息
  - `QuickControls` — 快捷操作

---

## 5. 数据流

```
用户操作 → React 组件 → API 调用 (api/current.ts)
                              │
                              ▼ HTTP
                     Axum Router (handlers.rs)
                              │
                              ▼
                     AppState (state.rs)
                       ┌──────┼──────┐
                       ▼      ▼      ▼
              ConfigManager  DB   Runtime
                       │      │      │
                       ▼      ▼      ▼
              文件系统   SQLite  D-Bus/QMI/AT → 基带硬件
```

**异步事件流（SMS 接收为例）：**
```
ModemManager D-Bus 信号 → SmsListener → Orchestrator(去重/选路)
    → Database 持久化
    → NotificationSender → 多通道推送 (Webhook/邮件/Telegram/...)
```

---

## 6. 部署架构

```
┌─────────────────────────────────┐
│     /opt/simadmin/              │
│  ├── simadmin        (后端二进制) │
│  ├── www/            (前端 SPA)  │
│  ├── config.json     (运行配置)  │
│  └── data.db         (SQLite)   │
└──────────────┬──────────────────┘
               │ systemd (simadmin.service)
               ▼
         0.0.0.0:8080 (默认)
```

- 单进程同时服务 API 和前端静态文件
- 支持 IPv4/IPv6 双栈
- 通过 GitHub Actions 自动构建 ARM64/AMD64 Release
- 支持 OTA 空中升级

---

## 7. CI/CD 流程

GitHub Actions 定义了 3 个工作流：

| 工作流 | 用途 |
|--------|------|
| `build-release.yml` | 构建正式发布包（跨平台交叉编译） |
| `build-ota-package.yml` | 构建 OTA 升级包 |
| `build-lpac-compat.yml` | 构建 lpac 兼容版（eSIM 工具） |

---

## 8. 核心设计亮点

### 8.1 共享核心 + 可插拔接入腿

IMS 协议栈设计为两层：
- **Core**：SIP 帧、Digest-AKA、SMS 编解码 — 所有接入方式共用
- **Access Leg**：VoWiFi (用户态 IKEv2) / VoLTE (内核 IPsec) — 独立实现

新增接入方式（如 ViLTE、CS）仅需在 `modems/` 下新增目录，复用 Core。

### 8.2 Line Registry 多线路管理

`LineRuntimeRegistry` 为每个物理 modem + SIM 绑定独立的 VoLTE/VoWiFi/Trunk Runtime，支持多卡多待场景。

### 8.3 Orchestrator 智能路由

`orchestrator/` 模块实现：
- `sms_router` — 发送路由选择（VoLTE / VoWiFi / 基带 3 条路径择优）
- `voice_router` — 语音路由
- `listener_election` — 多腿 MT 监听选举（避免重复接收）
- `dedup` — 跨传输去重

### 8.4 完整的 VoWiFi 用户态实现

从 IKEv2 SA 协商、EAP-AKA 鉴权、ESP 数据面、TUN 网关到 SIP 注册，全部用户态 Rust 实现，无需 strongSwan 等第三方依赖。

### 8.5 前端路由级代码分割

所有页面使用 `React.lazy` + `Suspense` 实现按需加载，减小首屏体积。

---

## 9. 代码规模统计

| 模块 | 估算代码量 |
|------|-----------|
| 后端 Rust | ~2.5 MB 源码 |
| 前端 TypeScript/TSX | ~700 KB 源码 |
| 配置/脚本 | ~80 KB |
| API 文档 (Bruno) | ~30 KB |

主要大文件：
- `vowifi/live.rs` — 274 KB（VoWiFi 完整生命周期）
- `api/handlers.rs` — 296 KB（所有 API handler）
- `platform/config.rs` — 171 KB（配置管理）
- `platform/db.rs` — 135 KB（数据库层）
- `vowifi/ike_state.rs` — 124 KB（IKEv2 状态机）
- `volte/live.rs` — 146 KB（VoLTE 生命周期）

---

## 10. 依赖关系方向

```
api → services → connectivity → hardware
                              → platform (config/db/utils)

connectivity/modems → connectivity/core (单向，core 不依赖 modems)
```

跨领域引用使用绝对路径 `crate::模块::子模块`，领域内部可用 `super::` 相对引用。

---

## 11. 安全设计

- 密码保护 + 会话认证（`api/auth.rs`）
- 空闲超时自动登出
- 密码强度策略验证
- 前端表单输入验证
- D-Bus 权限控制
- 单机部署，无外部网络暴露

---

## 12. 扩展性

- **新接入方式**：在 `connectivity/modems/` 下新增目录，实现 `core/access.rs` 定义的接口
- **新自动化任务**：在 `services/automation/tasks/` 下添加任务实现
- **新通知通道**：在 `services/notify/` 中扩展
- **新 API 端点**：在 `api/handlers.rs` 添加 handler + `main.rs` 注册路由
- **新前端页面**：在 `pages/` 下添加组件 + `App.tsx` 注册路由
