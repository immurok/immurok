# 重连 Challenge-Response 验证 (v2)

## 背景

BLE 重连后需验证设备身份，防止伪装设备绕过安全检查。同时需处理睡眠期间 BLE 断连导致指纹匹配通知丢失的问题。

## 目标

1. 重连后 App 验证设备身份，失败则进入降级模式
2. 验证结果可缓存，避免重复 challenge 往返
3. 睡眠断连后指纹匹配不丢失，用户按一次即可解锁

## 协议设计

### Challenge 命令

| 字段 | 值 |
|------|-----|
| 命令 ID | `IMMUROK_CMD_CHALLENGE` = `0x38` |
| 请求 | `[0x38][nonce:8B]` (9 字节) |
| 成功响应 | `[0x38][hmac:8B]` (9 字节) |
| 失败响应 | `[0x38][0xFF]` (2 字节，设备未配对) |

### HMAC 计算

```
hmac = HMAC-SHA256(shared_key, nonce)[0:8]
```

与 0x21 指纹匹配签名一致：HMAC-SHA256 + 截断前 8 字节。

### 验证缓存

首次 challenge 通过后，App 将设备的 CoreBluetooth UUID 存入 Keychain。后续连接时 UUID 匹配则跳过 challenge，直接标记 verified。

- 存储位置：Keychain (`com.immurok.verified-device`)
- 缓存粒度：per-device UUID（同一台 Mac 上同一设备 UUID 稳定）
- 失效条件：challenge 失败、重新配对、清除配对数据

### 连接时序

```
BLE didConnect
  → discoverServices
    → GATT 就绪
      → UUID 匹配缓存？
          是 → isDeviceVerified = true（跳过 challenge）
          否 → 发送 CMD_CHALLENGE(nonce)
               → 验证 HMAC
                 → 成功: isDeviceVerified = true, 缓存 UUID
                 → 失败: isDeviceVerified = false, 清除缓存
      → onDeviceConnected
        → GET_STATUS（检查 pending FP match）
        → SSH key sync
```

## Pending FP Match 机制

### 问题

睡眠期间 BLE 断连后，用户按指纹唤醒屏幕。HID 键盘（macOS 系统级）秒重连可发送 CTRL 唤醒屏幕，但 App 的 GATT 服务尚未重连，0x21 通知丢失。

### 方案：GET_STATUS 追加 pending match

```
设备匹配指纹 → 发 0x21 → GATT 未连，丢失 → pending=1
→ BLE 重连 → App 发 GET_STATUS
→ 固件在 GET_STATUS 响应中追加 pending match 数据
→ App 验证 HMAC → 触发解锁 → 发 ACK 清除 pending
```

### GET_STATUS 响应格式

正常（9 字节）：
```
[OK][bitmap][paired][battery][ver_major][ver_minor][ver_patch][build_hi][build_lo]
```

有 pending match 时（20 字节）：
```
[OK][bitmap][paired][battery][ver_major][ver_minor][ver_patch][build_hi][build_lo]
[0x21][page_id:2B LE][hmac:8B]
```

App 检查 `response.count >= 20 && response[9] == 0x21` 判断是否有 pending match。

### Pending 状态管理

- 设置：发送 0x21 通知时 `s_fp_notify_pending = 1`
- 清除：收到 ACK (0x22)、GET_STATUS 返回 pending 数据后、30 秒超期
- 跨连接保留：断连时不清除 `s_fp_notify_pending`，重连后仍可追加到 GET_STATUS

## 降级模式

验证失败时禁用敏感操作：

### 禁用

| 操作 | 检查位置 |
|------|---------|
| 屏幕解锁 | `AppDelegate.handleFingerprintMatch` |
| PAM 认证 | `PAMSocketServer` 认证请求处理 |
| SSH 签名 (KEY_SIGN) | App 发命令前 |
| OTP 计算 (KEY_OTP_GET) | App 发命令前 |
| 指纹录入/删除 | App 发命令前 |
| 密钥写入/删除/生成 | App 发命令前 |
| 恢复出厂 | App 发命令前 |

### 保留

- BLE 连接本身
- GET_STATUS / FP_LIST 等只读查询
- 重新配对（配对成功后自动标记 verified）
- 状态栏显示（标记"设备验证失败，敏感操作已禁用"）

## 睡眠连接保活

屏幕休眠时 `beginActivity(.idleSystemSleepDisabled)` 阻止系统进入深度省电，防止 BLE 连接断开。屏幕唤醒时释放。

不保活时的断连-重连循环：
```
屏幕休眠 → ~1min BLE 断连（macOS 深度省电）
→ 设备广播 → macOS 重连 HID 键盘 → 屏幕被唤醒
→ 无操作再次休眠 → 再断连 → 循环
```

## 安全考量

- 8 字节 HMAC 截断提供 64 位安全强度，对 BLE 近距离场景足够
- Challenge nonce 由 App 随机生成，防止重放
- 单向验证（App 验证设备），设备不验证 App
- UUID 缓存不降低安全性：BLE bond 已在链路层认证，每次指纹匹配仍有 HMAC 验签
- Pending match 的 HMAC 防止伪造，30 秒过期防止陈旧匹配

## 改动范围

### 固件

| 文件 | 改动 |
|------|------|
| `immurokservice.h` | 新增 `IMMUROK_CMD_CHALLENGE 0x38` |
| `immurok_security.h/c` | 新增 `immurok_security_challenge_response()` |
| `hidkbd.c` | CMD_CHALLENGE 处理；GET_STATUS 追加 pending match；连接时更新 immurokConnHandle；FP match 后标记 pending |

### App

| 文件 | 改动 |
|------|------|
| `ImmurokSecurity.swift` | challenge 生成/验证；verified device UUID 缓存（Keychain） |
| `BLEManager.swift` | challenge 验证流程（含 UUID 缓存快速路径）；`isDeviceVerified` 状态；GET_STATUS 解析 pending match；睡眠保活 |
| `AppDelegate.swift` | `handleFingerprintMatch` 检查 `isDeviceVerified`；`fakeKeyStrokes` 清除 modifier |
| `PAMSocketServer.swift` | 认证请求检查 `isDeviceVerified` |
| `AppViewModel.swift` | 暴露 `isDeviceVerified` 给 UI |
| `SettingsTabViews.swift` | 未验证时显示红色提示 |
| `LocalizationManager.swift` | 新增 `device.not.verified` 本地化 |
