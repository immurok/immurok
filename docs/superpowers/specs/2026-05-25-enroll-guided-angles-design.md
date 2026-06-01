# 12 步分角度指纹登记引导

**Date:** 2026-05-25
**Status:** Design approved, ready for planning
**Scope:** macOS App UI only — no firmware/protocol changes

## 背景

R559S 指纹登记一次需要 12 帧采集（`FP_ENROLL_CAPTURES = 12`）。当前 App 端 `EnrollmentSheet` 只提示用户"指肚正中按压"，12 次都用同一个动画。

实测发现 slot=0 的同手指匹配分数跨度高达 92–276（2026-05-25 测试，详见对话上下文），强烈暗示模板含噪声——12 次采集的角度/压力差异过大或过小都会让模板质量下降。

为让模板覆盖日常按压的真实角度变化范围、同时每帧本身质量稳定，引入按固定顺序提示用户调整角度的引导流程：

- 4 次正中（baseline）
- 3 次稍向左偏 5–10°
- 3 次稍向右偏 5–10°
- 1 次稍向指尖偏移
- 1 次稍向手腕偏移

## 目标

1. 按上述固定顺序为 12 次采集提供清晰的角度提示
2. 角度切换瞬间用户能察觉到变化
3. 不增加用户认知负担——动画自解释，不依赖文字阅读
4. 不动固件、不改 BLE 协议、不改 ViewModel 公开接口

## 非目标

- ❌ 检测用户实际按压位置（需 R559S 暴露 raw image，超出范围）
- ❌ 智能调整角度顺序
- ❌ 失败重采的特殊处理（progress 怎么变 UI 就跟着变）
- ❌ 自定义角度顺序

## 设计

### 步骤 → 角度映射

| Progress | 角度 | 椭圆偏移 | 箭头方向 | 文案 key |
|---|---|---|---|---|
| 1 | 正中 | (0, 0) | 静态脉冲 ↑ | `enroll.step.center.first` |
| 2–4 | 保持正中 | (0, 0) | 静态脉冲 ↑ | `enroll.step.center.keep` |
| 5 | 偏左 5–10° | (-16, 0) | ← | `enroll.step.left.first` |
| 6–7 | 保持左偏 | (-16, 0) | ← | `enroll.step.left.keep` |
| 8 | 偏右 5–10° | (+16, 0) | → | `enroll.step.right.first` |
| 9–10 | 保持右偏 | (+16, 0) | → | `enroll.step.right.keep` |
| 11 | 稍向指尖偏 | (0, -16) | ↑ | `enroll.step.up` |
| 12 | 稍向手腕偏 | (0, +16) | ↓ | `enroll.step.down` |

Y 轴：椭圆顶部对应"指尖方向"（即视觉上偏上 = 指尖偏向 sensor）。

### 动画规范

**正常状态**（waiting / captured 期间）：
- 椭圆位于当前 progress 对应位置
- 方向箭头 0.8s 节奏 opacity 脉冲（0.4 ↔ 1.0）
- 颜色：`accentColor`

**过渡**（progress 4→5、7→8、10→11、11→12 时，即 lift_finger / processing 状态期间）：
- 椭圆 0.3s `easeInOut` 从上一步位置滑到下一步位置
- 颜色：0.15s 渐变到 `orange`，再 0.15s 渐变回 `accentColor`
- 箭头方向：旧箭头 0.15s 淡出 → 新箭头 0.15s 淡入

**首次 progress=1**（sheet 刚打开）：直接正中显示，无过渡。

### Sheet 布局

保持现有 300×380 sheet 尺寸不变。垂直堆叠顺序不变：

```
┌──────────────────────┐
│  sensor 圆 (120×120) │  ← 替换中央内容
│    [背景圆 + pulse]   │
│    [手指椭圆]         │
│    [方向箭头]         │
│                      │
│  主标题（步骤说明）   │  ← 替换 enroll.hint.center
│                      │
│  status text         │  ← 不变
│                      │
│  ProgressView        │  ← 不变
│  N/12                │  ← 不变
│                      │
│  [取消]              │  ← 不变
└──────────────────────┘
```

### sensor 圆 ZStack 结构

- **背景圆** 120×120 `controlBackgroundColor`（保留）
- **手指椭圆** 60×80 圆角矩形（cornerRadius=30），用 `offsetForStep(progress)` 控制位置
- **方向箭头** SF Symbol（`arrow.up` / `arrow.down` / `arrow.left` / `arrow.right`），叠加在椭圆相对外侧（如左偏时箭头出现在椭圆左侧外缘）
- 现有的 pulse 圆 stroke 保留作为"等待手指"的视觉提示

### 文案（中英双语）

| Key | zh | en |
|---|---|---|
| `enroll.step.center.first` | 指肚正中按压 | Press finger pad on center |
| `enroll.step.center.keep` | 保持正中按压 | Keep pressing center |
| `enroll.step.left.first` | **稍向左偏 5–10°** | **Tilt slightly left 5–10°** |
| `enroll.step.left.keep` | 保持左偏 | Keep tilted left |
| `enroll.step.right.first` | **稍向右偏 5–10°** | **Tilt slightly right 5–10°** |
| `enroll.step.right.keep` | 保持右偏 | Keep tilted right |
| `enroll.step.up` | **稍向指尖方向偏** | **Shift slightly toward fingertip** |
| `enroll.step.down` | **稍向手腕方向偏** | **Shift slightly toward wrist** |

切换步（5/8/11/12）的文案用 `.fontWeight(.semibold)`，其余常规字重。

## 代码改动

### 文件清单

- `app-macos/Sources/FingerprintView.swift` — 改 `EnrollmentSheet` body
- `app-macos/Sources/LocalizationManager.swift` 或对应 `.strings` — 新增 8 个文案 key

### 新增 helper 函数

放在 `EnrollmentSheet` 内或作为文件级 fileprivate：

```swift
fileprivate func offsetForStep(_ progress: Int) -> CGSize {
    switch progress {
    case 1...4:  return .zero
    case 5...7:  return CGSize(width: -16, height: 0)
    case 8...10: return CGSize(width: 16, height: 0)
    case 11:     return CGSize(width: 0, height: -16)
    case 12:     return CGSize(width: 0, height: 16)
    default:     return .zero  // pre-start / post-complete
    }
}

fileprivate func arrowSymbolForStep(_ progress: Int) -> String {
    switch progress {
    case 1...4:  return "arrow.up"      // 静态参考方向
    case 5...7:  return "arrow.left"
    case 8...10: return "arrow.right"
    case 11:     return "arrow.up"
    case 12:     return "arrow.down"
    default:     return "arrow.up"
    }
}

fileprivate func titleKeyForStep(_ progress: Int) -> String {
    switch progress {
    case 1:      return "enroll.step.center.first"
    case 2...4:  return "enroll.step.center.keep"
    case 5:      return "enroll.step.left.first"
    case 6...7:  return "enroll.step.left.keep"
    case 8:      return "enroll.step.right.first"
    case 9...10: return "enroll.step.right.keep"
    case 11:     return "enroll.step.up"
    case 12:     return "enroll.step.down"
    default:     return "enroll.step.center.first"
    }
}

fileprivate func isTransitionStep(_ progress: Int) -> Bool {
    progress == 5 || progress == 8 || progress == 11 || progress == 12
}
```

### EnrollmentSheet body 关键变更

用 `withAnimation` 包裹 `viewModel.enrollmentProgress` 的变化（已经是 `@Published`，会触发 view rebuild）。SwiftUI 会自动对 `.offset` 修改产生 `easeInOut` 过渡。

颜色切换：用一个 `@State private var transitionFlash: Bool` + `.onChange(of: viewModel.enrollmentProgress)`，在切换步触发 `transitionFlash = true` 然后 0.3s 后归位。

伪代码：
```swift
.onChange(of: viewModel.enrollmentProgress) { newValue in
    if isTransitionStep(newValue) {
        withAnimation(.easeInOut(duration: 0.15)) { transitionFlash = true }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.15) {
            withAnimation(.easeInOut(duration: 0.15)) { transitionFlash = false }
        }
    }
}
```

椭圆颜色：`transitionFlash ? .orange : .accentColor`

### ViewModel 不变

`FingerprintViewModel.enrollmentProgress` / `enrollmentTotal` / `enrollmentStatus` 已经是 `@Published`，View 自动响应。不需要新增任何 published 属性。

## 假设与依赖

1. **固件 progress 单调递增**：当前认为 progress 从 1 → 12 严格递增，失败重采时 progress 保持不变（重发同 captured 值）。若实测发现回退，UI 直接跟着回退即可（offset 逆向滑动）。
2. **`enrollmentTotal == 12`**：FingerprintViewModel 默认 12，由固件 ENROLL_START 响应回填。本设计严格按 12 步映射。若未来固件改为非 12，映射表需同步调整。
3. **i18n 已就绪**：依赖现有 `LocalizationManager.localized` 机制。

## 测试与验证

### 单元/UI 测试

- 不引入新的 ViewModel 状态，无新的单元测试需求
- 手动 UI 测试覆盖：
  1. 进入登记 → progress 1–4 椭圆居中
  2. progress 5 触发滑动到左偏 + orange 闪烁
  3. progress 8 触发滑动到右偏
  4. progress 11/12 触发上/下偏
  5. progress 12 完成后正常关闭 sheet
  6. 中途 Cancel 正常工作
  7. 中英文切换文案正确

### 验证模板质量（设计目的的回归检查）

完成本特性后，让用户用引导式登记重做 slot=0，再跑一次 10 次按压测试（用已部署的 SEARCH score PRINT），对比：

- 旧（无引导）模板分数：92–276，跨度 184
- 新（引导式）模板分数：期望集中在 180–280，最小值 > 150

若分布显著收紧 → 设计目的达成。

## 风险

- **风险 1**：用户不按引导走（如全程不偏）。**缓解**：动画明显，文案粗体强调；即便用户不偏，新模板也不会比旧模板差。
- **风险 2**：偏移幅度不够明显（用户看到椭圆移动但没意识到自己要变姿势）。**缓解**：颜色闪烁强化注意；文案明确指示方向。
- **风险 3**：320px 宽度装不下中文加粗 + 箭头 + 椭圆。**缓解**：sheet 宽度 300px 已经容下 status text；新文案最长 8 字（"稍向手腕方向偏"），不需要扩宽。

## 不在范围内（后续可考虑）

- 完成 12 步后显示一个简短的"模板质量评分"（基于固件返回的 ENROLL_COMPLETE 数据）
- 引导式重采机制（如 sensor 内部判定第 N 帧质量差，UI 回退到上一步重采）
- 可选高级模式：跳过引导（给老用户）

---

**Status:** Design approved by user 2026-05-25. Next step: invoke `writing-plans` skill to produce implementation plan.
