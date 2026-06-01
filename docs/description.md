# immurok — 无线指纹认证器

**一触即达，告别密码。**

immurok 是一款小巧的无线蓝牙指纹认证设备，专为 Mac 和 Linux 桌面用户设计。它让你用指纹替代密码，完成屏幕解锁、sudo 授权、SSH 签名等日常操作——只需轻触一下。

---

## 为什么需要 immurok？

**你每天输入多少次密码？**

如果你用的是 Mac mini、Mac Studio、Mac Pro，或者 MacBook 外接显示器合盖使用——你没有 Touch ID。苹果唯一的方案是花 ¥1,999 买一把带 Touch ID 的妙控键盘，而且你必须用它当主键盘。如果你习惯了机械键盘，这不是一个选项。

Linux 用户的处境更糟：几乎没有可用的桌面指纹方案。

immurok 解决的就是这个问题。

---

## 核心功能

### 屏幕解锁
锁屏状态下，触摸传感器即可自动解锁。无需打开盖子，无需敲键盘。支持 macOS 和 Linux。

### sudo 和系统授权
终端里 `sudo` 弹出密码提示？系统偏好设置要求认证？触摸指纹即可通过。通过自定义 PAM 模块深度集成系统认证体系，不是简单的密码填充。

### SSH 签名与 Git 提交签名
设备内置 ECDSA P-256 密钥对，私钥永远不会离开硬件。你可以用指纹签署 Git 提交和 SSH 登录请求——比 GPG key 更方便，比密码更安全。

### 超长续航
空闲电流 <30µA，USB-C 充电，待机可达约 3 个月。设备在你不用时自动休眠，触摸瞬间唤醒。

---

## 新增：为 AI 编码 agent 而生的人机授权门

Claude Code、Cursor、Codex、Aider、Cline、Gemini CLI——这些 AI agent 越来越多地在你的机器上**替你执行命令**：跑 `sudo`、`git push`、读 API key、调远程 SSH。但它们没有手指，按不了指纹。

immurok 提供了一种新的解法：让 agent 把命令"递给你过一下指纹"。

```bash
# agent 即将运行的特权命令、SSH 命令、读密钥的命令，全部包一层
imk run --agent -- sudo systemctl restart api
imk run --agent -- git push origin main
imk run --agent --env-file .env -- python deploy.py
```

屏幕上会弹出一个 HUD overlay，把命令原样展示给你。**你触摸一下设备**——子进程跑起来；**关闭弹窗或 30 秒不动**——子进程被 `SIGTERM` 干净杀掉，agent 收到退出码 77，知道这是用户拒绝。

### 为什么这是 AI agent 时代必要的环节

| 传统做法的痛 | immurok 的解法 |
|------|------|
| 给 agent 配 sudo NOPASSWD → 等于把 root 交出去 | 每条特权命令要一次指纹，授权可视、可拒绝 |
| 给 agent 装 ssh-agent + 长期密钥 → 写代码的 agent 同时拥有 prod 推送权 | SSH 签名走设备 ECDSA 私钥，每次签名要指纹 |
| 把 API key 直接放进 `.env` → agent 整个对话都能看到 | `OPENAI_KEY=imk://api/openai`，只在 `exec()` 那一刻注入子进程，agent 从来读不到明文 |
| agent 卡在 sudo 密码提示上无法继续 | 一次指纹同时通过 sudo + SSH + 密钥读取，5 分钟窗口内的整个子进程都不会被打断 |

### 配套的 imk-skill

我们维护了一个开源 [imk-skill](https://github.com/immurok/imk-skill) 仓库——一份单文件的 markdown skill，告诉 agent：什么场景该 wrap、什么时候不要 wrap、被拒绝了怎么干净退出。

- **Claude Code**：`/plugin marketplace add immurok/imk-skill && /plugin install imk-tools@imk`
- **Cursor / Cline / Continue / Windsurf**：vendor 进 `.cursorrules` / `.clinerules` / `.continue/rules.md`
- **Codex / Aider / Gemini CLI**：贴进 `AGENTS.md` / `CONVENTIONS.md` / `GEMINI.md`

装好之后，agent 在你的 immurok 项目里第一次想跑特权命令，就会自动用 `imk run --agent --` 包起来。你的指纹是它的签到表。

---

## 它凭什么做到别人做不到的事？

### 第一个真正的无线蓝牙指纹认证器

市面上的指纹方案要么是笔记本内置的，要么是 USB 有线的。immurok 是第一个通过蓝牙无线连接、同时支持 macOS 和 Linux 的指纹认证设备。你可以把它放在桌面任何位置，不受线缆束缚。

### 同时支持 Mac 和 Linux

不是"支持 Mac，Linux 在计划中"——是现在就支持。macOS 有原生 Swift 菜单栏应用，Linux 有 Rust 守护进程和 TUI 管理界面。两个平台都有完整的 PAM 集成。

### 独立设备，不绑定键盘

苹果的 Touch ID 方案要求你必须使用妙控键盘。immurok 是独立设备，你可以继续用你最喜欢的机械键盘、HHKB、或任何键盘。

### 不只是解锁——深度系统集成

大多数生物识别设备只能做屏幕解锁。immurok 通过 PAM 模块实现了系统级集成：sudo、屏幕锁定、系统授权对话框都能用指纹通过。SSH Agent 功能更是独一无二——用指纹签署代码提交和远程登录。

---

## 安全设计

| 原则 | 实现 |
|------|------|
| 生物特征不出设备 | 指纹模板存储在传感器内部闪存，主控芯片都无法读取 |
| 端到端加密通信 | ECDH P-256 密钥交换配对，所有 BLE 消息经 HMAC-SHA256 签名 |
| 无云端、无账号、无遥测 | 纯本地蓝牙通信，不需要互联网，不收集任何数据 |
| 固件更新也加密 | OTA 升级包经 AES-128-CTR 加密 + HMAC-SHA256 签名验证 |
| 开源可审计 | 应用和 PAM 模块完全开源，不信任我们？自己看代码 |

---

## 与竞品对比

| 特性 | immurok | 妙控键盘 (Touch ID) | USB 指纹器 |
|------|---------|---------------------|-----------|
| 无线连接 | ✅ 蓝牙 LE | ✅ 蓝牙 | ❌ USB 有线 |
| Mac 桌面支持 | ✅ | ✅ | ⚠️ 部分 |
| Linux 支持 | ✅ | ❌ | ⚠️ 有限 |
| sudo / PAM 集成 | ✅ | ❌ | ❌ |
| SSH 签名 | ✅ | ❌ | ❌ |
| **AI agent 命令授权** | ✅ `imk run --agent` | ❌ | ❌ |
| **指纹门控的密钥仓库（API key / OTP）** | ✅ `imk://...` URI | ❌ | ❌ |
| 独立使用（不绑键盘） | ✅ | ❌ 必须当主键盘 | ✅ |
| 开源 | ✅ | ❌ | ❌ |
| 价格 | 远低于 ¥1,999 | ¥1,999 | ¥100-300 |

---

## 硬件规格

- **连接方式**：蓝牙 LE 5.4
- **指纹传感器**：电容式，508 DPI，识别时间 <0.5 秒
- **误识率 / 拒识率**：<0.001% / <1%
- **可存储指纹**：最多 10 枚
- **充电接口**：USB-C
- **待机续航**：约 3 个月
- **尺寸**：52 × 32 × 12 mm

---

## 适合谁？

- **重度使用 AI 编码 agent 的开发者**——希望 agent 能自主执行命令但不想交出 sudo / SSH / API key 的人
- 使用 Mac mini / Mac Studio / Mac Pro 的开发者和专业用户
- MacBook 合盖外接显示器的用户
- 需要桌面指纹认证的 Linux 用户
- 重视隐私、不想依赖云服务的安全意识用户
- 用机械键盘但又想要指纹解锁的人

---

*2026 年发布。加入等候名单，获取早鸟资格。*
