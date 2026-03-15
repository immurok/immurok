# 重连 Challenge-Response 验证

## 背景

当前 BLE 重连后，App 仅通过 `GET_STATUS` 读取设备的配对布尔值，没有验证设备是否真正持有 shared_key。伪装设备只需在 `GET_STATUS` 中声称 `paired=1` 即可通过检查。

## 目标

重连后 App 通过 challenge-response 验证设备身份。验证失败则进入降级模式，禁用敏感操作。

## 协议设计

### 新命令

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

与现有 0x21 指纹匹配签名一致：HMAC-SHA256 + 截断前 8 字节。

### 时序

```
BLE didConnect
  → discoverServices
    → GATT 就绪 (onDeviceReady)
      → App 发送 CMD_CHALLENGE(random_nonce_8B)
        → 设备回复 HMAC(shared_key, nonce)[0:8]
          → App 验证:
              匹配 → isDeviceVerified = true → 继续 GET_STATUS 正常流程
              不匹配 → isDeviceVerified = false → 降级模式
      → (若 App 本地无 shared_key → 跳过 challenge，isDeviceVerified = false)
```

Challenge 在 GET_STATUS 之前执行，确保后续所有操作都基于已验证的身份。

## 降级模式

### 禁用的操作

- 屏幕解锁（不发送密码）
- PAM 认证（拒绝 socket 请求）
- SSH 签名 (`KEY_SIGN`)
- OTP 计算 (`KEY_OTP_GET`)
- 指纹录入/删除

### 保留的操作

- BLE 连接本身
- 状态栏显示（标记"未验证"）
- 重新配对（配对成功后自动做一次 challenge 验证）
- GET_STATUS / FP_LIST 等只读查询

### 状态栏提示

状态栏图标旁显示警告标记，菜单中显示"设备未验证"提示。

## 改动范围

### 固件

| 文件 | 改动 |
|------|------|
| `immurokservice.h` | 新增 `IMMUROK_CMD_CHALLENGE 0x38` |
| `immurok_security.h` | 声明 `immurok_security_challenge_response()` |
| `immurok_security.c` | 实现：接收 8B nonce，HMAC-SHA256(shared_key, nonce)，返回前 8B |
| `hidkbd.c` | 新增 `IMMUROK_CMD_CHALLENGE` case，无需指纹门控，直接计算返回 |

### App

| 文件 | 改动 |
|------|------|
| `ImmurokSecurity.swift` | 新增 `generateChallenge() -> Data` 和 `verifyChallengeResponse(nonce:response:) -> Bool` |
| `BLEManager.swift` | GATT 就绪后发 challenge，新增 `isDeviceVerified` 属性，验证通过后触发 `onDeviceConnected` |
| `AppDelegate.swift` | 解锁/PAM 流程检查 `isDeviceVerified` |
| `PAMSocketServer.swift` | 认证请求检查 `isDeviceVerified` |
| `AppViewModel.swift` | 暴露 `isDeviceVerified` 给 UI |
| `ContentView.swift` | 未验证时状态栏显示警告 |

## 安全考量

- 8 字节 HMAC 截断提供 64 位安全强度，对 BLE 近距离场景足够
- nonce 由 App 随机生成，防止重放攻击
- 单向验证（App 验证设备），设备不验证 App
- 配对成功后自动触发 challenge，无需额外操作

## 不做的事

- 双向验证（当前威胁模型不需要）
- 加密 GATT 通道（依赖 BLE 层加密）
- nonce 持久化（每次重连生成新的）
