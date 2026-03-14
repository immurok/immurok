# 双 Slot 指纹识别设计

## 目标

通过让每个用户指纹占用两个 ZW3021 物理 slot，提升指纹识别精确度。用户体验不变（最多 5 个指纹），但每个指纹录入 12 次（两轮各 6 次），两个模板覆盖不同按压角度。

## 核心决策

| 决策 | 选择 |
|------|------|
| Slot 映射 | 连续映射：finger_id N → slot N*2, N*2+1 |
| 录入体验 | 分两阶段，中间提示"调整手指角度" |
| 匹配返回 | 固件端转换 page_id → user finger_id |
| 位图传输 | 固件返回用户视角 5-bit 位图 |
| 失败处理 | 全部回滚，不保留半成品 |
| 实现方案 | 固件状态机扩展，App 改动最小化 |

## 固件层设计

### 宏定义

```c
// fingerprint.h
#define FP_USER_MAX          5    // 用户可见最大指纹数（可改为 6、10）
#define FP_SLOTS_PER_FINGER  2    // 每个指纹占用的物理 slot 数
#define FP_SLOT_MAX          (FP_USER_MAX * FP_SLOTS_PER_FINGER)  // 10

#define FP_SLOT_FIRST(finger_id)  ((finger_id) * FP_SLOTS_PER_FINGER)
#define FP_SLOT_SECOND(finger_id) ((finger_id) * FP_SLOTS_PER_FINGER + 1)
```

### 注册状态机

现有 8 步扩展为两轮：

| 步骤 | 动作 |
|------|------|
| 0 | 唤醒模块 |
| 1-6 | 采集 6 次 → 合并 → 存储到 slot finger_id*2（第一轮） |
| 7 | 发送 FP_ENROLL_ADJUST 事件，提示调整角度 |
| 8-13 | 采集 6 次 → 合并 → 存储到 slot finger_id*2+1（第二轮） |
| 14 | 完成 |

通知格式不变：`[0x11, event, current, total]`，total=12。

新增事件值：
```c
FP_ENROLL_ADJUST = 5  // 提示用户调整角度
```

失败回滚：第二轮失败时调用 `fp_delete(finger_id*2, 1)` 清除第一轮。

### 删除逻辑

```c
fp_delete(FP_SLOT_FIRST(finger_id), 1);
fp_delete(FP_SLOT_SECOND(finger_id), 1);
```

### 位图转换

FP_LIST 返回用户视角位图：
```c
uint16_t raw_bitmap;
fp_get_fingerprint_bitmap(&raw_bitmap);
uint8_t user_bitmap = 0;
for(int i = 0; i < FP_USER_MAX; i++) {
    if((raw_bitmap & (1 << FP_SLOT_FIRST(i))) &&
       (raw_bitmap & (1 << FP_SLOT_SECOND(i)))) {
        user_bitmap |= (1 << i);
    }
}
```

### 匹配结果转换

```c
uint16_t user_finger_id = page_id / FP_SLOTS_PER_FINGER;
```

### 参数校验

- ENROLL_START: `finger_id >= FP_USER_MAX` → INVALID_PARAM
- 位图检查使用用户位图
- 原有 `slotId >= 5` 硬编码改为 `finger_id >= FP_USER_MAX`

## App 层设计

### 改动范围

1. **ENROLL_TOTAL**：6 → 12
2. **新增 EnrollEvent**：`.adjust = 0x05`，显示"请稍微调整手指角度"
3. **进度显示**：适配 12 步

### 不变部分

- 删除逻辑：无需改动
- 位图/匹配逻辑：无需改动（固件已转换）
- 指纹名称管理：无需改动
- 最大指纹数限制：由固件位图控制

## 涉及文件

| 文件 | 改动 |
|------|------|
| `firmware/APP/include/fingerprint.h` | 新增宏定义、FP_ENROLL_ADJUST |
| `firmware/APP/hidkbd.c` | 状态机扩展、删除成对、位图转换、匹配转换、参数校验 |
| `firmware/APP/fingerprint.c` | fp_get_fingerprint_bitmap 返回类型可能需调整 |
| `app-macos/Sources/FingerprintView.swift` | ENROLL_TOTAL、adjust 事件处理 |
| `app-macos/Sources/BLEManager.swift` | EnrollEvent 新增 adjust case |
