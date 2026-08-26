# DeepSeek Harness（dsh）详解

> 一切皆插件 · Everything is a Plugin

本文介绍 **DeepSeek Harness（简称 dsh）** 是什么、能做什么、以及如何上手使用。内容基于 [DeepSeek 官方介绍](https://deepseek.com/harness/zh/)、[GitHub 仓库](https://github.com/deepseek-ai/deepseek-harness) 与用户指南整理，适合想快速理解并试用 dsh 的开发者。

---

## 目录

1. [它是什么](#1-它是什么)
2. [和「模型」有什么区别](#2-和模型有什么区别)
3. [核心设计：一切皆插件](#3-核心设计一切皆插件)
4. [能做什么](#4-能做什么)
5. [四种运行模式](#5-四种运行模式)
6. [环境要求](#6-环境要求)
7. [怎么用：快速上手](#7-怎么用快速上手)
8. [配置模型与工作区](#8-配置模型与工作区)
9. [插件与扩展](#9-插件与扩展)
10. [常用命令速查](#10-常用命令速查)
11. [适用场景与注意事项](#11-适用场景与注意事项)
12. [相关链接](#12-相关链接)

---

## 1. 它是什么

**DeepSeek Harness（`dsh`）** 是由 [DeepSeek AI](https://deepseek.com) 开源的 **Agent Harness（智能体运行时 / 装具）**。

一句话概括：

> **模型是 Agent 的灵魂；Harness 让 Agent 能理解环境、调用工具，并在真实场景中持续工作。**

更具体地说，dsh 不是一个新的大模型，也不自带模型权重。它提供的是：

- 本地可运行的 **Web UI** 与无界面（headless）运行入口
- 基于 **Cordis** 的插件内核（加载 / 卸载 / 依赖管理）
- 可替换的 **模型适配、工具、会话、沙箱、存储、循环、调度、UI** 等全部能力
- **可追溯** 的会话事件流（Trajectory）：恢复、分叉、检索、回放都基于同一份日志

当前状态：**Developer Preview（开发者预览版）**，迭代快，**可能出现破坏性变更**。许可证为 **MIT**。

### 名称易混提醒

网上还有一个同名的社区 Python 项目（`pip install deepseek-harness`），那是 DeepSeek API 的客户端库，**与官方 TypeScript 版 dsh 无关**。本文只讨论官方项目：

| 项目 | 说明 |
| --- | --- |
| 官方 dsh | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)，Node.js / TypeScript，本地 Web UI |
| 社区 Python 包 | 独立作者的 API 客户端，不要和官方 dsh 混用文档 |

---

## 2. 和「模型」有什么区别

可以用三层来理解：

| 层级 | 是什么 | 做什么 | 不做什么 |
| --- | --- | --- | --- |
| **Model（模型）** | DeepSeek / Claude / GPT 等 | 推理、决定下一步、选择要调用的工具 | 自己打不开文件、跑不了命令 |
| **Harness（dsh）** | Agent 运行时 | 提供工作区、工具注册、权限/沙箱、会话日志、持续循环 | 不训练、不托管模型权重 |
| **Cordis（内核）** | 通用插件框架 | 挂载 / 卸载插件、处理依赖与事件 | 不特指 DeepSeek，也不内置业务能力 |

公式：

```text
Agent ≈ Model + Harness
```

你配置好模型提供商后，dsh 负责把「想法」落到真实工作区里的读文件、改代码、跑 shell、搜索网页、调度子 Agent 等动作上。

---

## 3. 核心设计：一切皆插件

### 3.1 Cordis 内核

Cordis 只负责：

- 插件的加载与卸载
- 依赖关系解析
- 服务（services）与类型化事件（events）协作

**Agent 的具体能力全部在插件里**，不在「特权核心」硬编码。

### 3.2 九类能力都是插件

官方把能力大致归为这些类别（均可配置层替换 / 扩展）：

1. **Models** — 模型适配器（DeepSeek、Anthropic、OpenAI、自定义 OpenAI 兼容端点等）
2. **Tools** — 工具注册与受控执行管线
3. **Skills** — 可复用的技能 / 指令包
4. **Sessions** — 仅追加（append-only）的会话事件日志
5. **Sandboxes** — 进程与文件副作用隔离（只读 / 工作区可写 / 全开等）
6. **Storage** — 会话与大数据落盘后端
7. **Loops** — Agent 循环本身也是插件，可换不同 turn 策略
8. **Scheduling** — 后台任务、提醒、子 Agent 调度
9. **UI** — 浏览器端、HTTP 服务等界面层也是 bundle

### 3.3 配置组合，而不是改源码

一次运行由多层配置拼出「插件树」：

1. Profile 声明的 **Bundles**（按顺序打补丁）
2. Profile 自己的 `cordis.patch.yml`
3. 全局（Harness home）级别的 patch
4. 命令行 `--patch` 覆盖层

**后层覆盖前层**；改配置即可替换模型、沙箱、工具集或 UI，而无需 fork 仓库改核心。

可用：

```sh
dsh web --dump-config
```

打印当前机器实际启动的插件树，再按需 patch。

### 3.4 每次运行都有迹可循

模型「看见」的一切都会写入 **仅追加** 的会话日志，例如：

- 系统提示词
- 思维链 / reasoning
- 工具调用与返回结果
- 子 Agent 调度
- 各类上下文注入

在 **Trajectory** 视图可按来源查看。**Resume / Fork / Search / Replay** 都基于同一事件流——原则是：**模型可见 ⇒ 必须可从日志重建**。

---

## 4. 能做什么

把 dsh 当成「可组合的本地编码 / 任务 Agent 平台」，典型能力包括：

### 4.1 编码与工程协作

- 在选定工作区中 **读 / 写 / 搜索文件**
- 执行 **Shell** 命令（受权限与沙箱策略约束）
- 网页与文件检索、规划（planning）、目标（goals）
- **Skills、子 Agent、工作流** 协作完成复杂任务
- 敏感操作按策略 **先征求批准再执行**

### 4.2 可观测与可复现

- Trajectory 审阅完整轨迹
- 会话恢复、分叉实验、检索历史、回放
- 便于调试 prompt、工具结果与调度行为

### 4.3 模型评测与对照

- **Minimal 模式** 只保留 bash + 文件编辑器，环境面稳定，适合横向对比不同模型
- 可固定工具面，减少「工具太多导致评测噪声」

### 4.4 自定义 Agent / 平台嵌入

- 用 Creator 模式检查运行时、内存试验 Cordis 插件、创作新 preset
- 团队可自建内部 Agent 运行时，而不是绑定封闭产品
- Headless 模式适合脚本化、CI、无 UI 批处理

### 4.5 插件生态扩展

社区通过 GitHub topic [`dsh-plugin`](https://github.com/topics/dsh-plugin) 发布扩展，例如：

- MCP 客户端（接外部工具服务）
- 视觉 / 搜索 / `@file` 引用 / 桌面通知
- 浏览器操控、侧边栏增强、主题与皮肤
- 终端 TUI 前端、Agent Teams、沙箱增强等

> 社区插件未经 DeepSeek 官方安全审计；装入能访问本地文件的 profile 前请自行审阅源码。

---

## 5. 四种运行模式

四种模式共享同一插件内核，差别在「挂了哪些能力」：

| 模式 | 定位 | 主要能力 | 适合 |
| --- | --- | --- | --- |
| **Standard（标准）** | 完整编码 Agent | 文件编辑、Shell、文件/网页检索、Skills、计划、目标、子代理、工作流 | 日常仓库开发 |
| **Code / PTC（代码编排）** | 用代码组合多步工具 | 标准能力 + Code Mode SDK，模型用 TypeScript 程序编排多轮工具调用 | 复杂多步操作希望一次编排 |
| **Minimal（极简）** | 双工具环境 | 持久 bash + `str_replace_editor` | 模型基准测试、最小环境对照 |
| **Creator（创造）** | Preset / 插件创作 | 标准能力 + 运行时检查、内存插件实验、preset 创作指导 | 开发自定义 Agent 形态 |

怎么选：

- 写业务代码、改仓库 → **Standard**
- 希望模型用一段 TS 编排多工具 → **Code**
- 评测模型、控制变量 → **Minimal**
- 造新 preset / 试插件 → **Creator**

---

## 6. 环境要求

| 项目 | 要求 |
| --- | --- |
| Node.js | **22.19+** 或 **24+**（`^22.19.0 \|\| >=24.0.0`） |
| 包管理（源码） | 通过 Corepack 使用仓库锁定的 **pnpm**（如 11.7.0） |
| Git（源码） | **2.26+** |
| 操作系统 | Linux / macOS / Windows（沙箱后端因平台而异） |
| 模型 | 至少一个已配置的提供商（DeepSeek API Key、目录内其他厂商、或自定义 OpenAI 兼容端点） |
| Web UI 默认地址 | `http://127.0.0.1:3080` |
| 网络监听 | 目前 **不支持** `--host 0.0.0.0` 对外网开放（本地使用） |

---

## 7. 怎么用：快速上手

### 方式 A：npx 一键启动（推荐先试用）

1. 安装符合版本要求的 Node.js  
2. 在任意目录执行：

```sh
npx @deepseek-ai/dsh web
```

3. 默认打开本地浏览器访问 `http://127.0.0.1:3080`  
4. 若不想自动打开浏览器：

```sh
npx @deepseek-ai/dsh web --no-open
```

SSH 远程启动时通常只打印主机 URL，由你在本机做端口转发后访问。

### 方式 B：从源码安装（便于开发 / 调试）

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

说明：

- `pnpm run build` 会准备仓库产物（生产运行需要构建后的包与前端资源）
- `pnpm dsh web` 使用已构建产物启动，不会自动再 build 一遍

---

## 8. 配置模型与工作区

服务启动后，按官方 Web UI 指南完成两步即可开始对话：

### 8.1 配置模型

1. 打开 **Settings → Models**
2. 填入 [DeepSeek API Key](https://platform.deepseek.com/) 并保存  
3. **无需重启** 即可使用该模型路由  

也可按官方「模型配置指南」接入其他厂商或 **自定义 OpenAI 兼容端点**。部分云厂商（如 Bedrock / Vertex / Azure 等）需使用其原生凭证，而不只是单一 API Key 字段。

常见问题：

| 现象 / 错误 | 原因 | 处理 |
| --- | --- | --- |
| `MISSING_CREDENTIAL` | 未存 Key，或环境变量未设置 | 在 Models 页保存 Key，或导出对应环境变量后再启动 |
| `UNKNOWN_MODEL` | 模型 ID / 路由未识别 | 检查模型配置与可用目录 |

### 8.2 选择工作区（Workspace）

1. 点击 **Choose workspace**
2. 添加并选中你的项目目录  

**未选择工作区时，会话输入框不可用。**  
`dsh` 进程的启动目录只是默认文件系统位置，不等于已选中工作区。

### 8.3 跑第一个任务

新建会话后可以试：

> Summarize this repository and identify its main packages.  
>（总结本仓库并指出主要包结构。）

Agent 可以读改工作区文件、执行命令、委派子任务、维护计划；在当前权限策略下需要批准的操作会先弹确认。

---

## 9. 插件与扩展

### 9.1 安装社区插件（示例）

插件通常安装到某个 **profile**（例如 Web UI 用的 `web`）：

```sh
# 向 web profile 添加插件包
dsh plugin --profile web add <package-or-source>

# 查看合成后的配置
dsh --profile web --dump-config

# 需要时重启 Web UI
dsh web
```

也可：

```sh
dsh plugin add <package>
```

（具体是否默认写到当前 profile，以你安装的 CLI 版本帮助为准。）

### 9.2 自己写插件（概念）

典型 DSH 插件是一个 TypeScript 模块，导出 `apply(ctx)`：

- 通过 `ctx` 注册服务、工具、事件监听、可逆副作用（`ctx.effect`）等
- 可用 `inject` 声明依赖服务，加载器会在依赖就绪后再调用 `apply`
- 插件卸载时，通过 `ctx` 做的注册会自动回滚

分发形态常见两类：

1. **Bundle 插件**：npm 包 + `dsh.bundle` 指向 patch  
2. **Repository 插件**：仓库内 `.dsh-plugin` 目录，经 patch / `repository-plugins` 挂载  

发布到 GitHub 时可加上 topic **`dsh-plugin`**，方便被生态发现。

开发文档入口见仓库：`docs/development.md`、`docs/architecture.md`、以及插件开发指南 `docs/user/develop/`。

---

## 10. 常用命令速查

```sh
# 启动 Web UI
npx @deepseek-ai/dsh web
# 或源码目录：
pnpm dsh web

# 不自动打开浏览器
npx @deepseek-ai/dsh web --no-open

# 打印当前合成的插件树 / 配置
dsh web --dump-config

# 管理 profile 插件
dsh plugin --profile web add <pkg>
dsh plugin --profile web remove <pkg>
```

其他能力（以仓库文档为准）：

- Headless / 其他 CLI 模式：见 `apps/cli/README.md`
- Python SDK：见 `docs/user/guide/python-sdk.md`
- 模型提供商细节：见 `docs/user/guide/providers.md`

---

## 11. 适用场景与注意事项

### 适合

- 希望 **自有 / 可组合** Agent 运行时，而不是封闭黑盒产品
- 要写、测、发布 **dsh-plugin**
- 需要 **稳定最小工具面** 做模型评测
- 要把可观测的 Agent 循环嵌进内部平台或自有 UI
- 想用配置 **热切换** 模型适配器、沙箱或存储后端

### 暂不适合 / 需谨慎

- 需要 **API 长期稳定** 的生产强依赖（预览版会破坏兼容）
- 想把 Web UI **直接暴露到公网**（当前不支持 `0.0.0.0` 监听）
- 期望「零配置成品产品」、不愿碰 profile / patch 的用户
- 未经审查就安装能读写本机文件的第三方插件

### 安全提示

- Agent 在工作区内具备强执行力；请使用合适的沙箱 / 审批策略
- API Key 等凭证通过 Settings 或环境变量管理，勿提交进仓库
- 社区插件请先读源码再安装

---

## 12. 相关链接

| 资源 | 链接 |
| --- | --- |
| 官方介绍（中文） | https://deepseek.com/harness/zh/ |
| 官方介绍（英文） | https://deepseek.com/harness/en/ |
| GitHub 仓库 | https://github.com/deepseek-ai/deepseek-harness |
| npm 包 | `@deepseek-ai/dsh` |
| Cordis 内核 | https://github.com/cordiverse/cordis |
| Cordis 设计论文 | https://github.com/cordiverse/paper |
| 社区插件 Topic | https://github.com/topics/dsh-plugin |
| Discussions | https://github.com/deepseek-ai/deepseek-harness/discussions |
| DeepSeek 开放平台 | https://platform.deepseek.com/ |

---

## 附录：一分钟心智模型

```text
你描述目标
    ↓
dsh 按 Profile + Bundles + Patch 组装插件树
    ↓
模型在循环中推理并调用工具
    ↓
工具在沙箱 / 权限策略下作用于工作区
    ↓
一切写入 append-only Session 日志
    ↓
可在 Trajectory 中审查、恢复、分叉、回放
```

**记住三句话：**

1. dsh 是 **Harness**，不是新模型。  
2. **一切皆插件**，用配置组合能力。  
3. **运行有迹可循**，日志即真相。

---

*文档整理日期：2026-08-25。DeepSeek Harness 处于开发者预览阶段，命令与 API 以官方仓库最新文档为准。*
