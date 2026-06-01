# 固件量产发布测试流程

适用于 FW 1.3.1+ / VER=5 / 单一 SKU（外壳 W/B/G 共用同一份固件，1.2.x 的三色编译期分支已废弃）。

---

## Phase 0 — 代码冻结准备

| # | 检查项 | 通过判据 |
|---|--------|---------|
| 0.1 | 确认 `version.h` patch 号已 bump | `git diff HEAD~1 firmware/APP/include/version.h` |
| 0.2 | 工作目录干净 | `git status` 无未提交、tag 已打 |
| 0.3 | OTA 密钥就位 | `firmware/APP/include/ota_keys.h` 存在且 **不在 git** |
| 0.4 | SDK 到位 | `firmware/SDK/EVT/...` 完整 |
| 0.5 | release 编译通过且零 warning | 见下 Phase 1 |
| 0.6 | App 端协议版本与固件对齐 | `protocol.md` ↔ `BLEManager.swift` 枚举值一致 |

---

## Phase 1 — 构建产物验证

单一 build：

```bash
ota/build-ota.sh release    # 编译 + 打包，不烧
# 或
make clean && make RELEASE=1 VER=5
```

| # | 检查项 | 通过判据 |
|---|--------|---------|
| 1.1 | 编译零警告（除已知 SDK 警告） | `make 2>&1 \| grep -E "warning\|error"` |
| 1.2 | `.text` ≤ 216KB | `riscv-wch-elf-size build/*.elf` 第 1 列 + 第 2 列 |
| 1.3 | `.bss + .data + .highcode + stack_guard + stack` ≤ 26KB | size 输出加和 |
| 1.4 | 单一 `.imfw` 生成且 SHA256 记录 | `shasum -a 256 firmware/dist/fw-1.3.1.*-r.imfw` |
| 1.5 | OTA combined hex 烧录布局正确 | 0x0000 JumpIAP / 0x1000 App / 0x6D000 IAP |
| 1.6 | DIS Model Number = `IK-1`（无 variant 后缀） | nRF Connect 读 0x2A24 |
| 1.7 | 公开私钥未泄露到固件 | `strings build/*.bin \| grep -i "PRIVATE\|BEGIN"` 应为空 |

---

## Phase 2 — 单板冒烟（Bench Smoke，≥ 3 台不同外壳）

烧录新固件后，**不连 App** 先验硬件层：

| # | 项目 | 步骤 | 通过判据 |
|---|------|------|---------|
| 2.1 | 上电启动 | 接电池/USB | RED 闪一次（启动指示） |
| 2.2 | 按键消抖 | 按 BTN 5 次 | 串口看到 5 次 BTN 事件，无抖动重触发 |
| 2.3 | 触摸检测 | 手指放/离指纹片 10 次 | 10 次 TOUCH_INT，无误触 |
| 2.4 | 指纹模组初始化 | 启动日志 | `Fingerprint module OK`，bitmap 读到 |
| 2.5 | 电池电压采样 | 读 GET_STATUS / 串口 | VBAT 在 3.0–4.2V 合理范围，相邻读数 ±50mV |
| 2.6 | RGB LED 三路单独点亮 | 厂测命令逐路点亮 R/G/B | 颜色对、不串色 |
| 2.7 | BLE 广播可见 | nRF Connect 扫 | 名称 `immurok IK-1`、RSSI 合理 |

---

## Phase 3 — 配对与认证主路径（含 App）

≥ 3 台。**断开旧配对、清 keystore 后开始**。

| # | 项目 | 通过判据 |
|---|------|---------|
| 3.1 | 首次配对（pairing handshake） | App "已配对"，HKDF 派生成功 |
| 3.2 | 录入指纹（12 capture 全程） | step 1 polling 期间 BLE 不掉，最终 `FP_ENROLL_DONE` |
| 3.3 | 删除指纹 | App 列表实时更新 |
| 3.4 | sudo 主动认证 | `imk run --agent -- sudo ls` 弹密码框 → 触摸即过 |
| 3.5 | sudo 被动认证（设备驱动） | 终端裸跑 `sudo ls` → 触摸即过 |
| 3.6 | 锁屏自动解锁 | Ctrl+Cmd+Q → 触摸 → 进入桌面 |
| 3.7 | 预发回车（无待处理 PAM） | 锁屏后先触摸 → 出对话框，再触摸 → 解锁 |
| 3.8 | KEY_SIGN（SSH 签名） | `imk run --agent -- git push` 正常推送 |
| 3.9 | KEY_OTP_GET | `imk get imk://otp/...` 返回 6 位数 |
| 3.10 | KEY_GETPUB | `imk list` 显示公钥 |
| 3.11 | LED 状态指示 | KEY_* 期间黄灯，认证成功绿闪 |

---

## Phase 4 — BLE 稳定性 / 长跑

| # | 项目 | 时长 | 通过判据 |
|---|------|------|---------|
| 4.1 | 长连接静默 | ≥ 8 小时 | 不掉线（1.2.31 keep-alive 已修） |
| 4.2 | 远离 / 走回 | 走出 5m → 回来 | 自动重连 ≤ 30s |
| 4.3 | Mac 睡眠唤醒 | sleep → wake ×10 | 每次都能重连，无需手动 |
| 4.4 | 反复触摸 | 连续 100 次触摸认证 | 无丢失、无 BLE 断 |
| 4.5 | 批量 KEY_SIGN | 连续 50 次 ssh-sign | 全部成功，无 timeout |
| 4.6 | GET_STATUS 高频 | App 每 1s 刷一次 ×30min | 无阻塞、无掉线（1.2.30 修过） |
| 4.7 | 配对加密会话 | 配对后 24h | 加密链路保持 |

---

## Phase 5 — OTA 升级

| # | 项目 | 通过判据 |
|---|------|---------|
| 5.1 | OTA 升级（任意外壳） | App 推送 → 进度 100% → 重启 → 新版本号 |
| 5.2 | _(已废弃)_ 1.3.1 起单一 SKU，无跨 variant | N/A |
| 5.3 | OTA 中途断电 | 重新上电仍能正常工作（旧 image 保留） |
| 5.4 | OTA 中途断 BLE | 重连后可重传或回退 |
| 5.5 | HMAC 篡改拒绝 | 改 1 字节 imfw → 设备返回 `OTA_ERR_HMAC_MISMATCH` |
| 5.6 | SHA256 篡改拒绝 | 改 payload → `OTA_ERR_SHA256_MISMATCH` |
| 5.7 | 升级后首启动 | LED 启动闪 + Fingerprint init OK + 配对仍在 |
| 5.8 | 跨 patch 升级（1.2.29→1.2.31） | 不丢配对、不丢 keystore |

---

## Phase 6 — 极端 / 边界

| # | 项目 | 通过判据 |
|---|------|---------|
| 6.1 | 低电量（VBAT < 3.3V） | 警告上报 App，仍能完成认证 |
| 6.2 | 极低电量（< 3.0V） | 安全关机，不写坏 EEPROM |
| 6.3 | 充电中认证 | 不影响指纹/BLE |
| 6.4 | 指纹模组拔插（开发板）| 上报 FP_ERR，不死机 |
| 6.5 | EEPROM 写满（keystore 满） | 返回 NO_SPACE，旧条目不损坏 |
| 6.6 | 恢复出厂 | 配对清、keystore 清、bitmap 清；可立即重新配对 |
| 6.7 | uECC 长签名期间触摸 | 不破坏 BLE 状态、不重启（stack guard 验证） |
| 6.8 | 运行中拍打 / 静电 ESD | 不重启 |

---

## Phase 7 — 量产烧录线

| # | 项目 | 通过判据 |
|---|------|---------|
| 7.1 | Burner CLI 流程 | `burner ping` → `erase` → `flash combined.hex` → `fuse fuse.json` 一次性走通 |
| 7.2 | Fuse 配置正确 | RESET_EN / DEBUG_EN / 读保护按出厂方案 |
| 7.3 | 烧录后回读校验 | CRC/SHA 与本地 hex 一致 |
| 7.4 | 烧录耗时 | ≤ 单台 30s（决定产线节拍） |
| 7.5 | 出厂自检脚本 | 厂测固件 → BLE 广播 + LED 自检 + 指纹 self-test 一次性通过 |
| 7.6 | MAC / 序列号管理 | 每台唯一、可追溯（烧录日志归档） |
| 7.7 | _(已废弃)_ 1.3.1 起单一 SKU，外壳颜色不影响固件 | — |

---

## Phase 8 — 包装前抽检（每批 3–5%）

| # | 项目 |
|---|------|
| 8.1 | 复跑 Phase 2 + Phase 3.1–3.6 |
| 8.2 | 出厂二维码 / SN / 颜色与盒子一致 |
| 8.3 | 静置 24h 自放电 < 5%（电池） |

---

## Phase 9 — 发布前最后确认

| # | 检查项 |
|---|--------|
| 9.1 | git tag `production-N`（参考 `production-1` = 1.2.27 模式） |
| 9.2 | `firmware/dist/` 单一 imfw 归档到 release 目录 |
| 9.3 | OTA 服务器（如有）灰度配置 |
| 9.4 | App Store 版本与固件兼容矩阵更新 |
| 9.5 | 回滚预案：上一个 stable 版本 imfw 备好 |
| 9.6 | 量产 1 = 1.2.27 经验已并入 memory，本次新坑 → 写 `project_production_batches.md` |

---

## 测试矩阵建议（最小集）

| 外壳 | 设备数 | 主测项 |
|------|-------|--------|
| W | 3 台 | 全 Phase 2–6 |
| B | 2 台 | Phase 2 + 3 + 5（OTA） |
| G | 2 台 | Phase 2 + 3 + 5 |

> 1.3.1 起固件单一 SKU，外壳颜色不影响固件行为，不再需要"跨色 OTA"用例。

---

## 通过条件

- Phase 0–7 **全部 PASS**（无 BLOCKER / CRITICAL）
- Phase 4.1（8h 长跑）至少 1 台跑满
- Phase 5（OTA）至少 2 台不同外壳跑过
- Phase 6.1b（低电量写拒绝）必须 PASS（数据完整性）
- Phase 6 其它极端项允许 NOTE 级别问题，但需在 release notes 明确
