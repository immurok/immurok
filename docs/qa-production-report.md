# 量产 QA 自动化检查报告

- 固件版本：**1.3.1.c2f6** / VER=5 / **单一 SKU**（W/B/G 共用）
- App 版本：1.10+ / 1.11
- 生成时间：2026-05-08
- 检查范围：Phase 0（代码冻结）+ Phase 1（构建产物） — 其余 Phase 需要硬件 / App 在环

---

## Phase 0 — 代码冻结

| # | 项 | 结果 | 备注 |
|---|----|------|------|
| 0.1 | version.h patch 已 bump | ✅ PASS | 1.3.1 |
| 0.2 | 工作目录干净 | ⚠️ WARN | untracked 中 build/ 中间产物 + 本批 docs/，无 modified |
| 0.3 | `ota_keys.h` 存在且不在 git | ✅ PASS | 657B，git ls-files 为空 |
| 0.4 | SDK 完整 | ✅ PASS | libCH59xBLE.a / libISP592.a 都在 |
| 0.5 | release 编译通过 | ✅ PASS | 0 warning |
| 0.6 | App ↔ 固件 enroll 状态枚举对齐 | ✅ PASS | 0/1/2/3/4 两边一致 |

---

## Phase 1 — 构建产物

### 1.1 编译警告

✅ **0 warning**（非 SDK），release 模式干净通过。

### 1.2 .text 大小

| 项 | 实测 | 限额 | 占比 | 结果 |
|----|------|------|------|------|
| .text | 171,996 B | 221,184 B (216KB) | 78% | ✅ PASS（剩 ~48KB） |

### 1.3 RAM 总占用

| 段 | 大小 |
|----|------|
| .highcode | 9,648 B |
| .data | 1,756 B |
| .bss | 10,576 B |
| .stack_guard | 4,096 B |
| .stack | 512 B |
| **合计** | **26,588 B / 26,624 B (99.9%)** |

✅ PASS — 与 1.3.0 一致，无 RAM 增长（防 R599S latch + low-batt check 都在 .text）。

### 1.4 imfw SHA256（单一 SKU）

```
fw-1.3.1.c2f6-r.imfw
  29a0c41dde4152861765eebac2212332361fb71a3510a7d159b30c26368ed19c
```

✅ 单一 dist 文件，1.2.x 的 W/B/G 三色文件已清理。

### 1.5 OTA combined hex 烧录布局

- ota-1.3.1.c2f6-r.bin = 458,752 B（覆盖 0x00000–0x70000）
- 首记录 0x0000（JumpIAP 起点）
- ✅ PASS

### 1.6 DIS Model Number

| 字符串 | 出现 |
|-------|------|
| `IK-1` | ✅ 存在（1 个 Model Number + 2 个 Manufacturer 复合用） |
| `IK-1-W` / `IK-1-B` / `IK-1-G` | ❌ 不存在 |

✅ PASS — 1.3.1 起单一 SKU，无 variant 后缀。

### 1.7 私钥泄漏检查

| `BEGIN.*PRIVATE` 命中 | 0 |

✅ PASS

---

## 总结

| Phase | 状态 |
|-------|------|
| Phase 0 | 5 PASS / 1 WARN（untracked，非阻塞） |
| Phase 1 | 7 PASS / 0 待修 |

**Blocker：0**

### 1.3.1 vs 1.3.0 行为差异

| 修复 | 类别 | 来源 |
|------|------|------|
| LOCK_HOLD_EVT 在 gate/auth pending 时抑制 | bug fix | #6 |
| < 5% 电量拒绝 KEY_WRITE/COMMIT/DELETE/GENERATE/OTA，返回 `SEC_ERR_LOW_BATTERY` (0xF4) | defense | D |
| 取消三色 variant 编译期分支，DIS Model Number = "IK-1" 单一 SKU | refactor | #7 |

### App 1.11 vs 1.10 行为差异

| 修复 | 类别 | 来源 |
|------|------|------|
| AUTH_REQUEST 期间持住命令队列，避免 1 字节通知被 GET_STATUS 误解（与 #5 同形 race） | bug fix | A |

---

## 未自动化项（需硬件 / App / 现场）

执行用文档：**[qa-manual-checklist.md](qa-manual-checklist.md)** —— 带 checkbox、签字栏、异常记录区，含 FAQ 区（#2 cache 行为 / #4 低电量阈值）。

完整流程定义：[qa-production-checklist.md](qa-production-checklist.md)
