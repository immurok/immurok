# 量产 QA 人工测试 Checklist

> 本文档只列**自动化跑不了、需要人/硬件/App 在环**的项。
> 自动化部分见 [qa-production-report.md](qa-production-report.md)。
> 完整流程定义见 [qa-production-checklist.md](qa-production-checklist.md)。

- 固件：1.3.1+ / VER=5 / **单一 SKU**（外壳 W/B/G 三色共用同一份固件）
- App：1.10+ / 1.11+
- 测试人：__________ 测试日期：__________
- 测试设备 SN：______ ______ ______

---

## 0. 测试前准备

- [ ] dist/ imfw 已就位（SHA256 见 report）
- [ ] burner 工具可用（`burner ping` 通）
- [ ] Mac 配 immurok.app 最新版（与固件兼容）
- [ ] nRF Connect / LightBlue 装好（用于 BLE 调试）
- [ ] 测试用 Mac 已开"辅助功能"权限给 immurok.app
- [ ] 至少 2 台已配对设备 + 2 台未配对设备
- [ ] 充电电源 + 万用表 + 电池电量计

---

## Phase 2 — 单板冒烟（≥ 3 台不同外壳）

**前置：** 烧 release combined hex，**不连 App**，串口 115200 接 PA4/PA5（仅 release-debug 才有日志，release 没有 — 走外观+nRF 验证）。

### W 设备
- [ ] **2.1 上电启动**：RED 灯闪一次（启动指示） — PASS / FAIL
- [ ] **2.2 按键消抖**：按 BTN 5 次，间隔 1s — 无重触发（看 nRF Connect 是否有异常事件） — PASS / FAIL
- [ ] **2.3 触摸检测**：手指放/离 10 次 — 每次 BLE 收到 fp_touch 事件 — PASS / FAIL
- [ ] **2.4 指纹模组初始化**：上电后用 nRF Connect 读 GET_STATUS — 返回 `bitmap` 字段不全为 0xFF — PASS / FAIL
- [ ] **2.5 电池采样**：GET_STATUS 读 batt 字段，万用表对比 — 误差 ≤ ±100mV — PASS / FAIL
- [ ] **2.6 LED 点亮**：通过 ImmurokService 测试命令逐路点 R/G/B（如无指令，至少观察启动 RED + 录入绿色） — 颜色对、不串色 — PASS / FAIL
- [ ] **2.7 BLE 广播**：nRF Connect 扫到 `immurok IK-1`，RSSI > -80dBm @ 1m — PASS / FAIL

### B 设备
- [ ] 复跑 2.1–2.7 — 全 PASS / 有 FAIL 记录：__________

### G 设备
- [ ] 复跑 2.1–2.7 — 全 PASS / 有 FAIL 记录：__________

---

## Phase 3 — 配对与认证主路径（≥ 3 台）

**前置：** 设备先恢复出厂（清旧配对），打开 App。

### W 设备
- [ ] **3.1 首次配对** —— App 显示"配对成功"，按设备 BTN 完成 handshake — PASS / FAIL
- [ ] **3.2 录入指纹（12 次 capture 全程）** — step 1 polling 期间 BLE 不掉线，最终 `FP_ENROLL_DONE`，App 列表显示新指纹 — PASS / FAIL
- [ ] **3.3 删除指纹** — App 删除按钮 → 列表实时更新 — PASS / FAIL
- [ ] **3.4 sudo 主动认证** — 终端 `imk run --agent -- sudo ls`，弹密码框 → 触摸 → 通过 — PASS / FAIL
- [ ] **3.5 sudo 被动认证** — 终端裸跑 `sudo ls` → 触摸 → 通过（无 imk 包装） — PASS / FAIL
- [ ] **3.6 锁屏自动解锁** — `Ctrl+Cmd+Q` 锁屏 → 触摸 → 直接进桌面 — PASS / FAIL
- [ ] **3.7 预发回车** — 锁屏后**先**触摸（无对话框）→ 出对话框 → **再**触摸 → 解锁 — PASS / FAIL
- [ ] **3.8 KEY_SIGN（SSH 签名）** — `imk run --agent -- git push`（任一已配 SSH 密钥的仓库） → 触摸 → 推送成功 — PASS / FAIL
- [ ] **3.9 KEY_OTP_GET** — `imk get imk://otp/<name>` → 返回 6 位 OTP — PASS / FAIL
- [ ] **3.10 KEY_GETPUB** — `imk list` → 显示公钥指纹 — PASS / FAIL
- [ ] **3.11 LED 状态指示** — KEY_* 操作期间黄灯，认证成功绿闪 1 下 — PASS / FAIL

### B 设备 — 复跑 3.1–3.11，记录差异：__________
### G 设备 — 复跑 3.1–3.11，记录差异：__________

---

## Phase 4 — BLE 稳定性 / 长跑（**至少 1 台 W**）

| # | 项目 | 时长 | 操作 | 通过判据 | 结果 |
|---|------|------|------|----------|------|
| 4.1 | 长连接静默 | ≥ 8h | App 开着不动 | 8h 后仍连接、可立即认证 | ⬜ |
| 4.2 | 远离 / 走回 | 5 分钟 | 拿设备走出 5m，等 60s，走回 | ≤ 30s 内自动重连 | ⬜ |
| 4.3 | Mac 睡眠唤醒 ×10 | 30 分钟 | 反复 sleep/wake | 每次都能重连，无需手动 | ⬜ |
| 4.4 | 反复触摸 | 30 分钟 | 连续 100 次触摸认证 | 无丢失、无 BLE 断 | ⬜ |
| 4.5 | 批量 KEY_SIGN | 10 分钟 | 脚本循环 50 次 ssh-sign | 全部成功，无 timeout | ⬜ |
| 4.6 | GET_STATUS 高频 | 30 分钟 | App 每 1s 刷新 | 无阻塞、无掉线 | ⬜ |
| 4.7 | 加密会话保持 | 24h | 配对后挂机 | 加密链路保持，无重新协商 | ⬜ |

---

## Phase 5 — OTA 升级

**前置：** 准备一个上一版本 imfw（如 1.3.0）烧到设备上，再升 1.3.1。

| # | 项目 | 操作 | 通过判据 | 结果 |
|---|------|------|----------|------|
| 5.1 | OTA 升级 | App 推送 fw-1.3.1-r.imfw（任意外壳设备）| 进度 100% → 重启 → 版本号 1.3.1 | ⬜ |
| 5.2 | _(已废弃)_ 1.3.1 起单一固件 SKU，无跨 variant | — | — | N/A |
| 5.3 中途断电 | 推到 50% 时拔电池 | 上电后旧 image 仍可用 | 设备能正常启动+认证 | ⬜ |
| 5.4 中途断 BLE | 推到 50% 时关 App | 重连后可重传或自动取消 | 不变砖 | ⬜ |
| 5.5 HMAC 篡改 | `xxd` 改 imfw header 1 字节 → 重推 | 设备返回 `OTA_ERR_HMAC_MISMATCH` (0xF2) | 拒收 | ⬜ |
| 5.6 SHA256 篡改 | `xxd` 改 payload 1 字节 → 重推 | 返回 `OTA_ERR_SHA256_MISMATCH` (0xF1) | 拒收 | ⬜ |
| 5.7 升级后首启动 | 1.3.1 升级完成后 | LED 启动闪 + bitmap 有指纹 + 配对仍在 | 全 OK | ⬜ |
| 5.8 跨 patch 升级 | 1.3.0 → 1.3.1 | 不丢配对、不丢 keystore | OK | ⬜ |
| 5.9 低电量拒 OTA | 放电到 < 5% 推 OTA | 设备返回 `0xF4`，OTA 中止 | 拒收 | ⬜ |

---

## Phase 6 — 极端 / 边界

| # | 项目 | 操作 | 通过判据 | 结果 |
|---|------|------|----------|------|
| 6.1 低电量 < 15% | 放电到 VBAT < 3.6V | LED 黄色提示（FP search 期间），认证仍正常 | OK | ⬜ |
| 6.1b 写操作拒绝 < 5% | 放电到 < 5% 后试 KEY_WRITE / KEY_GENERATE / 删除指纹 | 返回 `SEC_ERR_LOW_BATTERY` (0xF4)；KEY_SIGN/AUTH 仍可用 | OK | ⬜ |
| 6.2 极低电量 | 放电到 < 3.0V | 安全关机，重新充电后 EEPROM 数据完好 | OK | ⬜ |
| 6.3 充电中认证 | 接 USB 充电同时触摸 | 不影响指纹/BLE | OK | ⬜ |
| 6.4 指纹模组拔插（开发板） | 运行中拔指纹 FPC | 上报 FP_ERR，不死机 | OK | ⬜ |
| 6.5 EEPROM 写满 | 持续 KEY_GENERATE 直到 NO_SPACE | 旧条目不损坏，错误码正确 | OK | ⬜ |
| 6.6 恢复出厂 | App 走"恢复出厂"流程 | 配对清、keystore 清、bitmap 清；可立即重新配对 | OK | ⬜ |
| 6.7 长签名 + 触摸 | uECC 签名期间（~2s）持续触摸 | BLE 不断、不重启（stack guard 验证） | OK | ⬜ |
| 6.8 ESD / 拍打 | 手指刮一下设备外壳；轻拍桌面 | 不重启、不掉线 | OK | ⬜ |

---

## Phase 7 — 量产烧录线（产线工程师执行）

| # | 项目 | 操作 | 通过判据 | 结果 |
|---|------|------|----------|------|
| 7.1 一键烧录 | `burner ping → erase → flash combined.hex → fuse.json` 一条龙 | 全程零交互、零错误 | ⬜ |
| 7.2 Fuse 配置 | 烧完读 fuse | RESET_EN / DEBUG_EN / 读保护 = 出厂方案值 | ⬜ |
| 7.3 回读校验 | 烧完读取 flash → SHA256 | 与本地 hex 一致 | ⬜ |
| 7.4 节拍 | 单台从下板到下板 | ≤ 30s | ⬜ |
| 7.5 厂测自检 | 出厂自检脚本一次跑 | BLE 广播 + LED 自检 + 指纹 self-test 全 PASS | ⬜ |
| 7.6 SN 追溯 | 烧录日志 | 每台 MAC/SN 唯一、可追溯 | ⬜ |
| 7.7 防错色 | 故意烧 W 到 B 外壳 | 厂测 fail（颜色编码或人工核） | ⬜ |

---

## Phase 8 — 包装前抽检（每批 3–5%）

- [ ] 抽检台数：____ / 总数：____
- [ ] 复跑 Phase 2 全部 + Phase 3.1–3.6（每台）— 通过率 ___ %
- [ ] 出厂二维码 / SN / 颜色与盒子一致 — 不一致数：___
- [ ] 静置 24h 自放电 < 5%（电池） — 超标数：___

---

## Phase 9 — 发布最终签字

- [ ] **9.1** `git tag production-N` 已打（参考 `production-1` = 1.2.27 模式）
   - tag 名：__________
- [ ] **9.2** 单一 `firmware/dist/*.imfw` 已归档到 release 目录 / GitHub Release
   - 路径：__________
- [ ] **9.3** OTA 服务器灰度配置（如有）
- [ ] **9.4** App Store 提交 / 兼容矩阵更新
- [ ] **9.5** 回滚预案：上一个 stable 版本 imfw 备好
   - 路径：__________
- [ ] **9.6** 本批新坑写进 memory / `project_production_batches.md`

---

## 通过条件总结

- Phase 2、3、5（OTA） — **每台测试设备都必须 PASS**
- Phase 4.1（8h 长跑） — **至少 1 台 W 跑满**
- Phase 5.5/5.6（OTA 篡改拒绝） — **必须 PASS（安全性）**
- Phase 6 极端项 — 允许 NOTE 级别问题，但需在 release notes 明确
- Phase 7 — 量产线全流程 PASS 才能开线

---

## 异常记录区

| 项目 | 现象 | 复现率 | 严重度 | 决议 |
|------|------|--------|--------|------|
|      |      |        |        |      |
|      |      |        |        |      |

---

## 签字

| 角色 | 姓名 | 签字 | 日期 |
|------|------|------|------|
| 测试 |      |      |      |
| 固件负责 |      |      |      |
| 量产负责 |      |      |      |

---

## FAQ / 已知行为（非 bug）

测试时遇到下列现象**不要**记为缺陷：

### Q1：`imk list` 似乎每次都向设备读取
不是。流程是：
1. App 持本地 cache（`~/.immurok/ssh_keys.json`）
2. 每次 `imk list` 走一次 6 字节 BLE 往返拿设备的 (count, checksum)
3. 与 cache 匹配 → 直接返回 cache（**无逐条 KEY_READ**）
4. 不匹配 → 全量重新拉

`Console.app` 看 `SSHKeyCache: cache hit` vs `cache miss` 日志可确认。

### Q2：固件低电量阈值
1.3.1 起：

| VBAT 段 | 行为 |
|---------|------|
| > 3.6V (≥ 15%) | 正常，FP search LED 绿 |
| 3.0–3.6V (5–15%) | 正常，FP search LED **黄**（低电量提示） |
| 0.5%–4.99% | **写操作（KEY_WRITE / KEY_COMMIT / KEY_DELETE / KEY_GENERATE / OTA）拒绝**，返回 `SEC_ERR_LOW_BATTERY` (0xF4) |
| 写操作被拒后 | KEY_SIGN / AUTH / OTP_GET 仍可用（认证不受限） |
| < ~3.0V | 仍工作，App 显示 0% |
| ~2.5V | LDO dropout，CH592F BOR 触发，设备复位 |

**测试用例**：放电到 < 5%，尝试 `imk` 录入新指纹或 OTA → 应被拒绝并提示低电量。

### Q3：`imk run --agent -- git push` 无 SSH key 不再卡住（1.10+）
设备未配置 SSH key 时，`imk` 立即返回 EX_CONFIG (78) 而不是吃指纹后让 git 卡 stdin。如果你看到：
```
imk: git needs an SSH key but none is configured on the device.
     Configure via the immurok app → Settings → SSH keys, then retry.
```
→ 这是正常行为，不是 bug。先去 App 配 SSH key 再重试。

### Q4：被动认证时长按指纹**不会**触发锁屏（1.3.1+）
1.3.1 起 LOCK_HOLD_EVT 在以下情况被抑制：
- 有 pending gate 命令（KEY_SIGN / DELETE_FP / KEY_GENERATE / ENROLL 等）
- 有 pending AUTH_REQUEST

仅当用户**纯空闲**长按指纹（无任何待处理操作）才发 LOCK_REQUEST。

### Q5：固件不再区分颜色 variant（1.3.1+）
DIS Model Number 统一为 `"IK-1"`，无 W/B/G 后缀。三色外壳走同一份固件。dist/ 文件名也无 variant 后缀（`fw-1.3.1.<hash>-r.imfw` 单一文件）。
