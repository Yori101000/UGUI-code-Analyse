# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库性质

纯 Markdown 中文技术书：《Unity UGUI 源码级原理分析》，四部分 25 章 + `_阅读指南/`，共 31 个 `.md`（约 550KB）。**没有工程代码、没有构建/测试/lint 命令**——所有任务都是读、写、校对 Markdown。文中的 C# / Shader 片段是"简化后的源码摘录"，用于讲解，不会被编译。

## 先读这三个文件

进入仓库后按顺序读，本文件只补充它们没写的东西：

1. `AGENTS.md` — 行为守则（源码参考规则、强制维护义务），对所有 Agent 生效
2. `agent.md` — 导航手册（章节地图、快速定位表、源码引用基线、已知问题清单）
3. `README.md` — 总目录、章节依赖图、阅读路径

## 两条硬约束（违反会造成实际损坏）

**1. 事实基线：uGUI 官方仓库 main 分支**
https://github.com/Unity-Technologies/uGUI/tree/main

对任何 UGUI 类、方法、路径、行为不确定时，**必须先联网核实再下结论或修改**，禁止凭记忆或二手资料作答。本仓库已因此做过一次全库审计（`_阅读指南/源码一致性审计报告.md`，17 项修正）——编造 API、搞错引擎归属是这里最常见也最严重的错误类型。

引用源码事实时必须区分三种来源，不要把后两种升级为"源码证实"：

| 来源 | 典型对象 | 说法 |
|---|---|---|
| uGUI 仓库 C# 源码 | `Graphic`、`VertexHelper`、`CanvasScaler`、`ScrollRect`、`EventSystem/` | 可直接引路径与方法名 |
| 引擎内置、C# 侧不可见 | `Canvas`/`CanvasRenderer`/`UIVertex`/`TextGenerator`/`RenderMode`（`UnityEngine.UIModule`）、`RectTransform`（`CoreModule`）、`UI-Default.shader` | 以官方文档为准，内部行为标注"行为推断" |
| 独立包，不在 uGUI 仓库 | TextMeshPro（`com.unity.textmeshpro`，路径 `Scripts/Runtime/`） | 单独说明归属 |

注意 main 是 `com.unity.ugui` 包结构（`Runtime/UGUI/UI/Core/`），旧分支才是 `UnityEngine.UI/UI/`；`EventSystem/` 与 `UI/Core/` 同级。

**2. 改完必须同步 `agent.md`**

任何内容改动（新增/删除/重命名章节、编辑正文、更新 `_阅读指南/`、修勘误、动文件结构）完成后，回到 `agent.md` 同步第 2 节章节地图、快速定位表、第 4 节已知问题清单，并在最终回复中明确说明"已同步 agent.md"或"本次修改不影响 agent.md"。这是 `AGENTS.md` 规定的强制义务，不是可选项。

## 编码：UTF-8 无 BOM

全库统一。PowerShell 读取必须显式带 `-Encoding UTF8`，否则中文乱码：

```powershell
Get-Content -Encoding UTF8 "第一部分_基础理论\第05章_Graphic系统.md"
```

用 Read/Write/Edit 工具则无此问题，优先用工具而非 shell 读写。

## 章节文件与写作模板

文件名格式 `第NN章_主题.md`，**文件名编号（01~25）是章节顺序的唯一可信来源**。目录即部分归属：

- `第一部分_基础理论/` 01~09 ｜ `第二部分_渲染链路/` 10~19 ｜ `第三部分_性能与工具/` 20~21 ｜ `第四部分_工程实践/` 22~25

每章固定结构，新增或改写时保持一致，不要引入新的结构约定：

```
# 第N章 主题
> 引言 blockquote
## 概述
## N.1 / N.2 ...（编号小节，与章号一致）
## 源码阅读路径      ← 调用链 ASCII 图 + 方法说明表
## 本章总结
## 勘误汇总          ← 全部 25 章都有此段
```

风格：大量表格、ASCII 框图、简化 C# 片段、性能数字对比。无仓库源码的章节，「源码阅读路径」用"官方文档/行为推断/第三方库"说明代替。

## 勘误优先于静默改写

发现源码级错误时，**在该章「勘误汇总」表追加一行，保留"原文声称 vs 实际情况"对照**，而不是直接把正文改掉了事：

```markdown
| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 | 03 / 3.6 | RectTransformUtility 是 UGUI 源码文件 | 引擎内置类型（UnityEngine.UIModule） |
```

严重程度：🔴 严重 / 🟡 中等 / 🟢 轻微。跨章批量审计另在 `_阅读指南/源码一致性审计报告.md` 留明细。

## 编号陷阱

正文中的「本章对应原书结构中的第 N 章」「原文第 N 章 / 原文 N.M 节」指的是**重构前的旧编号**，是纠错留痕，**不能据此判断当前章节号**。用户说"第 N 章"时先确认是否指文件名编号；回复中一律使用文件名编号并给出文件链接。

`_阅读指南/整体结构分析（历史存档）.md` 已过时，仅供追溯，勿当现状引用。

## 提交约定

分支 `master`，提交信息为中文单条描述。全库性改动（编号统一、目录重组、章节合并）作为**独立提交**，与内容修改分离，便于 review 和回滚。新增章节需同步 `README.md` 的目录表与依赖图。
