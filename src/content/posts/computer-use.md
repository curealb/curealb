---
title: computer-use
published: 2026-11-06
description: ''
image: ''
tags: [agent]
category: 'AI'
draft: false 
lang: 'zh_CN'
---

根据 2026 年 11 月的官方文档和 GitHub 项目，ChatGPT App 的 Computer Use（以下简称 CU）已经有相当多的技术细节可以研究。

首先有三个重要发现：

* ChatGPT 桌面版的 CU 采用了 MCP Server + Skill 的插件架构。 它可以控制 macOS、Windows 的真实桌面程序，并且 macOS 支持后台操作。ChatGPT Learn
* OpenAI 已经开源了一套完整的 CU 示例实现，包括模型调用循环、截图反馈、鼠标键盘控制和持久化执行环境。代码采用 MIT 许可证，但 ChatGPT 产品使用的是另一套内部运行时。GitHub
* 社区还有专门实现后台控制的开源 CU 驱动，能够利用 macOS Accessibility、Windows UI Automation 等系统接口，在部分场景下实现不抢占鼠标的操作。GitHub +1

下面分别分析它的原理、开源实现以及逆向情况。

一、ChatGPT App 的 CU 是怎么实现的？

从官方公开信息看，可以分成三个层次：

Agent 决策层

理解任务、分析界面、生成操作代码或结构化指令

工具调用与执行反馈

CU 插件与执行层

MCP Server · Skill · 权限控制 · 执行环境

截图、控件信息、输入事件

macOS

Screen Recording / Accessibility

Windows

前台桌面操作

依据官方披露的信息整理的逻辑架构，非 ChatGPT 客户端内部模块的逆向结果。

典型执行过程就是：

观察界面 → 决定操作 → 执行操作 → 获取新界面 → 验证结果

OpenAI 公开支持两种操作方式：模型生成 Python/JavaScript 代码，由执行器调用 PyAutoGUI、Playwright 等库；或者模型直接输出 click、scroll、keypress 等结构化操作。官方目前对 GPT-6 Astra 推荐代码执行方式，因为它能够在一次调用中完成循环、条件判断和多个 UI 操作。OpenAI API

这也解释了为什么 CU 和传统的鼠标宏存在明显的能力差异：Agent 可以在操作之后重新观察环境，根据结果改变后续步骤。

二、ChatGPT CU 的逆向工程：已经有人做了

最有价值的发现是，macOS 版 ChatGPT 的 CU 内部使用了一套名为 Sky 的组件。社区已经找到了它的 SDK、接口规范、原生服务以及通信协议。

1. mikilolipop/mac-computer-use：直接逆向 Sky

mac-computer-use⁠￼

这个仓库包含中文的逆向报告，记录了 ChatGPT 安装包中的实际文件及调用关系。作者还提供了 IPC 协议重构客户端、Mock 守护进程，以及独立的 Swift CU 引擎。GitHub

根据作者的分析，macOS 端大致有如下调用链：

ChatGPT / Codex Agent

Node REPL + @oai/sky SDK

Unix Domain Socket

长度前缀 + JSON-RPC 2.0

SkyComputerUseService

macOS 原生窗口、无障碍和输入接口

据这份逆向报告：

* @oai/sky 封装应用控制 API。
* SkyComputerUseService 是原生执行服务。
* IPC 使用 Unix Domain Socket，消息具有 4 字节小端长度前缀及 JSON-RPC 负载。
* macOS 端利用 AXUIElement 获取控件语义信息，结合截图和输入事件进行操作。

其中协议帧格式、具体类名及内部构建方式属于第三方逆向结论，可能随版本改变。GitHub

2. 真正值得注意的是 Accessibility Tree（无障碍树）

ChatGPT CU 可以利用 UI 的结构信息定位控件。比如一个桌面窗口可能给出类似这样的内容：

Window: Settings
  [4] Tab: General
  [5] Tab: Appearance
  [6] Checkbox: Auto Save, checked=false
  [7] Button: Apply

模型知道第 6 个元素是复选框，于是可以直接通过元素编号执行操作。逆向发现的 sky 接口包括：

const state = await sky.get_app_state({
  app: "TextEdit"
});
await sky.click({
  app: "TextEdit",
  element_index: 7
});
await sky.type_text({
  app: "TextEdit",
  text: "Hello"
});

此外，get_app_state 支持返回无障碍树的增量变化（diff）。Agent 可以只观察上一步之后变化的内容，从而降低大量截图和重复 UI 信息造成的 Token 消耗。GitHub +1

这套语义定位机制对复杂软件很重要。例如你之前遇到的 UE 编辑器 MCP 无法设置特殊属性的问题，CU 可以尝试通过属性面板、按钮、菜单等实际界面操作。对于没有暴露完整无障碍信息的 Slate 控件，仍可能需要截图和坐标定位。

3. 已经有人把 OpenAI 原版 CU 接入第三方 Agent

直接复用官方组件

tmustier/codex-computer-use-mcp⁠￼

这个项目会调用本机已安装的、由 OpenAI 签名的 CU 组件，将 10 个官方 Computer Use 方法暴露为 MCP 工具。

它支持 Pi 等 MCP 客户端，能够复用 Sky 的应用状态、控件编号和截图能力。其实现基于实验性的 Codex app-server 接口，因此 ChatGPT 更新可能破坏兼容性。GitHub

更有意思的是下面这个项目。

ckanner/dsh-plugin-lcu⁠￼

这是专门为 DeepSeek Harness 开发的插件，能够启动 ChatGPT App 随附的 CU 运行时，并通过 MCP 与它通信。

它支持 macOS Apple Silicon，可以调用桌面应用和 Chrome，还实现了应用授权、执行生命周期清理等机制。开发者说明，该插件无需通过 ChatGPT 账号认证来调用本地 CU 组件。GitHub

插件的安装命令是 dsh plugin --profile <profile> add dsh-plugin-lcu，但完整使用还需配置 Harness 的 preset。它相当适合研究 DeepSeek 模型能否复用 OpenAI 的桌面控制层。需要留意项目规模较小，以及第三方接入带来的兼容性和权限风险。

三、有哪些值得研究的独立开源 CU？

下面这些项目有完整的开源实现，无需依赖 ChatGPT 自带的原生 CU 服务。

OpenAI CUA Sample App⁠￼

OpenAI 官方示例，MIT 许可证。提供 JavaScript + Playwright 浏览器控制，以及 Python + PyAutoGUI 桌面控制，包含执行循环和截图记录。适合研究 Agent Harness 的基础实现。GitHub

trycua/cua⁠￼

相对成熟的 CU 基础设施，包含跨平台驱动、虚拟机及评测工具。macOS 后台控制方案研究了 SkyLight 私有接口及定向输入。项目持续活跃，2026 年 11 月 2 日仍有新版构建。GitHub +1

![Excited to share that I will be joining Microsoft Research as a Research Intern over the Summer working with Scott Counts and the Special Projects team!

Really looking forward to working with some… | Shramay Palta](https://images.openai.com/static-rsc-4/7PZBnUciC_QSG8sKR8JMSwnxR2O6lTUfV2Os023S9KmIpEzk3KYq2q9CmwUlx_K4te2z2A1qhJ-cdXrJ6-T2CB5OfEjKyIxXdfEXg34Aa3zhPcPOb7nV0VajNL8FBT1UzP4V2AzSHyVGSgczroM_hBb_Ez4WZA8-FxVz0HVx8W_iGVcMogzqZ5gzvPEfN2YV?purpose=fullsize)

Microsoft Hybrid Computer Use⁠￼

微软研究院的开源项目，MIT 许可证，支持 Windows、macOS、Linux。将 Accessibility、截图 GUI 和程序控制结合，并提供元素状态校验与批量操作。很适合学习现代 CU 的系统设计。GitHub

ByteDance UI-TARS Desktop⁠￼

字节跳动的开源 GUI Agent，结合视觉语言模型与桌面控制，支持 Windows、macOS 和浏览器。适合研究专门训练的 GUI 模型如何执行复杂交互。其近期维护状态还需要进一步核对。GitHub

四、真正的技术难点在哪里？

从这些项目看，CU 至少存在三个值得深入研究的工程问题。

第一，如何准确定位 UI 元素。 纯视觉方案需要模型判断按钮的像素坐标，而 AX 方案可以获得控件类型、文本、位置、启用状态和层级结构。成熟方案通常结合两种感知通道，并使用状态校验避免点击已经移动的控件。

第二，如何在后台控制应用。 Windows 官方 CU 当前占用前台桌面；macOS 可以在用户继续工作的同时操作其他应用。trycua 的公开研究展示了如何利用 SkyLight 的定向事件和窗口管理技术实现后台控制，不过这只能证明一种可行实现，OpenAI 自身采用的全部底层机制仍未公开。ChatGPT Learn +1

第三，如何减少 Agent 与执行器之间的通信开销。 每一次点击都重新调用大模型会产生高延迟。代码执行式 CU 可以一次执行多个动作、条件判断及局部验证；无障碍树增量反馈进一步减少上下文。OpenAI 官方已经明确推荐代码执行式集成作为主要路径之一。OpenAI API