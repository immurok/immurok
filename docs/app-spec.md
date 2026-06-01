# immurok macOS App 技术规格

## 概述

immurok macOS App 是一个常驻后台的 Menu Bar 应用（LSUIElement），通过 BLE 与 CH592F 指纹设备通信，提供屏幕解锁、系统认证、SSH 密钥管理等功能。

## 架构

```
┌──────────────────────────────────────────────────────┐
│                    immurokApp (SwiftUI)               │
│  MenuBarExtra (状态栏) + Settings Window (500x420)     │
├────────────┬──────────────┬───────────┬───────────────┤
│ AppDelegate│ AppViewModel │ContentView│ QuickFillPanel│
│  生命周期   │  UI 状态     │ 设置 Tab  │  全局快捷键    │
├────────────┴──────────────┴───────────┴───────────────┤
│                     BLEManager                        │
│            GATT 通信 / 命令队列 / 重连 / 保活           │
├──────────┬──────────┬──────────┬──────────────────────┤
│ PAMSocket│SSHAgent  │CLISocket │  ImmurokSecurity     │
│ Server   │Server    │Server    │  ECDH/HMAC/Keychain  │
└──────────┴──────────┴──────────┴──────────────────────┘
```

## 源文件结构

| 文件 | 行数 | 功能 |
|------|------|------|
| `immurokApp.swift` | ~190 | 应用入口，MenuBarExtra + Settings Window |
| `AppDelegate.swift` | ~600 | 初始化服务，指纹匹配处理，屏幕解锁 |
| `AppViewModel.swift` | ~420 | UI 状态管理，设备操作封装 |
| `BLEManager.swift` | ~1960 | BLE GATT 核心，命令协议，重连保活 |
| `ImmurokSecurity.swift` | ~230 | ECDH 配对，HMAC 验签，Keychain 存储 |
| `PAMSocketServer.swift` | ~940 | Unix socket PAM 认证服务 |
| `SSHAgentServer.swift` | ~370 | OpenSSH Agent 协议实现 |
| `CLISocketServer.swift` | ~380 | CLI 工具通信接口 |
| `FingerprintView.swift` | ~630 | 指纹管理界面与录入流程 |
| `SettingsTabViews.swift` | ~2190 | 设置窗口各 Tab |
| `QuickFillPanel.swift` | ~820 | 快速填充浮动面板 |
| `KeystoreViewModel.swift` | ~400 | 密钥浏览/导入导出 |
| `SSHKeyCache.swift` | ~270 | SSH 公钥本地缓存 |
| `KeyNameCache.swift` | ~105 | 密钥名称统一缓存 |
| `LocalizationManager.swift` | ~1110 | 8 语言本地化 |
| `SetupManager.swift` | ~110 | 初始状态检查 |
| `SetupWizardView.swift` | ~360 | 首次启动向导 |
| `GlobalHotKey.swift` | — | 全局快捷键注册 |
| `LogManager.swift` | ~25 | 运行日志 |

## BLE 通信

### GATT 服务

| 服务 | UUID | 用途 |
|------|------|------|
| immurok 自定义 | `12340010-...` | 命令/响应通道 |
| CMD 特征 | `12340011-...` | App → 设备（Write with Response） |
| RSP 特征 | `12340012-...` | 设备 → App（Notify） |
| OTA 服务 | `FEE0` / `FEE1` | 固件升级 |
| Device Info | `180A` / `2A26` | 固件版本读取 |

### 命令协议

包格式：`[cmd:1B][payloadLen:1B][payload...]`

| 命令 | ID | 说明 | 需要验证 |
|------|----|------|---------|
| GET_STATUS | 0x01 | 状态/电量/版本/pending match | 否 |
| ENROLL_START | 0x10 | 录入指纹 | 是 |
| ENROLL_CANCEL | 0x11 | 取消录入 | 否 |
| DELETE_FP | 0x12 | 删除指纹 | 是 |
| FP_LIST | 0x13 | 指纹位图 | 否 |
| FP_MATCH_ACK | 0x22 | 匹配确认 | 否 |
| PAIR_INIT | 0x30 | ECDH 配对初始化 | 否 |
| PAIR_CONFIRM | 0x31 | ECDH 配对确认 | 否 |
| PAIR_STATUS | 0x32 | 配对状态查询 | 否 |
| AUTH_REQUEST | 0x33 | PAM 认证请求 | 是 |
| FACTORY_RESET | 0x36 | 恢复出厂 | 是 |
| GATE_CANCEL | 0x37 | 取消指纹门控 | 否 |
| CHALLENGE | 0x38 | 设备身份验证 | 否 |
| KEY_COUNT | 0x60 | 密钥数量 | 否 |
| KEY_READ | 0x61 | 读密钥（公开部分） | 否 |
| KEY_WRITE | 0x62 | 写密钥 | 是 |
| KEY_DELETE | 0x63 | 删除密钥 | 是 |
| KEY_COMMIT | 0x64 | 提交到 Flash | 是 |
| KEY_SIGN | 0x65 | ECDSA 签名 | 是 |
| KEY_GETPUB | 0x66 | 获取公钥 | 否 |
| KEY_GENERATE | 0x67 | 生成密钥对 | 是 |
| KEY_RESULT | 0x68 | 读结果缓冲 | 否 |
| KEY_OTP_GET | 0x69 | TOTP 计算 | 是 |

### 连接管理

- **设备发现**：`retrieveConnectedPeripherals` 查找已配对 HID 设备
- **重连定时器**：断连后每 1 秒 `doConnect()` 重试
- **连接超时**：`.connecting` 状态超过 10 秒自动重置
- **睡眠保活**：`beginActivity(.idleSystemSleepDisabled)` 防止系统深度省电断开 BLE
- **命令队列**：串行发送，`commandInFlight` 互斥，5 秒超时

### 设备状态机

```
disconnected → connecting → connected(name)
     ↑                           │
     └───── didDisconnect ───────┘
```

## 安全机制

### ECDH 配对

```
App                          Device
 │  PAIR_INIT                   │
 │──────────────────────────────►│
 │  [0x30][device_pubkey:33B]   │ ← 设备生成 P-256 密钥对
 │◄──────────────────────────────│
 │  PAIR_CONFIRM(app_pubkey)    │
 │──────────────────────────────►│ ← App 生成密钥对
 │  [0x31][OK]                  │
 │◄──────────────────────────────│
 │                              │
 │  双方: ECDH → HKDF-SHA256   │
 │  salt: "immurok-pairing-salt"│
 │  info: "immurok-shared-key"  │
 │  → shared_key (32 bytes)     │
```

### Challenge-Response 验证

```
连接后:
  UUID 匹配缓存？→ 是 → isDeviceVerified = true（跳过 challenge）
                → 否 → App 发 [0x38][nonce:8B]
                       设备回 [0x38][HMAC-SHA256(key, nonce):8B]
                       App 验证 → 通过则缓存 UUID
```

### 指纹匹配签名

```
设备 → App: [0x21][page_id:2B LE][HMAC-SHA256(key, 0x21||page_id):8B]
App 验证 HMAC 后才触发解锁/认证
```

### Keychain 存储

| 服务标识 | 内容 | 访问级别 |
|---------|------|---------|
| `com.immurok.shared-key` | ECDH 共享密钥 (32B) | AfterFirstUnlock |
| `com.immurok.password` | 屏幕解锁密码 | AfterFirstUnlock |
| `com.immurok.verified-device` | 已验证设备 UUID | AfterFirstUnlock |

### 降级模式

`isDeviceVerified = false` 时禁用：屏幕解锁、PAM 认证、SSH 签名、OTP、指纹录入/删除、密钥写入/删除/生成、恢复出厂。

保留：BLE 连接、只读查询（GET_STATUS/FP_LIST/KEY_READ/KEY_GETPUB）、重新配对。

## 屏幕解锁

### 流程

```
1. 用户按指纹传感器
2. 设备发 HID CTRL 键 → macOS 唤醒屏幕
3. 设备搜索指纹 → 匹配 → 发 0x21 通知（HMAC 签名）
4. App 收到 → 验证 HMAC → handleFingerprintMatch()
5. 检测锁屏窗口（loginwindow/ScreenSaverEngine）
6. fakeKeyStrokes() 输入密码 + 回车
7. 2 秒后检查 isScreensaverWindowVisible()，失败重试一次
```

### Pending Match 机制

睡眠期间 BLE 断连时指纹匹配的 0x21 通知丢失。设备标记 `pending`，重连后 App 发 GET_STATUS 时固件在响应中追加 pending match 数据（30 秒过期）。

### 安全保护

- 输入密码前发 `CGEvent flagsChanged` 清除所有 modifier（防 CTRL 卡住）
- HID_KEY_RELEASE_EVT 失败自动 10ms 重试
- HidDev_Report press 失败不安排 release

## PAM 认证

### Unix Socket 协议

```
Socket: ~/.immurok/pam.sock (0o600)
请求: AUTH:username:service
响应: OK | DENY | RETRY:remaining
```

### 处理流程

```
1. PAM 模块连接 → AUTH:user:service
2. 验证对端 UID（root 或当前用户）
3. 检查服务是否启用（sudo/authorization）
4. 检查预授权（10 秒窗口内自动批准）
5. 检查 isDeviceVerified
6. 发 AUTH_REQUEST → 等待指纹 → 30 秒超时
7. 指纹匹配 → OK | 3 次失败 → DENY
```

### 预授权

指纹匹配后无 PAM 请求时，设置 10 秒预授权窗口。窗口内收到的 PAM 请求自动批准。可按 service 限定范围。

### 支持的 PAM 服务

| 服务 | 设置项 | 用途 |
|------|-------|------|
| sudo / sudo_local | immurok.sudoAuthEnabled | sudo 命令 |
| authorization | immurok.authorizationEnabled | 系统权限弹窗 |

## SSH Agent

### Socket 协议

```
Socket: ~/.immurok/agent.sock (0o600)
协议: OpenSSH Agent Protocol
```

### 支持的消息

| 消息 | ID | 说明 |
|------|----|------|
| REQUEST_IDENTITIES | 11 | 返回所有 SSH 公钥 |
| SIGN_REQUEST | 13 | ECDSA-SHA256-NISTP256 签名（需指纹门控） |
| 其他 | — | 返回 AGENT_FAILURE |

### 密钥缓存

```
~/.immurok/ssh_keys.json
[{index, name, publicKeyBlob, fingerprint}, ...]
```

设备连接时自动同步：KEY_COUNT → KEY_READ 链 → 计算 SHA256 指纹 → 写入 JSON。

## 密钥管理

### 存储类别

| 类别 | ID | 条目大小 | 最大数量 | 内容 |
|------|----|---------|---------|------|
| SSH | 0 | 112B | 32 | name(16) + pubkey(64) + privkey(32) |
| OTP | 1 | 92B | 128 | name(30) + service(30) + secret(32) |
| API | 2 | 160B | 50 | name(32) + key(128) |

### 操作权限

| 操作 | 需指纹门控 | 说明 |
|------|-----------|------|
| KEY_READ | 否（API secret 除外） | 公钥/名称可直接读 |
| KEY_WRITE + KEY_COMMIT | 是 | 写入需指纹确认 |
| KEY_DELETE | 是 | 删除需指纹确认 |
| KEY_SIGN | 是 | ECDSA 签名需指纹 |
| KEY_GENERATE | 是 | 生成密钥对需指纹 |
| KEY_OTP_GET | 是 | TOTP 计算需指纹 |

## Quick Fill

### 激活

- 全局快捷键：`Ctrl+\`（可自定义）
- 通过 `GlobalHotKey` 注册 Carbon Event / NSEvent 监听

### 面板功能

- 搜索框 + 条目列表（最多 6 行）
- 搜索前缀过滤：`o ` OTP、`a ` API、`s ` SSH
- 选中条目 → 需指纹门控（OTP/API）→ 自动粘贴到焦点窗口
- 30 秒指纹验证超时

## CLI 接口

```
Socket: ~/.immurok/cli.sock (0o600)
协议: 行文本，冒号分隔

命令:
  PING              → OK:immurok
  LIST:category     → OK:count\nname1\nname2\n\n
  GET:category:name → OK:value
  ERROR:reason
```

## 设置界面

### Tab 结构

| Tab | 内容 |
|-----|------|
| Device | 连接状态、电量、版本、指纹管理、配对、密码配置 |
| Keys | SSH/OTP/API 密钥浏览、添加、删除、导入导出 |
| Permissions | PAM 功能开关、辅助功能权限、自启动 |
| Status | 系统状态检查清单（连接/指纹/密码/权限/PAM） |
| About | 版本信息、运行日志窗口 |

## 本地化

支持 8 种语言，自动检测系统语言：

zh-Hans（简体中文）、zh-Hant（繁体中文）、en、ja、fr、es、pt、ru

支持外部自定义翻译：`~/.immurok/strings.json`

## 文件系统布局

```
~/.immurok/
  ├── pam.sock          Unix socket (PAM)
  ├── agent.sock        Unix socket (SSH Agent)
  ├── cli.sock          Unix socket (CLI)
  ├── ssh_keys.json     SSH 公钥缓存
  └── strings.json      自定义翻译（可选）

/usr/local/lib/pam/
  └── pam_immurok.so    PAM 模块
```

## 启动序列

```
applicationDidFinishLaunching
  → BLEManager.connect()
  → setupBLECallbacks()
  → PAMSocketServer()
  → SSHAgentServer()    (if enabled)
  → CLISocketServer()   (if enabled)
  → GlobalHotKey()      (Quick Fill)
  → SMAppService.mainApp.register()  (登录项)

BLE 连接成功后:
  → performChallengeVerification()  (UUID 缓存 / challenge)
  → onDeviceConnected
    → refreshDeviceStatus()         (GET_STATUS + pending match)
    → SSHKeyCache.sync()            (SSH 密钥同步)
    → KeyNameCache.syncNonSSH()     (OTP/API 名称同步)
```
