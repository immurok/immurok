# Dual-Slot Fingerprint Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Each user fingerprint occupies two ZW3021 physical slots for improved recognition accuracy, with 12-capture enrollment (two phases of 6), transparent to the App.

**Architecture:** Firmware state machine expanded from 8 to ~16 steps with two capture-merge-store rounds. Mapping: `finger_id N → slot N*2, N*2+1`. Firmware translates all physical slot references before sending to App (bitmap, page_id, enroll response). App changes minimal: total=12, new adjust event.

**Tech Stack:** C (CH592F firmware), Swift (macOS App)

---

## Chunk 1: Firmware — Macros, Bitmap, and Search Conversion

### Task 1: Add dual-slot macros to fingerprint.h

**Files:**
- Modify: `firmware/APP/include/fingerprint.h:16` (replace FP_MAX_TEMPLATES usage context)
- Modify: `firmware/APP/include/fingerprint.h:62-67` (add FP_ENROLL_ADJUST)

- [ ] **Step 1: Add macros after line 16**

```c
// fingerprint.h, after FP_MAX_TEMPLATES line:

#define FP_USER_MAX             5       // Max user-visible fingerprints (adjustable: 6, 10, etc.)
#define FP_SLOTS_PER_FINGER     2       // Physical slots per fingerprint
#define FP_SLOT_MAX             (FP_USER_MAX * FP_SLOTS_PER_FINGER)  // 10

// Mapping: user finger_id → physical slot
#define FP_SLOT_FIRST(fid)      ((fid) * FP_SLOTS_PER_FINGER)
#define FP_SLOT_SECOND(fid)     ((fid) * FP_SLOTS_PER_FINGER + 1)

// Reverse mapping: physical slot → user finger_id
#define FP_FINGER_ID(slot)      ((slot) / FP_SLOTS_PER_FINGER)
```

- [ ] **Step 2: Add FP_ENROLL_ADJUST to fp_enroll_event_t enum**

In `fingerprint.h:62-67`, add after `FP_ENROLL_LIFT_FINGER = 3`:
```c
typedef enum {
    FP_ENROLL_WAITING = 0,
    FP_ENROLL_CAPTURED = 1,
    FP_ENROLL_PROCESSING = 2,
    FP_ENROLL_LIFT_FINGER = 3,
    FP_ENROLL_COMPLETE = 4,     // (already used in hidkbd.c as literal 4)
    FP_ENROLL_ADJUST = 5,       // Adjust finger angle for second slot
    FP_ENROLL_FAILED = 0xFF,    // (already used in hidkbd.c as literal 0xFF)
} fp_enroll_event_t;
```

- [ ] **Step 3: Update fp_get_fingerprint_bitmap signature**

Change `fingerprint.h:146` from:
```c
int fp_get_fingerprint_bitmap(uint8_t *bitmap);
```
to:
```c
int fp_get_fingerprint_bitmap(uint16_t *bitmap);
```

This returns the raw 10-bit bitmap. The caller (hidkbd.c) will do the user-level conversion.

- [ ] **Step 4: Commit**

```bash
git add firmware/APP/include/fingerprint.h
git commit -m "feat: 添加双 slot 指纹宏定义和 FP_ENROLL_ADJUST 事件"
```

### Task 2: Update fingerprint.c — bitmap and search range

**Files:**
- Modify: `firmware/APP/fingerprint.c:978-1005` (fp_get_fingerprint_bitmap)
- Modify: `firmware/APP/fingerprint.c:1240` (ENROLL_CAPTURE_COUNT stays 6, no change needed)

- [ ] **Step 1: Update fp_get_fingerprint_bitmap to return uint16_t**

In `fingerprint.c`, change the function signature and mask:
```c
int fp_get_fingerprint_bitmap(uint16_t *bitmap)
{
    // ... existing code up to the mask line ...

    // Raw bitmap: bits 0-9 for slots 0-9 (FP_SLOT_MAX)
    // s_rx_buf[10] has bits 0-7, s_rx_buf[11] has bits 8-15
    *bitmap = (uint16_t)(s_rx_buf[10]) | ((uint16_t)(s_rx_buf[11]) << 8);
    *bitmap &= ((1 << FP_SLOT_MAX) - 1);  // Keep only bits 0-9

    PRINT("Fingerprint raw bitmap: 0x%04X\n", *bitmap);

    return FP_OK;
}
```

- [ ] **Step 2: Commit**

```bash
git add firmware/APP/fingerprint.c
git commit -m "feat: fp_get_fingerprint_bitmap 返回 16-bit 原始位图"
```

### Task 3: Update hidkbd.c — global state and bitmap conversion

**Files:**
- Modify: `firmware/APP/hidkbd.c` — global variables and helper function

- [ ] **Step 1: Change g_cached_fp_bitmap to uint16_t and add conversion helper**

At `hidkbd.c:~130` area, change:
```c
// Before:
// (wherever g_cached_fp_bitmap is declared, likely near top or extern)
```

Find the declaration of `g_cached_fp_bitmap` and change from `uint8_t` to `uint16_t`. Then add a helper function:

```c
// Raw physical bitmap (10 bits for slots 0-9)
uint16_t g_cached_fp_bitmap = 0;

// Convert raw bitmap to user-visible bitmap
// Both physical slots must be present for a finger to count
static uint8_t fp_user_bitmap(void)
{
    uint8_t ubm = 0;
    for(int i = 0; i < FP_USER_MAX; i++) {
        if((g_cached_fp_bitmap & (1 << FP_SLOT_FIRST(i))) &&
           (g_cached_fp_bitmap & (1 << FP_SLOT_SECOND(i)))) {
            ubm |= (1 << i);
        }
    }
    return ubm;
}
```

- [ ] **Step 2: Update FP_LIST handler**

At `hidkbd.c:2358-2365`, change to use user bitmap:
```c
case IMMUROK_CMD_FP_LIST:
    {
        uint8_t ubm = fp_user_bitmap();
        rspBuf[0] = IMMUROK_RSP_OK;
        rspBuf[1] = ubm;
        rspLen = 2;
        PRINT("  FP_LIST: raw=0x%04X, user=0x%02X\n", g_cached_fp_bitmap, ubm);
    }
    break;
```

- [ ] **Step 3: Update search range in FP_SEARCH_EVT**

At `hidkbd.c:1124-1132`, change search range from `FP_MAX_TEMPLATES` to `FP_SLOT_MAX`:
```c
case 2:  // SEARCH
{
    uint8_t search_params[5];
    search_params[0] = 1;
    search_params[1] = 0;
    search_params[2] = 0;
    search_params[3] = 0;
    search_params[4] = FP_SLOT_MAX;  // was FP_MAX_TEMPLATES
    fp_send_cmd(0x04, search_params, 5);
```

- [ ] **Step 4: Convert page_id to user finger_id in match result**

At `hidkbd.c:1144`, after `uint16_t page_id = ...`:
```c
uint16_t page_id = (search_result[0] << 8) | search_result[1];
uint16_t match_score = (search_result[2] << 8) | search_result[3];
// Convert physical slot to user finger ID
page_id = FP_FINGER_ID(page_id);
```

This single conversion point ensures all downstream code (signed notification, gate responses, etc.) uses user finger_id.

- [ ] **Step 5: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat: 位图转换为用户视角 + 搜索范围和匹配 ID 转换"
```

---

## Chunk 2: Firmware — Enrollment State Machine

### Task 4: Expand enrollment state machine to dual-slot

**Files:**
- Modify: `firmware/APP/hidkbd.c:1437-1652` (FP_ENROLL_EVT handler)

The key change: the existing state machine captures 6 times into buffers 1-6, merges, stores to one slot. We expand it to:
- Round 1 (step 1-6): capture 6 times → merge → store to `finger_id*2`
- Adjust notification (step 7): send FP_ENROLL_ADJUST
- Round 2 (step 8-13): capture 6 times → merge → store to `finger_id*2+1`
- Complete (step 14)

If round 2 fails, delete round 1's slot (rollback).

- [ ] **Step 1: Add second-round state variables**

At `hidkbd.c`, near `s_enroll_page_id`:
```c
static uint16_t s_enroll_page_id = 0;      // User finger ID (0-4)
static uint8_t s_enroll_round = 0;          // 0=round1, 1=round2
```

- [ ] **Step 2: Update s_enroll_send_done handler**

At `hidkbd.c:1453-1465` (the `s_enroll_send_done` block), change the response to return user finger_id:
```c
if(s_enroll_send_done) {
    s_enroll_send_done = 0;
    fp_get_fingerprint_bitmap(&g_cached_fp_bitmap);
    uint8_t rspBuf[2];
    rspBuf[0] = IMMUROK_RSP_OK;
    rspBuf[1] = (uint8_t)s_enroll_page_id;  // Already user finger_id
    ImmurokService_SendResponse(rspBuf, 2);
    s_enroll_active = 0;
    enroll_step = 0;
    s_enroll_round = 0;
    fp_reset_power_timer();
    return (events ^ FP_ENROLL_EVT);
}
```

- [ ] **Step 3: Update step 0 (init)**

At `hidkbd.c:1474-1501`, change total from 6 to 12:
```c
if(enroll_step == 0) {
    PRINT("ENROLL_EVT: waking module for finger %d (slot %d,%d)\n",
          s_enroll_page_id, FP_SLOT_FIRST(s_enroll_page_id), FP_SLOT_SECOND(s_enroll_page_id));
    int wake_ret = fp_ensure_ready();
    if(wake_ret != FP_OK) {
        PRINT("ENROLL_EVT: wake failed %d\n", wake_ret);
        rspBuf[0] = 0x11;
        rspBuf[1] = FP_ENROLL_FAILED;
        rspBuf[2] = 0;
        rspBuf[3] = 0;
        ImmurokService_SendResponse(rspBuf, 4);
        s_enroll_active = 0;
        fp_reset_power_timer();
        return (events ^ FP_ENROLL_EVT);
    }
    PRINT("ENROLL_EVT: starting finger %d\n", s_enroll_page_id);
    s_enroll_round = 0;
    enroll_step = 1;
    enroll_start = TMOS_GetSystemClock();
#if HAS_RGB_LED
    led_blink_start('G');
#endif
    rspBuf[0] = 0x11;
    rspBuf[1] = FP_ENROLL_WAITING;
    rspBuf[2] = 0;
    rspBuf[3] = 12;     // total = 12
    ImmurokService_SendResponse(rspBuf, 4);
    tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 160);
}
```

- [ ] **Step 4: Update capture loop (steps 1-6)**

At `hidkbd.c:1518-1580`, the capture loop logic. Key changes:
- `capture_num` now considers the round: `round * 6 + local_step` for progress reporting
- After 6th capture, go to merge (step 7) instead of directly
- Use buffer IDs 1-6 in both rounds (ZW3021 reuses buffers per merge)

```c
else if(enroll_step >= 1 && enroll_step <= 6) {
    uint8_t local_step = enroll_step;
    uint8_t global_capture = s_enroll_round * 6 + local_step;  // 1-12 for progress

    fp_send_cmd(0x01, NULL, 0);  // CMD_GET_IMAGE
    uint8_t ack;
    int ret = fp_recv_ack(&ack, NULL, NULL, 50);
    WWDG_SetCounter(0);

    if(ret == FP_OK && ack == 0x00) {
        uint8_t buf_id = local_step;  // buffers 1-6
        uint8_t params[1] = { buf_id };
        fp_send_cmd(0x02, params, 1);  // CMD_IMAGE2TZ
        ret = fp_recv_ack(&ack, NULL, NULL, 100);
        WWDG_SetCounter(0);

        if(ret == FP_OK && ack == 0x00) {
            PRINT("ENROLL_EVT: captured %d/12 (round %d, local %d/6)\n", global_capture, s_enroll_round, local_step);
#if HAS_RGB_LED
            led_solid('G', 0);
#endif
            rspBuf[0] = 0x11;
            rspBuf[1] = FP_ENROLL_CAPTURED;
            rspBuf[2] = global_capture;
            rspBuf[3] = 12;
            ImmurokService_SendResponse(rspBuf, 4);

            enroll_step++;
            if(enroll_step <= 6) {
                s_enroll_send_lift = global_capture;
                tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 320);
            } else {
                enroll_step = 7;  // merge
                tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 160);
            }
        } else {
            PRINT("ENROLL_EVT: gen_char failed\n");
            tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 320);
        }
    }
    else {
        uint32_t elapsed = (TMOS_GetSystemClock() - enroll_start) * 625 / 1000;
        if(elapsed > 60000) {
            PRINT("ENROLL_EVT: timeout\n");
            rspBuf[0] = 0x11;
            rspBuf[1] = FP_ENROLL_FAILED;
            rspBuf[2] = 0;
            rspBuf[3] = 0;
            ImmurokService_SendResponse(rspBuf, 4);
            // Rollback if round 2 timeout
            if(s_enroll_round == 1) {
                fp_delete(FP_SLOT_FIRST(s_enroll_page_id), 1);
                PRINT("ENROLL_EVT: rolled back slot %d\n", FP_SLOT_FIRST(s_enroll_page_id));
            }
            s_enroll_active = 0;
            enroll_step = 0;
            s_enroll_round = 0;
            fp_reset_power_timer();
        } else {
            tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 320);
        }
    }
}
```

- [ ] **Step 5: Update lift finger handler**

At `hidkbd.c:1503-1517`, update total:
```c
else if(s_enroll_send_lift > 0) {
    uint8_t capture_num = s_enroll_send_lift;
    s_enroll_send_lift = 0;
#if HAS_RGB_LED
    led_blink_start('G');
#endif
    rspBuf[0] = 0x11;
    rspBuf[1] = FP_ENROLL_LIFT_FINGER;
    rspBuf[2] = capture_num;
    rspBuf[3] = 12;
    ImmurokService_SendResponse(rspBuf, 4);
    tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 768);
}
```

- [ ] **Step 6: Update merge step (step 7)**

At `hidkbd.c:1582-1609`, merge now stores to the current round's physical slot:
```c
else if(enroll_step == 7) {
    PRINT("ENROLL_EVT: merging round %d...\n", s_enroll_round);
    uint8_t global_capture = (s_enroll_round + 1) * 6;  // 6 or 12
    rspBuf[0] = 0x11;
    rspBuf[1] = FP_ENROLL_PROCESSING;
    rspBuf[2] = global_capture;
    rspBuf[3] = 12;
    ImmurokService_SendResponse(rspBuf, 4);
    fp_send_cmd(0x05, NULL, 0);  // CMD_REG_MODEL
    uint8_t ack;
    int ret = fp_recv_ack(&ack, NULL, NULL, 500);
    WWDG_SetCounter(0);

    if(ret == FP_OK && ack == 0x00) {
        enroll_step = 8;  // store
        tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 160);
    } else {
        PRINT("ENROLL_EVT: merge failed (round %d)\n", s_enroll_round);
        rspBuf[0] = 0x11;
        rspBuf[1] = FP_ENROLL_FAILED;
        rspBuf[2] = 0;
        rspBuf[3] = 0;
        ImmurokService_SendResponse(rspBuf, 4);
        // Rollback if round 2 merge fails
        if(s_enroll_round == 1) {
            fp_delete(FP_SLOT_FIRST(s_enroll_page_id), 1);
            PRINT("ENROLL_EVT: rolled back slot %d\n", FP_SLOT_FIRST(s_enroll_page_id));
        }
        s_enroll_active = 0;
        enroll_step = 0;
        s_enroll_round = 0;
        fp_reset_power_timer();
    }
}
```

- [ ] **Step 7: Update store step (step 8) — dual-round logic**

Replace `hidkbd.c:1611-1649` with:
```c
else if(enroll_step == 8) {
    // Store to physical slot based on current round
    uint16_t phys_slot = (s_enroll_round == 0)
        ? FP_SLOT_FIRST(s_enroll_page_id)
        : FP_SLOT_SECOND(s_enroll_page_id);
    PRINT("ENROLL_EVT: storing to physical slot %d (round %d)\n", phys_slot, s_enroll_round);
    uint8_t params[3];
    params[0] = 1;  // buffer 1
    params[1] = (phys_slot >> 8) & 0xFF;
    params[2] = phys_slot & 0xFF;
    fp_send_cmd(0x06, params, 3);  // CMD_STORE
    uint8_t ack;
    int ret = fp_recv_ack(&ack, NULL, NULL, 500);
    WWDG_SetCounter(0);

    if(ret == FP_OK && ack == 0x00) {
        if(s_enroll_round == 0) {
            // Round 1 done — send adjust notification, start round 2
            PRINT("ENROLL_EVT: round 1 stored, sending adjust\n");
            rspBuf[0] = 0x11;
            rspBuf[1] = FP_ENROLL_ADJUST;
            rspBuf[2] = 6;   // current progress
            rspBuf[3] = 12;  // total
            ImmurokService_SendResponse(rspBuf, 4);
            s_enroll_round = 1;
            enroll_step = 1;  // restart capture loop for round 2
            enroll_start = TMOS_GetSystemClock();  // reset timeout
#if HAS_RGB_LED
            led_blink_start('G');
#endif
            tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 1600);  // 1s pause for user to adjust
        } else {
            // Round 2 done — complete!
            PRINT("ENROLL_EVT: SUCCESS (both slots stored)!\n");
#if HAS_RGB_LED
            led_stop();
#endif
            rspBuf[0] = 0x11;
            rspBuf[1] = FP_ENROLL_COMPLETE;
            rspBuf[2] = 12;
            rspBuf[3] = 12;
            ImmurokService_SendResponse(rspBuf, 4);
            s_enroll_send_done = 1;
            tmos_start_task(hidEmuTaskId, FP_ENROLL_EVT, 160);
            return (events ^ FP_ENROLL_EVT);
        }
    } else {
        PRINT("ENROLL_EVT: store failed (round %d)\n", s_enroll_round);
        rspBuf[0] = 0x11;
        rspBuf[1] = FP_ENROLL_FAILED;
        rspBuf[2] = 0;
        rspBuf[3] = 0;
        ImmurokService_SendResponse(rspBuf, 4);
        // Rollback if round 2 store fails
        if(s_enroll_round == 1) {
            fp_delete(FP_SLOT_FIRST(s_enroll_page_id), 1);
            PRINT("ENROLL_EVT: rolled back slot %d\n", FP_SLOT_FIRST(s_enroll_page_id));
        }
        s_enroll_active = 0;
        enroll_step = 0;
        s_enroll_round = 0;
        fp_reset_power_timer();
    }
}
```

- [ ] **Step 8: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat: 注册状态机扩展为双 slot 12 次采集"
```

---

## Chunk 3: Firmware — Delete, Enroll-Start, and Remaining References

### Task 5: Update ENROLL_START command handler

**Files:**
- Modify: `firmware/APP/hidkbd.c:2368-2409`

- [ ] **Step 1: Update ENROLL_START to use FP_USER_MAX and user bitmap**

Replace `hidkbd.c:2368-2409`:
```c
case IMMUROK_CMD_ENROLL_START:
    if(payloadLen < 1) {
        rspBuf[0] = IMMUROK_RSP_INVALID_PARAM;
        break;
    }
    {
        uint8_t fingerId = pData[2];
        PRINT("  ENROLL_START finger=%d\n", fingerId);

        if(s_enroll_active) {
            rspBuf[0] = IMMUROK_RSP_BUSY;
        } else if(fingerId >= FP_USER_MAX) {
            rspBuf[0] = IMMUROK_RSP_INVALID_PARAM;
        } else {
            uint8_t ubm = fp_user_bitmap();
            if(ubm & (1 << fingerId)) {
                PRINT("  Finger %d already enrolled (user_bitmap=0x%02X)\n", fingerId, ubm);
                rspBuf[0] = IMMUROK_RSP_INVALID_PARAM;
                break;
            }

            if(ubm != 0) {
                PRINT("  FP gate: caching ENROLL_START, waiting for FP verify\n");
                s_pending_cmd = IMMUROK_CMD_ENROLL_START;
                s_pending_cmd_start = TMOS_GetSystemClock();
                s_pending_payload[0] = fingerId;
                s_pending_payload_len = 1;
                fp_gate_enter();
                rspBuf[0] = IMMUROK_RSP_WAIT_FP;
            } else {
                s_enroll_page_id = fingerId;
                s_enroll_active = 1;
                tmos_set_event(hidEmuTaskId, FP_ENROLL_EVT);
                rspBuf[0] = IMMUROK_RSP_OK;
            }
        }
    }
    break;
```

- [ ] **Step 2: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat: ENROLL_START 使用 FP_USER_MAX 和用户位图"
```

### Task 6: Update DELETE_FP to delete both physical slots

**Files:**
- Modify: `firmware/APP/hidkbd.c:2411-2436` (DELETE_FP command handler)
- Modify: `firmware/APP/hidkbd.c:1748-1764` (FP_GATE_EXEC_EVT DELETE handler)

- [ ] **Step 1: Update DELETE_FP command handler**

Replace `hidkbd.c:2411-2436`:
```c
case IMMUROK_CMD_DELETE_FP:
    if(payloadLen < 1) {
        rspBuf[0] = IMMUROK_RSP_INVALID_PARAM;
        break;
    }
    {
        uint8_t fingerId = pData[2];
        PRINT("  DELETE_FP finger=%d\n", fingerId);

        if(fingerId >= FP_USER_MAX) {
            rspBuf[0] = IMMUROK_RSP_INVALID_PARAM;
            break;
        }

        uint8_t ubm = fp_user_bitmap();
        if(ubm != 0) {
            PRINT("  FP gate: caching DELETE_FP, waiting for FP verify\n");
            s_pending_cmd = IMMUROK_CMD_DELETE_FP;
            s_pending_cmd_start = TMOS_GetSystemClock();
            s_pending_payload[0] = fingerId;
            s_pending_payload_len = 1;
            fp_gate_enter();
            rspBuf[0] = IMMUROK_RSP_WAIT_FP;
        } else {
            rspBuf[0] = IMMUROK_RSP_OK;
        }
    }
    break;
```

- [ ] **Step 2: Update FP_GATE_EXEC_EVT DELETE handler**

At `hidkbd.c:1748-1764`, change to delete both slots:
```c
if(s_pending_cmd == IMMUROK_CMD_DELETE_FP && s_latency_accepted)
{
    uint8_t fingerId = s_pending_payload[0];
    int del_ret = fp_ensure_ready();
    if(del_ret == FP_OK) {
        del_ret = fp_delete(FP_SLOT_FIRST(fingerId), 1);
        if(del_ret == FP_OK)
            fp_delete(FP_SLOT_SECOND(fingerId), 1);
        if(del_ret == FP_OK)
            fp_get_fingerprint_bitmap(&g_cached_fp_bitmap);
    }
    uint8_t rspDel[1];
    rspDel[0] = (del_ret == FP_OK) ? IMMUROK_RSP_OK : SEC_ERR_INTERNAL;
    ImmurokService_SendResponse(rspDel, 1);
    s_pending_cmd = 0;
    fp_reset_power_timer();
    return (events ^ OTA_FLASH_ERASE_EVT);
}
```

- [ ] **Step 3: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat: DELETE_FP 成对删除两个物理 slot"
```

### Task 7: Update remaining bitmap references

**Files:**
- Modify: `firmware/APP/hidkbd.c` — all remaining `g_cached_fp_bitmap` usages that compare as uint8_t

- [ ] **Step 1: Search and update all g_cached_fp_bitmap references**

Key locations to update:
1. Factory reset handler (`hidkbd.c:1206`): `g_cached_fp_bitmap = 0;` — works as-is (uint16_t = 0 is fine)
2. All `bitmap != 0` checks in GATT handler — these should use `fp_user_bitmap()` where checking user-visible state
3. The `extern` declaration (if any) — update to `uint16_t`

Find and update the extern declaration. Search for `extern.*g_cached_fp_bitmap`:
```c
// Change from:
extern uint8_t g_cached_fp_bitmap;
// To:
extern uint16_t g_cached_fp_bitmap;
```

Update FACTORY_RESET handler at `hidkbd.c:2521-2542` — the `bitmap != 0` check should use `fp_user_bitmap()`:
```c
case IMMUROK_CMD_FACTORY_RESET:
    PRINT("  FACTORY_RESET\n");
    {
        if(fp_user_bitmap() != 0) {
            // ... gate code unchanged
        } else {
            // ... direct reset code unchanged
        }
    }
    break;
```

Similarly update PAIR_INIT at `hidkbd.c:2456`:
```c
if(fp_user_bitmap() != 0 && fp_gate_needed()) {
```

- [ ] **Step 2: Commit**

```bash
git add firmware/APP/hidkbd.c
git commit -m "feat: 所有位图引用更新为 uint16_t + 用户位图"
```

### Task 8: Build and verify firmware compiles

- [ ] **Step 1: Build firmware**

```bash
cd firmware && make clean && make
```

Expected: Compiles without errors.

- [ ] **Step 2: Commit any fixups if needed**

---

## Chunk 4: App — Enrollment UI Changes

### Task 9: Add adjust event to BLEManager.swift

**Files:**
- Modify: `app-macos/Sources/BLEManager.swift:88-95`

- [ ] **Step 1: Add adjust case to EnrollEvent enum**

```swift
enum EnrollEvent: UInt8 {
    case waiting = 0x00
    case captured = 0x01
    case processing = 0x02
    case liftFinger = 0x03
    case complete = 0x04
    case adjust = 0x05      // NEW: adjust finger angle for second slot
    case failed = 0xFF
}
```

- [ ] **Step 2: Commit**

```bash
git add app-macos/Sources/BLEManager.swift
git commit -m "feat: 添加 EnrollEvent.adjust 枚举值"
```

### Task 10: Update FingerprintView enrollment handling

**Files:**
- Modify: `app-macos/Sources/FingerprintView.swift:311` (enrollmentTotal)
- Modify: `app-macos/Sources/FingerprintView.swift:444-484` (onEnrollStatus callback)

- [ ] **Step 1: Change enrollmentTotal default to 12**

At `FingerprintView.swift:311`:
```swift
// Before:
@Published var enrollmentTotal = 6  // Match firmware ENROLL_CAPTURE_COUNT
// After:
@Published var enrollmentTotal = 12  // Dual-slot: 2 rounds × 6 captures
```

- [ ] **Step 2: Add adjust case to onEnrollStatus handler**

At `FingerprintView.swift:452-484`, add the adjust case:
```swift
switch event {
case .waiting:
    self.enrollmentStatus = "enroll.place.finger".localized
    self.enrollmentProgress = current
case .captured:
    self.enrollmentStatus = "enroll.captured".localized(current, total)
    self.enrollmentProgress = current
case .liftFinger:
    self.enrollmentStatus = "enroll.lift.finger".localized
    self.enrollmentProgress = current
case .adjust:
    self.enrollmentStatus = "enroll.adjust.angle".localized
    self.enrollmentProgress = current
case .processing:
    self.enrollmentStatus = "enroll.processing".localized
    self.enrollmentProgress = current
case .complete:
    // ... existing code unchanged
case .failed:
    // ... existing code unchanged
}
```

- [ ] **Step 3: Commit**

```bash
git add app-macos/Sources/FingerprintView.swift
git commit -m "feat: 录入进度 12 步 + adjust 角度调整提示"
```

### Task 11: Add localization strings

**Files:**
- Modify: `app-macos/Sources/LocalizationManager.swift` (all language sections)
- Modify: `app-macos/Resources/Localization/template.json`
- Modify: `app-macos/Resources/Localization/*.json` (all locale files)

- [ ] **Step 1: Add to LocalizationManager.swift Chinese (Simplified)**

After `"enroll.lift.finger"` line:
```swift
"enroll.adjust.angle": "请稍微调整手指角度，再次按下...",
```

- [ ] **Step 2: Add to Chinese (Traditional)**

```swift
"enroll.adjust.angle": "請稍微調整手指角度，再次按下...",
```

- [ ] **Step 3: Add to English**

```swift
"enroll.adjust.angle": "Slightly adjust your finger angle, then press again...",
```

- [ ] **Step 4: Add to template.json and all locale JSON files**

After `"enroll.lift.finger"`:
```json
"enroll.adjust.angle": "",
```

- [ ] **Step 5: Commit**

```bash
git add app-macos/Sources/LocalizationManager.swift app-macos/Resources/Localization/
git commit -m "feat: 添加指纹角度调整提示本地化字符串"
```

### Task 12: Build and verify App compiles

- [ ] **Step 1: Build App**

```bash
cd app-macos && swift build
```

Expected: Compiles without errors.

- [ ] **Step 2: Commit any fixups**

---

## Chunk 5: Full Build and OTA Test

### Task 13: Full firmware build + OTA package

- [ ] **Step 1: Build release-debug firmware**

```bash
ota/upload-ota.sh release-debug
```

Expected: Compiles, packages OTA, and flashes successfully.

- [ ] **Step 2: Build and deploy App**

```bash
app-macos/build-deploy.sh -a -s
```

- [ ] **Step 3: Functional test checklist**

Manual testing:
1. Open App → Settings → Fingerprints
2. Click "Add" → verify progress shows x/12
3. After 6th capture → verify "adjust angle" prompt appears
4. Complete all 12 captures → verify success
5. Verify fingerprint unlocks screen
6. Delete fingerprint → verify both slots cleared
7. Re-enroll → verify works on same slot
8. If round 2 fails (remove finger during capture timeout) → verify round 1 is rolled back, no orphan slot
