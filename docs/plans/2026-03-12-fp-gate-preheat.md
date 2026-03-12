# 指纹传感器统一预热 + LED 反馈 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 所有需要指纹验证的固件命令统一预热传感器 + LED 反馈，替换 AUTH_REQUEST 的特殊预热路径。

**Architecture:** 新增 `fp_gate_enter()` 函数统一预热逻辑。在 `fp_gate_needed()` 返回 true 后调用：上电 + 绿灯闪烁 + 等待触摸。匹配成功蓝灯，失败红灯，3 次失败或超时终止。VER0 用 ZW3021 LED，VER2 用板载 RGB LED。

**Tech Stack:** C (CH592F WCH SDK), TMOS cooperative scheduler

**Design doc:** `docs/plans/2026-03-12-fp-gate-preheat-design.md`

---

### Task 1: 新增状态变量，替换 `s_auth_preheat`

**Files:**
- Modify: `firmware/APP/hidkbd.c:131` (s_auth_preheat 定义)
- Modify: `firmware/APP/hidkbd.c:858-862` (FP_WAKE_DONE_EVT 中的 s_auth_preheat 分支)

**Step 1: 替换状态变量定义**

在 `firmware/APP/hidkbd.c` 第 131 行附近，将：

```c
static uint8_t s_auth_preheat = 0;
```

替换为：

```c
static uint8_t s_gate_preheat = 0;        // FP gate preheat: module powered, waiting for touch
static uint8_t s_gate_fail_count = 0;      // FP match failure count in current gate session
#define FP_GATE_MAX_RETRIES  3             // Max fingerprint attempts before gate failure
```

**Step 2: 更新 FP_WAKE_DONE_EVT 中的 preheat 分支**

在 `firmware/APP/hidkbd.c` 第 858-862 行，将：

```c
        if(s_auth_preheat) {
            // AUTH_REQUEST preheat: module is ready, wait for touch GPIO
            s_auth_preheat = 0;
            PRINT("FP ready, waiting for touch...\n");
            return (events ^ FP_WAKE_DONE_EVT);
        }
```

替换为：

```c
        if(s_gate_preheat) {
            // Gate preheat: module is ready, start green LED + wait for touch
            s_gate_preheat = 0;
            PRINT("FP ready, green LED, waiting for touch...\n");
#if HAS_RGB_LED
            led_blink_start('G');
#else
            fp_led_flash(FP_LED_GREEN, 20, 0);  // continuous flash
#endif
            return (events ^ FP_WAKE_DONE_EVT);
        }
```

**Step 3: 编译验证**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1/firmware && make clean && make`
Expected: 编译成功，无 warning（`s_auth_preheat` 未使用的 warning 预期会出现，下个 task 修复）

**Step 4: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat(fw): 新增 s_gate_preheat 替换 s_auth_preheat + 绿灯闪烁"
```

---

### Task 2: 实现 `fp_gate_enter()` 函数

**Files:**
- Modify: `firmware/APP/hidkbd.c` (在 `fp_gate_needed()` 附近添加新函数，约 line 2028)

**Step 1: 添加 fp_gate_enter() 函数**

在 `fp_gate_needed()` 函数（line 2027）之后添加：

```c
/* Unified FP gate preheat: power on + green LED + wait for touch.
 * Called from command handlers after fp_gate_needed() returns true.
 * Module will be ready for instant search when user touches sensor. */
static void fp_gate_enter(void)
{
    s_gate_fail_count = 0;

    if(!fp_is_powered()) {
        fp_power_on();
        s_fp_power_on_tick = RTC_GetCycle32k();
        s_gate_preheat = 1;
        tmos_start_task(hidEmuTaskId, FP_WAKE_DONE_EVT, 48);  // 30ms
    } else if(!fp_is_ready()) {
        s_gate_preheat = 1;
        tmos_start_task(hidEmuTaskId, FP_WAKE_DONE_EVT, 16);  // 10ms
    } else {
        // Already powered and ready — just start green LED
        PRINT("FP already ready, green LED\n");
#if HAS_RGB_LED
        led_blink_start('G');
#else
        fp_led_flash(FP_LED_GREEN, 20, 0);
#endif
    }
}
```

**Step 2: 编译验证**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1/firmware && make clean && make`
Expected: 编译成功（`fp_gate_enter` 会有 unused 警告，下个 task 使用后消除）

**Step 3: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat(fw): 添加 fp_gate_enter() 统一预热函数"
```

---

### Task 3: 改造命令 Handler — 调用 `fp_gate_enter()`

**Files:**
- Modify: `firmware/APP/hidkbd.c` — 6 个命令 handler

**Step 1: AUTH_REQUEST (line 2150-2170)**

将整个预热逻辑替换为 `fp_gate_enter()` 调用：

```c
    case IMMUROK_CMD_AUTH_REQUEST:
        // No payload needed
        PRINT("  AUTH_REQUEST\n");
        {
            immurok_security_set_auth_state(AUTH_STATE_WAIT_FINGERPRINT);
            rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            fp_gate_enter();
        }
        break;
```

注意：AUTH_REQUEST 不走 `fp_gate_needed()` 检查——它总是需要指纹验证（PAM 认证不受 cooldown 影响）。

**Step 2: KEY_DELETE (line 2369-2393)**

在 `rspBuf[0] = IMMUROK_RSP_WAIT_FP;` 之前添加 `fp_gate_enter();`：

```c
            if(fp_gate_needed()) {
                PRINT("  FP gate: caching KEY_DELETE\n");
                s_pending_cmd = IMMUROK_CMD_KEY_DELETE;
                s_pending_cmd_start = TMOS_GetSystemClock();
                s_pending_payload[0] = cat;
                s_pending_payload[1] = idx;
                s_pending_payload_len = 2;
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            }
```

**Step 3: KEY_SIGN (line 2425-2467)**

在 fp_gate_needed() 分支中添加 `fp_gate_enter();`：

```c
            if(fp_gate_needed()) {
                PRINT("  FP gate: caching KEY_SIGN\n");
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            }
```

**Step 4: KEY_GENERATE (line 2490-2514)**

同样模式：

```c
            if(fp_gate_needed()) {
                PRINT("  FP gate: caching KEY_GENERATE\n");
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            }
```

**Step 5: DELETE_FP (line 2124-2148)**

```c
            if(bitmap != 0) {
                PRINT("  FP gate: caching DELETE_FP, waiting for FP verify\n");
                s_pending_cmd = IMMUROK_CMD_DELETE_FP;
                s_pending_cmd_start = TMOS_GetSystemClock();
                s_pending_payload[0] = slotId;
                s_pending_payload_len = 1;
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            }
```

**Step 6: ENROLL_START (line 2106-2112)**

```c
                if(bitmap != 0) {
                    PRINT("  FP gate: caching ENROLL_START, waiting for FP verify\n");
                    s_pending_cmd = IMMUROK_CMD_ENROLL_START;
                    s_pending_cmd_start = TMOS_GetSystemClock();
                    s_pending_payload[0] = slotId;
                    s_pending_payload_len = 1;
                    fp_gate_enter();
                    rspBuf[0] = IMMUROK_RSP_WAIT_FP;
                }
```

**Step 7: PAIR_INIT (line 2176-2182)**

```c
            if(g_cached_fp_bitmap != 0 && fp_gate_needed()) {
                PRINT("  FP gate: caching PAIR_INIT, waiting for FP verify\n");
                s_pending_cmd = IMMUROK_CMD_PAIR_INIT;
                s_pending_cmd_start = TMOS_GetSystemClock();
                s_pending_payload_len = 0;
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
                break;
            }
```

**Step 8: KEY_OTP_GET (line 2559-2569)**

```c
            if(fp_gate_needed()) {
                PRINT("  FP gate: caching KEY_OTP_GET\n");
                s_pending_cmd = IMMUROK_CMD_KEY_OTP_GET;
                s_pending_cmd_start = TMOS_GetSystemClock();
                s_pending_payload[0] = idx;
                s_pending_payload[1] = pData[3];
                s_pending_payload[2] = pData[4];
                s_pending_payload[3] = pData[5];
                s_pending_payload[4] = pData[6];
                s_pending_payload_len = 5;
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            }
```

**Step 9: 编译验证**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1/firmware && make clean && make`
Expected: 编译成功，无 warning

**Step 10: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat(fw): 所有 gated 命令统一调用 fp_gate_enter() 预热"
```

---

### Task 4: 失败计数 + 最终失败/超时 LED 反馈

**Files:**
- Modify: `firmware/APP/hidkbd.c` — FP_SEARCH_EVT match fail 路径 (line 1200-1234)
- Modify: `firmware/APP/hidkbd.c` — FP_SEARCH_EVT timeout 路径 (line 920-957)

**Step 1: match fail 路径添加失败计数和 LED 恢复**

在 line 1200-1234 的 no-match 分支中，替换为：

```c
            else
            {
#if HAS_RGB_LED
                led_solid('R', LED_FLASH_TICKS);
#else
                fp_led_flash(FP_LED_RED, 25, 1);
#endif
                PRINT("FP no match (ack=0x%02X)\n", ack);

                // Track gate failure count
                if(s_pending_cmd != 0 || immurok_security_has_pending_auth())
                {
                    s_gate_fail_count++;
                    PRINT("FP gate fail count: %d/%d\n", s_gate_fail_count, FP_GATE_MAX_RETRIES);

                    if(s_gate_fail_count >= FP_GATE_MAX_RETRIES)
                    {
                        // Max retries reached — terminate gate
                        PRINT("FP gate: max retries reached, cancelling\n");
#if HAS_RGB_LED
                        // Red blink for 1s then off (LED_OFF_EVT will fire)
                        led_blink_start('R');
                        tmos_start_task(s_led_task_id, LED_OFF_EVT, 1600);  // 1s
#else
                        fp_led_flash(FP_LED_RED, 15, 3);  // fast flash 3x
#endif
                        if(s_pending_cmd != 0) {
                            uint8_t rspBuf[1] = { 0x07 };  // AUTH_FAIL
                            ImmurokService_SendResponse(rspBuf, 1);
                            s_pending_cmd = 0;
                            s_pending_payload_len = 0;
                        }
                        if(immurok_security_has_pending_auth()) {
                            uint8_t rspBuf[1] = { SEC_ERR_AUTH_FAILED };
                            ImmurokService_SendResponse(rspBuf, 1);
                            immurok_security_auth_cancel();
                        }
                        s_gate_fail_count = 0;
                        fp_power_off();
                        break;
                    }

                    // Not max retries — notify App + restore green LED after red flash
                    uint8_t notifyBuf[1] = { 0x07 };  // FP_NOT_MATCH
                    ImmurokService_SendResponse(notifyBuf, 1);

                    // Check overall gate timeout
                    if(s_pending_cmd != 0)
                    {
                        uint32_t gate_elapsed = (TMOS_GetSystemClock() - s_pending_cmd_start) * 625 / 1000;
                        if(gate_elapsed > FP_GATE_TIMEOUT_MS)
                        {
                            PRINT("FP gate timeout (%dms), cancelling pending cmd 0x%02X\n",
                                  (int)gate_elapsed, s_pending_cmd);
#if HAS_RGB_LED
                            led_blink_start('R');
                            tmos_start_task(s_led_task_id, LED_OFF_EVT, 1600);
#else
                            fp_led_flash(FP_LED_RED, 15, 3);
#endif
                            s_pending_cmd = 0;
                            s_pending_payload_len = 0;
                            uint8_t rspBuf[1] = { 0x07 };
                            ImmurokService_SendResponse(rspBuf, 1);
                            s_gate_fail_count = 0;
                            fp_power_off();
                            break;
                        }
                    }
                }
                else
                {
                    // Proactive match (no pending cmd/auth) — just red flash, no counting
                }

                // Keep power on for retry via next touch IRQ
                fp_reset_power_timer();
            }
```

**Step 2: 单次失败后恢复绿灯**

在 TOUCH_SCAN_EVT 的触摸确认路径（line 769-800），现有代码已经在触摸时设置 `led_solid('G', 0)`（line 774）。预热模式下，触摸时绿灯已在闪烁，触摸后会进入 FP_AUTH_EVT → FP_SEARCH_EVT。如果失败，红灯闪完后下次触摸会再次触发。

需要在 TOUCH_SCAN_EVT 中：当有 pending gate 时，恢复绿色闪烁而非绿色常亮：

```c
#if HAS_RGB_LED
                    if(s_pending_cmd != 0 || immurok_security_has_pending_auth()) {
                        led_blink_start('G');  // restore blink for gate mode
                    } else {
                        led_solid('G', 0);
                    }
#endif
```

**Step 3: search timeout 路径添加 LED 反馈**

在 FP_SEARCH_EVT timeout 路径（line 920-957），当有 pending cmd 且 gate 超时时，添加红灯快闪：

在 `gate_elapsed > FP_GATE_TIMEOUT_MS` 分支中（line 930-938）：

```c
                if(gate_elapsed > FP_GATE_TIMEOUT_MS)
                {
                    PRINT("FP search timeout + gate expired (%dms), clearing cmd 0x%02X\n",
                          (int)gate_elapsed, s_pending_cmd);
#if HAS_RGB_LED
                    led_blink_start('R');
                    tmos_start_task(s_led_task_id, LED_OFF_EVT, 1600);
#else
                    fp_led_flash(FP_LED_RED, 15, 3);
#endif
                    s_pending_cmd = 0;
                    s_pending_payload_len = 0;
                    s_gate_fail_count = 0;
                    uint8_t rspBuf[1] = { 0x07 };
                    ImmurokService_SendResponse(rspBuf, 1);
                    fp_power_off();
                }
```

**Step 4: 编译验证**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1/firmware && make clean && make`
Expected: 编译成功

**Step 5: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat(fw): 失败计数 + 最终失败/超时红灯反馈 + 绿灯恢复"
```

---

### Task 5: 匹配成功 LED — 统一蓝灯逻辑

**Files:**
- Modify: `firmware/APP/hidkbd.c` — FP_SEARCH_EVT match success (line 1018-1034)

**Step 1: 统一蓝灯 + 条件编译**

现有代码（line 1022-1025）：

```c
                fp_led_flash(FP_LED_BLUE, 25, 1);
#if HAS_RGB_LED
                led_solid('B', LED_FLASH_TICKS);
#endif
```

替换为（互斥，不同时调用两种 LED）：

```c
#if HAS_RGB_LED
                led_solid('B', 1600);  // 蓝灯 1s (1600 ticks = 1s)
#else
                fp_led_flash(FP_LED_BLUE, 25, 1);
#endif
```

同时重置失败计数器（在蓝灯之后，power_off 之前）：

```c
                s_gate_fail_count = 0;
```

**Step 2: 编译验证**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1/firmware && make clean && make`
Expected: 编译成功

**Step 3: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat(fw): 匹配成功蓝灯统一为互斥条件编译 + 重置失败计数"
```

---

### Task 6: 清理 + 其他 LED 调用点统一

**Files:**
- Modify: `firmware/APP/hidkbd.c` — 所有剩余的同时调用 fp_led_flash + led_solid 的地方

**Step 1: search timeout 无 pending 时的红灯 (line 951-954)**

```c
                fp_led_flash(FP_LED_RED, 25, 1);
#if HAS_RGB_LED
                led_solid('R', LED_FLASH_TICKS);
#endif
```

替换为：

```c
#if HAS_RGB_LED
                led_solid('R', LED_FLASH_TICKS);
#else
                fp_led_flash(FP_LED_RED, 25, 1);
#endif
```

**Step 2: 确认删除所有 s_auth_preheat 引用**

搜索确认 `s_auth_preheat` 不再出现在代码中。Task 1 已替换定义和 FP_WAKE_DONE_EVT 用法，Task 3 已替换 AUTH_REQUEST 用法。

Run: `grep -n s_auth_preheat firmware/APP/hidkbd.c`
Expected: 无输出

**Step 3: 编译验证**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1/firmware && make clean && make`
Expected: 编译成功，无 warning

**Step 4: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "refactor(fw): LED 调用统一为互斥条件编译 + 清理 s_auth_preheat"
```

---

### Task 7: 功能测试

**Step 1: 烧录固件**

Run: `cd /Users/katsu/Documents/Projects/imPress-v1 && ota/upload-ota.sh release-debug`
Expected: 编译 + 烧录成功

**Step 2: 测试 SSH 签名（KEY_SIGN gate + preheat）**

```bash
ssh -T git@github.com
```

Expected:
1. 设备收到 KEY_SIGN → 绿灯开始闪烁（~1Hz）
2. 按指纹 → 匹配成功 → 蓝灯亮 1s → 签名完成
3. 终端显示 "Hi xxx! You've successfully authenticated"

**Step 3: 测试失败重试**

用未注册的手指触摸 3 次：
1. 第 1 次：红灯闪 → 恢复绿灯闪烁
2. 第 2 次：红灯闪 → 恢复绿灯闪烁
3. 第 3 次：红灯快闪 3 下 → 关电源 → SSH 报错

**Step 4: 测试 AUTH_REQUEST（sudo）**

```bash
sudo echo test
```

Expected: 同 KEY_SIGN 的 LED 行为，PAM 验证通过

**Step 5: 测试超时**

触发 KEY_SIGN 后不按指纹，等待 25s：
Expected: 红灯快闪 → 关电源 → SSH 报错

**Step 6: Commit test results**

如果测试通过，创建最终 commit：

```bash
git commit --allow-empty -m "test(fw): 指纹预热 + LED 反馈功能测试通过"
```
