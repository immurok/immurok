# 12 步分角度指纹登记引导 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 macOS App 的指纹登记 sheet 里加入 12 步固定顺序的角度引导（4 正 / 3 左 / 3 右 / 1 上 / 1 下），用动画椭圆 + 方向箭头提示每一步的目标角度。

**Architecture:** 纯 App 侧 SwiftUI 改动。`EnrollmentSheet` 中央动画区域改造为带偏移椭圆 + 方向箭头的 ZStack，监听 `viewModel.enrollmentProgress` 自动切换。新增 4 个 fileprivate pure helper 函数把 progress 映射到 offset/arrow/title。新增 8 个 i18n key 覆盖中（简繁）+ 英。

**Tech Stack:** SwiftUI, Combine, `LocalizationManager`（项目自有简单 dict-based i18n）

**Spec:** [`docs/superpowers/specs/2026-05-25-enroll-guided-angles-design.md`](../specs/2026-05-25-enroll-guided-angles-design.md)

**No unit tests:** App 端没有 testTarget。验证通过手动 UI 测试（步骤详见 Task 5）。

---

## File Structure

**Modify:**
- `app-macos/Sources/FingerprintView.swift` — `EnrollmentSheet` body 替换，加 4 个 fileprivate helper（约 +90 行 / -20 行）
- `app-macos/Sources/LocalizationManager.swift` — 在 zhHansStrings / zhHantStrings / enStrings 三个字典里各加 8 个 key（共 +24 行）

**No new files.**

---

## Task 1: 新增 i18n 文案 key

**Files:**
- Modify: `app-macos/Sources/LocalizationManager.swift`（3 处字典：zh-Hans 约 305 附近、zh-Hant 约 661 附近、en 约 1017 附近）

8 个新 key，每个字典插在 `"enroll.captured"` 那一行后面（保持 enroll 相关 key 聚类）。

- [ ] **Step 1: 在 zhHansStrings 字典里加 8 个简体中文 key**

定位锚点：`"enroll.captured": "已捕获 (%d/%d)",`（在 `static let zhHansStrings: [String: String] = [` 字典里，约 308 行）

在该行之后插入：

```swift
        "enroll.step.center.first": "指肚正中按压",
        "enroll.step.center.keep": "保持正中按压",
        "enroll.step.left.first": "稍向左偏 5–10°",
        "enroll.step.left.keep": "保持左偏",
        "enroll.step.right.first": "稍向右偏 5–10°",
        "enroll.step.right.keep": "保持右偏",
        "enroll.step.up": "稍向指尖方向偏",
        "enroll.step.down": "稍向手腕方向偏",
```

- [ ] **Step 2: 在 zhHantStrings 字典里加 8 个繁体中文 key**

定位锚点：`"enroll.captured": "已擷取 (%d/%d)",`（约 664 行）

在该行之后插入：

```swift
        "enroll.step.center.first": "指腹正中按壓",
        "enroll.step.center.keep": "保持正中按壓",
        "enroll.step.left.first": "稍向左偏 5–10°",
        "enroll.step.left.keep": "保持左偏",
        "enroll.step.right.first": "稍向右偏 5–10°",
        "enroll.step.right.keep": "保持右偏",
        "enroll.step.up": "稍向指尖方向偏",
        "enroll.step.down": "稍向手腕方向偏",
```

- [ ] **Step 3: 在 enStrings 字典里加 8 个英文 key**

定位锚点：`"enroll.captured": "Captured (%d/%d)",`（约 1020 行）

在该行之后插入：

```swift
        "enroll.step.center.first": "Press finger pad on center",
        "enroll.step.center.keep": "Keep pressing center",
        "enroll.step.left.first": "Tilt slightly left 5–10°",
        "enroll.step.left.keep": "Keep tilted left",
        "enroll.step.right.first": "Tilt slightly right 5–10°",
        "enroll.step.right.keep": "Keep tilted right",
        "enroll.step.up": "Shift slightly toward fingertip",
        "enroll.step.down": "Shift slightly toward wrist",
```

- [ ] **Step 4: 编译验证**

Run: `cd app-macos && swift build 2>&1 | tail -20`
Expected: build succeeds (字符串改动不会引入编译错误).

- [ ] **Step 5: Commit**

```bash
git add app-macos/Sources/LocalizationManager.swift
git commit -m "$(cat <<'EOF'
feat(app): enroll 引导新增 8 个文案 key (中英三语)

为 12 步分角度登记引导准备 i18n key，覆盖 zh-Hans/zh-Hant/en。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: 新增 4 个 fileprivate helper 函数

**Files:**
- Modify: `app-macos/Sources/FingerprintView.swift`（在 `// MARK: - Enrollment Sheet` 之前加 fileprivate helper 区段）

- [ ] **Step 1: 在 `// MARK: - Enrollment Sheet` 前插入 helper 区段**

定位锚点：`// MARK: - Enrollment Sheet`（FingerprintView.swift:257）

在该行**之前**插入：

```swift
// MARK: - Enrollment Step Helpers

/// 12-step guided enrollment offset map (椭圆偏移量，px).
/// progress 1..4: 正中 / 5..7: 左偏 / 8..10: 右偏 / 11: 上偏 / 12: 下偏
/// Out-of-range progress (0 pre-start, >12 post-complete) returns .zero.
fileprivate func enrollOffsetForStep(_ progress: Int) -> CGSize {
    switch progress {
    case 1...4:  return .zero
    case 5...7:  return CGSize(width: -16, height: 0)
    case 8...10: return CGSize(width: 16, height: 0)
    case 11:     return CGSize(width: 0, height: -16)
    case 12:     return CGSize(width: 0, height: 16)
    default:     return .zero
    }
}

/// SF Symbol name for direction arrow at given step.
fileprivate func enrollArrowSymbolForStep(_ progress: Int) -> String {
    switch progress {
    case 1...4:  return "arrow.up"
    case 5...7:  return "arrow.left"
    case 8...10: return "arrow.right"
    case 11:     return "arrow.up"
    case 12:     return "arrow.down"
    default:     return "arrow.up"
    }
}

/// Arrow position offset relative to the 60×80 ellipse (椭圆外侧).
/// Direction-aware: arrow sits outside the ellipse in the same direction
/// the user should tilt toward.
fileprivate func enrollArrowOffsetForStep(_ progress: Int) -> CGSize {
    switch progress {
    case 1...4:  return CGSize(width: 0, height: -55)   // 顶部居中
    case 5...7:  return CGSize(width: -55, height: 0)   // 左侧
    case 8...10: return CGSize(width: 55, height: 0)    // 右侧
    case 11:     return CGSize(width: 0, height: -55)   // 顶部
    case 12:     return CGSize(width: 0, height: 55)    // 底部
    default:     return CGSize(width: 0, height: -55)
    }
}

/// Localization key for the step's primary title text.
fileprivate func enrollTitleKeyForStep(_ progress: Int) -> String {
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

/// True for the 4 steps where the angle category changes (用于颜色闪烁触发).
/// Step 5: 正中→左, Step 8: 左→右, Step 11: 右→上, Step 12: 上→下.
fileprivate func enrollIsTransitionStep(_ progress: Int) -> Bool {
    progress == 5 || progress == 8 || progress == 11 || progress == 12
}

```

- [ ] **Step 2: 编译验证**

Run: `cd app-macos && swift build 2>&1 | tail -20`
Expected: build succeeds. Helper 没被引用是正常的（Task 3 才用），Swift 不会因 fileprivate 未使用而 warn-as-error.

- [ ] **Step 3: Commit**

```bash
git add app-macos/Sources/FingerprintView.swift
git commit -m "$(cat <<'EOF'
feat(app): 加 4 个 enrollment step helper 函数

把 progress (1..12) 映射到椭圆偏移、箭头方向、文案 key、过渡判定。
纯函数，Task 3 接入 EnrollmentSheet view 时使用。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: 改造 EnrollmentSheet 中央动画 + 标题

**Files:**
- Modify: `app-macos/Sources/FingerprintView.swift`（`EnrollmentSheet` struct，约 259-316 行）

整段 body 替换。把现有静态 `touchid` 图标 + `enroll.hint.center` 文案替换为：
- 偏移椭圆（位置由 progress 决定）
- 方向箭头（位置 + 方向由 progress 决定）
- 步骤标题（key 由 progress 决定，切换步加粗）

颜色闪烁逻辑（Task 4 加），本任务先用 `accentColor` 不闪。

- [ ] **Step 1: 替换 EnrollmentSheet body 与 onAppear**

定位锚点：整个 `struct EnrollmentSheet: View { ... }`，从 `struct EnrollmentSheet: View {` 到对应的闭合 `}`（约 259-316 行）。

把这一整段替换为：

```swift
struct EnrollmentSheet: View {
    @ObservedObject var viewModel: FingerprintViewModel
    @Environment(\.dismiss) private var dismiss
    @State private var pulseScale: CGFloat = 1.0
    @State private var arrowOpacity: Double = 1.0

    /// Clamped progress used for the animation (避免 0 / >12 时的极端 offset).
    private var animProgress: Int {
        min(max(viewModel.enrollmentProgress, 1), 12)
    }

    private var isTransitionStep: Bool {
        enrollIsTransitionStep(viewModel.enrollmentProgress)
    }

    var body: some View {
        VStack(spacing: 20) {
            // sensor 圆 + 偏移椭圆 + 方向箭头
            ZStack {
                // 背景圆（不变）
                Circle()
                    .fill(Color(NSColor.controlBackgroundColor))
                    .frame(width: 120, height: 120)

                // pulse 圆（不变，等待手指视觉提示）
                Circle()
                    .stroke(Color.accentColor.opacity(0.3), lineWidth: 2)
                    .frame(width: 60, height: 60)
                    .scaleEffect(pulseScale)
                    .opacity(2.0 - Double(pulseScale))

                // 手指椭圆（按 progress 偏移）
                RoundedRectangle(cornerRadius: 30)
                    .fill(Color.accentColor)
                    .frame(width: 60, height: 80)
                    .offset(enrollOffsetForStep(animProgress))
                    .opacity(viewModel.enrollmentProgress > 0 ? 1.0 : 0.5)
                    .animation(.easeInOut(duration: 0.3), value: animProgress)

                // 方向箭头（位置 + symbol 都按 progress 切换）
                Image(systemName: enrollArrowSymbolForStep(animProgress))
                    .font(.system(size: 22, weight: .bold))
                    .foregroundColor(.accentColor)
                    .offset(enrollArrowOffsetForStep(animProgress))
                    .opacity(arrowOpacity)
                    .animation(.easeInOut(duration: 0.3), value: animProgress)
            }

            // 主步骤标题（切换步加粗）
            Text(enrollTitleKeyForStep(animProgress).localized)
                .font(.headline)
                .fontWeight(isTransitionStep ? .semibold : .regular)
                .multilineTextAlignment(.center)

            // 现有 status text（waiting / captured / lift_finger 等动态文案）
            Text(viewModel.enrollmentStatus)
                .font(.caption)
                .foregroundColor(.secondary)
                .multilineTextAlignment(.center)

            ProgressView(value: Double(viewModel.enrollmentProgress), total: Double(viewModel.enrollmentTotal))
                .frame(width: 200)

            Text("\(viewModel.enrollmentProgress)/\(viewModel.enrollmentTotal)")
                .font(.caption)
                .foregroundColor(.secondary)

            Button("alert.cancel".localized) {
                viewModel.cancelEnrollment()
                dismiss()
            }
            .keyboardShortcut(.cancelAction)
        }
        .padding(32)
        .frame(width: 300, height: 380)
        .onAppear {
            withAnimation(.easeInOut(duration: 1.5).repeatForever(autoreverses: true)) {
                pulseScale = 1.6
            }
            // 箭头脉动节奏（0.8s 周期）
            withAnimation(.easeInOut(duration: 0.8).repeatForever(autoreverses: true)) {
                arrowOpacity = 0.4
            }
        }
    }
}
```

注意：
- 原本的 `Text("enroll.hint.center".localized)` 被两层文字替代：动态主标题 + 原 status text 降级为副文案
- 椭圆 `.animation(.easeInOut(duration: 0.3), value: animProgress)` 让 `.offset` 修改自动 0.3s 过渡 ✓
- 箭头 symbol 切换会因为 view identity 变化重新淡入，配合 `.animation` 自然过渡
- 颜色闪烁逻辑 Task 4 加

- [ ] **Step 2: 编译验证**

Run: `cd app-macos && swift build 2>&1 | tail -30`
Expected: build succeeds. 如有 warning 关于未用变量 `isTransitionStep` 中的某些 case，忽略——Task 4 会用上。

- [ ] **Step 3: 启动 App 手测进入登记界面**

Run: `cd app-macos && ./build-deploy.sh -a -s 2>&1 | tail -10`（编译 + 部署 + 替换 /Applications/immurok.app）

然后：
- 打开 `/Applications/immurok.app`
- 状态栏点击 → 打开设置 → 指纹管理
- 点击空 slot 进入登记界面

预期看到（设备没真按指纹的情况下，progress=0 → 1 后）：
- 椭圆居中、向上箭头脉冲
- 标题文字"指肚正中按压"（如系统中文）/ "Press finger pad on center"（如系统英文）
- 进度条 0/12 或 1/12

确认这些 OK 后再继续。如果布局有问题（比如箭头跑出 sensor 圆外），调 `enrollArrowOffsetForStep` 的常数。

- [ ] **Step 4: Commit**

```bash
git add app-macos/Sources/FingerprintView.swift
git commit -m "$(cat <<'EOF'
feat(app): EnrollmentSheet 椭圆 + 方向箭头 + 步骤标题

整段 body 替换：sensor 圆 ZStack 内加入按 progress 偏移的椭圆和
方向箭头，主标题随步骤动态切换；原 status text 降级为副文案。
切换过渡颜色闪烁 Task 4 接入。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: 接入过渡颜色闪烁

**Files:**
- Modify: `app-macos/Sources/FingerprintView.swift`（`EnrollmentSheet`）

在切换步（progress 5/8/11/12）触发 0.15s 渐变到 orange 再 0.15s 回 accentColor。

- [ ] **Step 1: 加 transition flash state 与 onChange**

定位锚点：在 `EnrollmentSheet` struct 内，找到这一行：

```swift
    @State private var arrowOpacity: Double = 1.0
```

在它**下面**新增一行：

```swift
    @State private var transitionFlash: Bool = false
```

- [ ] **Step 2: 把椭圆 fill 改成动态颜色**

定位锚点：

```swift
                RoundedRectangle(cornerRadius: 30)
                    .fill(Color.accentColor)
                    .frame(width: 60, height: 80)
```

替换 `.fill(Color.accentColor)` 这一行为：

```swift
                    .fill(transitionFlash ? Color.orange : Color.accentColor)
```

完整片段应该是：

```swift
                RoundedRectangle(cornerRadius: 30)
                    .fill(transitionFlash ? Color.orange : Color.accentColor)
                    .frame(width: 60, height: 80)
                    .offset(enrollOffsetForStep(animProgress))
                    .opacity(viewModel.enrollmentProgress > 0 ? 1.0 : 0.5)
                    .animation(.easeInOut(duration: 0.3), value: animProgress)
```

- [ ] **Step 3: 在 `.frame(width: 300, height: 380)` 之后加 onChange**

定位锚点：

```swift
        .padding(32)
        .frame(width: 300, height: 380)
        .onAppear {
```

在 `.frame(width: 300, height: 380)` 与 `.onAppear` 之间插入：

```swift
        .onChange(of: viewModel.enrollmentProgress) { newProgress in
            if enrollIsTransitionStep(newProgress) {
                withAnimation(.easeInOut(duration: 0.15)) {
                    transitionFlash = true
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.15) {
                    withAnimation(.easeInOut(duration: 0.15)) {
                        transitionFlash = false
                    }
                }
            }
        }
```

完整连贯段应该是：

```swift
        .padding(32)
        .frame(width: 300, height: 380)
        .onChange(of: viewModel.enrollmentProgress) { newProgress in
            if enrollIsTransitionStep(newProgress) {
                withAnimation(.easeInOut(duration: 0.15)) {
                    transitionFlash = true
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.15) {
                    withAnimation(.easeInOut(duration: 0.15)) {
                        transitionFlash = false
                    }
                }
            }
        }
        .onAppear {
            withAnimation(.easeInOut(duration: 1.5).repeatForever(autoreverses: true)) {
                pulseScale = 1.6
            }
            withAnimation(.easeInOut(duration: 0.8).repeatForever(autoreverses: true)) {
                arrowOpacity = 0.4
            }
        }
```

- [ ] **Step 4: 编译验证**

Run: `cd app-macos && swift build 2>&1 | tail -20`
Expected: build succeeds.

注意：`onChange(of:perform:)` 在 macOS 14+ 是 deprecated，新签名是 `onChange(of:initial:_:)`。如果 deployment target ≥ macOS 14，编译会有 deprecation warning——可以忽略，或改用新签名：

```swift
.onChange(of: viewModel.enrollmentProgress) { _, newProgress in
    ...
}
```

按当前 project 的 deployment target 选用（看 `Package.swift` 里的 `platforms`）。

- [ ] **Step 5: Commit**

```bash
git add app-macos/Sources/FingerprintView.swift
git commit -m "$(cat <<'EOF'
feat(app): enroll 步骤切换时椭圆颜色闪烁 (accent → orange → accent)

progress 落在切换步 (5/8/11/12) 时触发 0.3s 闪烁，提示用户注意
角度变化。配合椭圆 0.3s 滑动过渡形成完整视觉反馈。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: 端到端手动测试 + 模板质量回归验证

**Files:** 不改代码，纯测试 + 文档化结果

本任务验证设计目的（让登记后的指纹匹配分数收紧）是否达成。

- [ ] **Step 1: 部署最新 App**

Run: `cd app-macos && ./build-deploy.sh -a -s 2>&1 | tail -10`

- [ ] **Step 2: 删除现有指纹 slot 0**

打开 immurok.app → 设置 → 指纹管理 → slot 0 → 删除（需要按指纹确认）。

预期：slot 0 状态变为"空"。

- [ ] **Step 3: 用新引导式登记重新登记 slot 0**

点击 slot 0 → 进入登记。按提示完成 12 次按压：

1. 进入时椭圆居中、向上箭头脉冲、标题"指肚正中按压"
2. 第 1-4 次：指肚正中、稳定压力按压
3. 第 5 次按下后抬指——观察椭圆**0.3s 滑动到左偏 + 颜色闪一下橙色**，标题加粗"稍向左偏 5–10°"
4. 第 5-7 次：手指稍向左转 5-10°
5. 第 8 次开始：椭圆滑到右偏、箭头转 →
6. 第 8-10 次：手指稍向右转
7. 第 11 次：椭圆滑到上偏（指尖方向），箭头 ↑
8. 第 12 次：椭圆滑到下偏（手腕方向），箭头 ↓
9. 完成 → sheet 关闭，slot 0 状态变为"已登记"

每步骤切换观察清单：
- [ ] 椭圆位置变化丝滑（0.3s easeInOut，非突变）
- [ ] 颜色闪到橙色再回 accent
- [ ] 箭头方向正确（左偏时 ←、右偏时 →、上 ↑、下 ↓）
- [ ] 标题文字按步骤切换、切换步加粗

- [ ] **Step 4: 跑分数分布回归测试**

设备当前跑的是 1.4.7 含 PRINT 的诊断固件（2026-05-25 烧的）。接 UART3 调试线，picocom 监听 115200。

按 slot 0 同手指 10–20 次，记录每次的 `SEARCH ack OK (Xms): slot=0 score=Y` 数据。

对比 2026-05-25 旧模板的 baseline：

| 指标 | 旧（无引导） | 新（引导）期望 |
|---|---|---|
| score 中位数 | ~200 | ≥ 220 |
| score 最小值 | 92 | > 150 |
| score 跨度 | 184 | < 100 |
| 通过率（floor=150） | 62.5% (5/8) | > 90% |

如果新模板的 worst-case 分数 > 150 → 设计目的达成，floor 不需要改。

如果仍有 < 150 的真匹配 → 需要后续讨论是否调 floor（不在本计划范围内）。

- [ ] **Step 5: 记录结果到 memory + commit 验证报告**

如果测试结果符合预期，在 memory 加一条 project 记录（仿 [[project_fp_false_accept_bug]] 的格式）：

写文件 `~/.claude/projects/-Users-katsu-Documents-Projects-imPress-v1/memory/project_enroll_guided_angles.md`：

```markdown
---
name: project-enroll-guided-angles
description: 12 步分角度登记引导（2026-05-25 上线），把同手指匹配分数从跨度 184 收紧到 <100
metadata:
  type: project
---

App 端登记 sheet 改造为 4正/3左/3右/1上/1下 的固定 12 步引导，
椭圆滑动 + 颜色闪烁提示用户每一步的目标角度。

**Why:** 2026-05-25 实测 slot=0 旧模板 score 分布 92-276 跨度 184，
30%+ 拒识率，根因是 12 次采集差异过大导致模板含噪声。引导式登记
覆盖日常按压真实角度变化范围，每帧本身质量可控。

**How to apply:** 用户报告指纹识别率波动时，先建议重登（确保跑过
新引导）；如新引导后仍有 < 150 分数，再考虑降 FP_MIN_MATCH_SCORE。

**实测改善**：[填入 Step 4 实际数据]

相关：[[project_fp_false_accept_bug]]、[[ble-power]]
```

同步在 `MEMORY.md` 添加索引（在"安全漏洞历史"或"待优化项"之后合适分类下）。

然后 commit memory + design/plan docs：

```bash
git add docs/superpowers/specs/2026-05-25-enroll-guided-angles-design.md \
        docs/superpowers/plans/2026-05-25-enroll-guided-angles.md
git commit -m "$(cat <<'EOF'
docs: 12 步分角度登记引导 design + plan + 实测验证

设计目的：解决同手指匹配分数跨度过大（92-276）导致的识别率波动。
实测结果详见 plan Step 4 表格。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**Spec 覆盖检查**：
- ✅ 12 步映射 → Task 2 (helper 函数)
- ✅ 动画规范（椭圆滑动 0.3s、颜色闪 0.15+0.15s、箭头切换） → Task 3 + Task 4
- ✅ 文案中英双语 → Task 1
- ✅ Sheet 布局不变 → Task 3 保留 300×380
- ✅ 不动 ViewModel → 所有 task 都只改 View
- ✅ 不动固件 → 无 firmware/ 改动
- ✅ 假设说明（progress 单调递增、enrollmentTotal=12）→ Task 3 用 `min(max(progress, 1), 12)` clamp
- ✅ 模板质量回归验证 → Task 5 Step 4 用 1.4.7 诊断 PRINT 数据对比

**Placeholder 扫描**：无 TBD/TODO/"add appropriate"，所有代码块完整。

**类型一致性**：
- `enrollOffsetForStep` / `enrollArrowSymbolForStep` / `enrollArrowOffsetForStep` / `enrollTitleKeyForStep` / `enrollIsTransitionStep` 命名与 Task 3/4 引用一致 ✓
- `transitionFlash: Bool`、`arrowOpacity: Double`、`pulseScale: CGFloat` 类型一致 ✓
- i18n key 8 个全部在 helper 的 switch 里能映射到 ✓

**Scope**：单一 PR / sub-system，无需拆分。
