# 指纹传感器统一预热 + LED 反馈

## 背景

当前只有 AUTH_REQUEST 会预热指纹传感器（`s_auth_preheat`），KEY_SIGN、KEY_GENERATE、KEY_DELETE 等 gated 命令不预热——模块保持关电源状态直到用户触摸后才上电，体验不一致且响应慢。

## 目标

所有需要指纹验证的命令统一：收到命令后立即预热传感器 + LED 提示用户按指纹 + 匹配结果 LED 反馈。

## 设计决策

1. **统一预热**：所有 gated 命令（含 AUTH_REQUEST）走同一套 `fp_gate_enter()` 逻辑，删除 `s_auth_preheat` 特殊路径。
2. **预热时机**：在 `fp_gate_needed()` 返回 true 后才预热，尊重 10s cooldown，避免浪费电。
3. **LED 分版本**：VER0 用 ZW3021 内部 LED（UART 命令），VER1/VER2 用板载 RGB LED（TMOS 非阻塞）。

## 架构

### 核心函数：`fp_gate_enter()`

在所有 gated 命令 handler 中，`fp_gate_needed()` 返回 true 后调用：

```c
static void fp_gate_enter(void)
{
    s_gate_fail_count = 0;
    if (!fp_is_powered()) {
        fp_power_on();
        s_fp_power_on_tick = RTC_GetCycle32k();
    }
    s_gate_preheat = 1;  // 替换 s_auth_preheat
    tmos_start_task(hidEmuTaskId, FP_WAKE_DONE_EVT, 48);  // 30ms
}
```

FP_WAKE_DONE_EVT 中：当 `s_gate_preheat` 时发送绿灯闪烁命令，清除标志，等待触摸。

### LED 反馈矩阵

| 阶段 | VER2 板载 RGB | VER0 ZW3021 |
|------|-------------|-------------|
| 等待触摸 | `led_blink_start('G')` ~1Hz | `fp_led_flash(GREEN, 20, 0)` |
| 匹配成功 | `led_solid('B', 1s)` | `fp_led_flash(BLUE, 25, 1)` |
| 单次失败 | `led_solid('R', 500ms)` → 恢复绿闪 | `fp_led_flash(RED, 25, 1)` → 恢复绿闪 |
| 最终失败(3次) | `led_blink_start('R')` 1s → 关 | `fp_led_flash(RED, 15, 3)` |
| 超时(25s) | `led_blink_start('R')` 1s → 关 | `fp_led_flash(RED, 15, 3)` |

条件编译 `#if HAS_RGB_LED` 选择实现，不新建抽象层。

### 改动的命令 Handler

所有 gated 命令统一模式：

```c
if(fp_gate_needed()) {
    s_pending_cmd = IMMUROK_CMD_XXX;
    s_pending_cmd_start = TMOS_GetSystemClock();
    // 缓存 payload...
    fp_gate_enter();           // 新增
    rspBuf[0] = IMMUROK_RSP_WAIT_FP;
}
```

涉及：AUTH_REQUEST, KEY_SIGN, KEY_GENERATE, KEY_DELETE, DELETE_FP, PAIR_INIT。

### 失败计数与超时

```c
static uint8_t s_gate_fail_count = 0;
#define FP_GATE_MAX_RETRIES  3
```

- `fp_gate_enter()` 重置计数器
- match fail → 计数 +1，< 3 则红灯 0.5s 后恢复绿闪，>= 3 则红灯快闪 1s 后关电源并返回失败
- match success → 蓝灯 1s 后关电源，执行缓存命令
- gate timeout 25s → 红灯快闪 1s 后关电源并返回失败

### 删除的代码

- `s_auth_preheat` 变量及 AUTH_REQUEST / FP_WAKE_DONE_EVT 中的所有引用
- AUTH_REQUEST handler 中的独立预热逻辑（`fp_power_on` + `FP_WAKE_DONE_EVT`）

## 不变的部分

- `fp_gate_needed()` 逻辑不变（bitmap + 10s cooldown）
- `FP_GATE_TIMEOUT_MS` 保持 25s
- `FP_GATE_EXEC_EVT` 执行逻辑不变
- TOUCH_SCAN_EVT 触摸检测不变
- App 端不需要改动（响应码 WAIT_FP 不变）
