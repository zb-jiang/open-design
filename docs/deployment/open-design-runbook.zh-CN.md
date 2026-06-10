# Open Design 源码开发与调试运行手册

本文档面向“基于源码拓展和调试自定义功能”的场景，目标是让你在 Windows 11 上按步骤启动本地开发环境，并掌握常用调试、构建、日志和配置方法。

## 1. 项目概览

Open Design 是一个 pnpm workspace 单仓项目，核心模块如下：

| 模块           | 目录              | 作用                                                                       |
| ------------ | --------------- | ------------------------------------------------------------------------ |
| Web 前端       | `apps/web`      | Next.js 16 + React 18 前端界面，主要入口在 `apps/web/app` 和 `apps/web/src/App.tsx` |
| Daemon 后端    | `apps/daemon`   | Express + SQLite，本地 API、SSE、Agent 调度、项目/产物存储                             |
| Desktop 壳    | `apps/desktop`  | Electron 桌面壳                                                             |
| Packaged 运行时 | `apps/packaged` | 打包后的 Electron 运行入口                                                       |
| 开发启动器        | `tools/dev`     | 统一管理 daemon、web、desktop 的本地开发生命周期                                        |
| 共享包          | `packages/*`    | contracts、components、platform、sidecar 等工作区共享包                            |

***

## 2. 开发环境要求

### 2.1 必备工具

| 工具                        | 版本要求    | 验证命令               | 说明                                 |
| ------------------------- | ------- | ------------------ | ---------------------------------- |
| Node.js                   | 24.x    | `node -v`          | 项目 `package.json` 要求 `~24`         |
| pnpm                      | 10.33.2 | `pnpm -v`          | 项目固定 `pnpm@10.33.2`                |
| Git                       | 较新版本即可  | `git --version`    | 用于拉取和管理源码                          |
| Python                    | 3.x     | `python --version` | native Node 模块编译可能需要               |
| Visual Studio Build Tools | 2022    | 无固定命令              | `better-sqlite3` 等 native 依赖编译可能需要 |

项目根目录的版本约束：

```text
node: ~24
pnpm: >=10.33.2 <11
```

### 2.2 Windows 推荐环境

你当前是 Windows 11，建议使用 PowerShell 执行命令。

如果 PowerShell 脚本执行被限制，先执行：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

然后关闭并重新打开 PowerShell。

***

## 3. 安装 Node.js 24 和 pnpm

### 3.1 使用 nvm-windows 安装 Node 24

如果你的 Node 环境在 `D:\nvm`，通常可以直接使用 nvm-windows：

```powershell
nvm install 24
nvm use 24
node -v
```

确认输出类似：

```text
v24.x.x
```

### 3.2 启用 Corepack 并使用项目指定 pnpm

```powershell
corepack enable
corepack pnpm --version
```

期望输出：

```text
10.33.2
```

如果 `corepack enable` 因权限失败，可以改用全局安装：

```powershell
npm install -g pnpm@10.33.2
pnpm -v
```

***

## 4. 安装项目依赖

进入项目根目录：

```powershell
cd D:\works\open-design
```

安装依赖：

```powershell
$env:ELECTRON_MIRROR = 'https://npmmirror.com/mirrors/electron/'
pnpm install
```

如果 pnpm 提示构建脚本被阻止，例如 `better-sqlite3`、`electron`、`esbuild`：

```powershell
pnpm approve-builds
pnpm install
```

如果出现 `gyp ERR! find VS could not find Visual Studio` 或 Windows SDK 相关报错，需要安装 **Build Tools for Visual Studio 2022**，并勾选：

- Desktop development with C++
- MSVC v143 - VS 2022 C++ x64/x86 build tools
- Windows 11 SDK

安装完成后重新打开 PowerShell，再执行：

```powershell
pnpm install
pnpm tools-dev run web
```

***

## 5. Agent CLI 与模型配置

Open Design 可以调用本机已安装的代码 Agent CLI，例如 Claude Code、Codex、Gemini CLI、OpenCode、Cursor Agent、Qwen 等。

### 5.1 不安装本地 Agent 可以启动吗？

可以。系统本身可以启动。

如果没有本地 Agent CLI，可在应用设置中使用 BYOK/API 模式配置：

- OpenAI-compatible endpoint
- Anthropic
- Google Gemini
- Azure OpenAI
- Ollama / LM Studio / vLLM 等本地或兼容接口

### 5.2 安装本地 Agent 后未识别

先在 PowerShell 中确认命令可用：

```powershell
where.exe opencode
opencode --version
```

如果命令不可用：

1. 确认 CLI 已安装成功。
2. 确认 CLI 所在目录已加入用户 PATH。
3. 重新打开 PowerShell。
4. 重新启动 Open Design。

如果命令可用但界面仍显示未安装：

1. 打开 Open Design。
2. 进入 `Settings -> Execution mode`。
3. 点击 Rescan。
4. 仍不行时执行：

```powershell
pnpm tools-dev restart
```

***

## 6. 自定义设计体系与 Logo 使用

如果你为公司创建了专属设计体系（例如 `design-systems/siwei-executive-education/`），并希望在生成产物（PPT、海报、网页等）中正确使用品牌 Logo，可参考以下方法。

### 6.1 设计体系 + Logo 使用规则

1. 把 Logo 使用规则写进设计体系：
   - 在 `DESIGN.md` 的 **Layout & Composition** 或 **Voice & Brand** 部分增加“Logo 使用规则”。
   - 在 `USAGE.md` 中写明读取顺序和 Logo 文件位置。
   - 把 Logo 文件放在设计体系的 `assets/` 目录下，例如 `assets/logo.png`。

2. 生成时显式要求使用 Logo：

   即使设计体系里写了规则，生成时也建议在提示词中补充一句，确保 Agent 读取并使用 Logo：

   ```text
   使用“四为高管教育”设计体系。请读取并使用设计体系中的 Logo 文件：assets/logo.png。
   生成 PPT、封面、封底、海报和官网 Hero 时尽量露出四为 Logo；
   内容页可在页眉或页脚小尺寸露出，不要拉伸、变形、换色、加阴影。
   ```

   如果是生成 HTML / 网页 / 海报，提示词可以更明确：

   ```text
   请把四为 Logo 作为真实图片素材使用，优先引用或复制 design-systems/siwei-executive-education/assets/logo.png。
   如果生成物需要本地相对路径，请放到生成项目的 assets/logo.png，并在 HTML 中用 <img src="./assets/logo.png"> 引用。
   ```

### 6.2 内容结构应该放在哪里

设计体系（Design System）只负责**视觉身份**（颜色、字体、版式、组件、品牌语气、禁用风格），不负责**内容结构**。

内容结构（例如“先公司介绍，再课程介绍，最后客户案例”）建议按复用频率选择放置位置：

| 场景 | 推荐位置 | 说明 |
| --- | --- | --- |
| 一次性需求 | 本次生成提示词 | 直接在 prompt 里写明结构顺序，最灵活 |
| 经常复用的固定结构 | Skill | 做成 Skill（`skills/`），管理固定章节顺序、交付流程、写作口径 |
| 固定页面版式 | Design Template 或模板型 Skill | 不仅规定结构，还规定每页布局、标题区、Logo 区 |

**示例：生成课程介绍 PPT 的提示词**

```text
使用“四为高管教育”设计体系，采用高管课程模式，生成一份课程介绍 PPT。

必须使用四为 Logo：design-systems/siwei-executive-education/assets/logo.png。
封面、章节页、结束页必须出现 Logo；内容页可在页脚小尺寸露出。

内容结构：
1. 公司介绍
2. 课程背景与客户痛点
3. 课程目标
4. 课程模块
5. 教学方式
6. 学员收益
7. 客户案例分享
8. 合作方式
9. 结束页

风格要求：
红黑为主，蓝色只作辅助；纯色，不要渐变；不要卡通、二次元、花哨插画。
```

**示例：生成海报的提示词**

```text
使用“四为高管教育”设计体系，采用品牌传播模式，生成一张高管课程招生海报。

请使用四为 Logo：design-systems/siwei-executive-education/assets/logo.png。
Logo 放在顶部或底部品牌区，不要变形，不要加阴影。

内容包括：课程名称、核心卖点、适合人群、时间地点、报名方式。
视觉使用四为红和四为黑，白底，纯色，不要渐变。
```

### 6.3 三层分工建议

```text
Design System：四为品牌视觉规范（颜色、字体、版式、Logo、禁用项）
Skill：四为业务产物生成流程（固定内容结构、章节顺序、交付口径）
Template：具体 PPT / 海报 / 官网页面结构模板（每页版式、标题区、Logo 区）
```

对应关系：

| 内容 | 放哪里 |
| --- | --- |
| 红黑主色、蓝色辅色、字体、Logo、不要渐变 | Design System |
| 公司介绍 → 课程介绍 → 客户案例分享 | Skill 或本次 prompt |
| 固定 PPT 页数、每页版式、标题区、Logo 区 | Template / Skill |
| 这次临时要讲哪些内容 | 本次 prompt |

***

## 7. 构建 Electron 桌面应用

### 7.1 构建可运行目录（不生成安装包）

```powershell
pnpm tools-pack win build --namespace dev --to dir --json
```

构建产物在：

```text
.tmp\tools-pack\out\win\namespaces\dev\builder\win-unpacked\
```

目录中的 `Open Design.exe` 可直接双击运行。

### 7.2 启动已构建的 Electron 应用

```powershell
pnpm tools-pack win start --namespace dev --json
```

### 7.3 停止运行中的应用

```powershell
pnpm tools-pack win stop --namespace dev --json
```

### 7.4 生成 NSIS 安装包

```powershell
pnpm tools-pack win build --namespace dev --to nsis --json
```

安装包在：

```text
.tmp\tools-pack\out\win\namespaces\dev\builder\Open Design-dev-setup.exe
```

### 7.5 其他常用命令

| 命令 | 用途 |
| --- | --- |
| `pnpm tools-pack win install --namespace dev --json` | 安装已构建的 NSIS 包 |
| `pnpm tools-pack win uninstall --namespace dev --json` | 卸载应用 |
| `pnpm tools-pack win logs --namespace dev --json` | 查看运行日志 |
| `pnpm tools-pack win cleanup --namespace dev --json` | 清理构建产物和运行时数据 |

> **注意**：`--namespace` 不要太长，Windows 路径有 260 字符限制。本地开发建议用 `dev`、`smoke` 等短名。

