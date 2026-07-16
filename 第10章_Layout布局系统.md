# 第10章 Layout 布局系统（修正版）

> 本文是对原文的精炼总结，并对照 UGUI 源码修正了原文中的 6 处错误。

---

## 概述

在 UGUI 体系中，RectTransform 定义了 UI 的空间表达能力，而 Layout 系统负责决定这些空间如何被组织与分配。Layout 的核心目标是将"手动摆放 UI"转变为"规则驱动布局"。

整个 Layout 系统由以下部分协同组成：

| 组件 | 职责 |
|------|------|
| **LayoutGroup** | 父级驱动，负责子节点排列规则 |
| **LayoutElement** | 子级反馈，提供自身尺寸需求（min / preferred / flexible） |
| **ContentSizeFitter** | 子级反向驱动，根据内容调整容器自身尺寸 |
| **ILayoutElement** | 接口，描述"我需要多少空间" |
| **ILayoutController** | 接口，负责"最终分配多少空间" |
| **LayoutRebuilder** | 单节点重建执行器 |
| **CanvasUpdateRegistry** | 统一调度所有 Layout 重建 |

Layout 的执行发生在 UI 生命周期中的 **Layout Rebuild 阶段**。布局计算的最终结果，本质上都是对 `anchoredPosition`、`sizeDelta` 等 RectTransform 参数的重新计算与写入。Layout 系统本身不参与渲染——它只负责驱动 RectTransform。

---

## 10.1 Layout 系统架构

### 10.1.1 核心职责

Layout 系统做两件事：
1. **计算 UI 需要的空间**（自下而上：子节点向父节点反馈尺寸需求）
2. **计算 UI 应该摆放的位置**（自上而下：父节点根据规则为子节点分配空间）

这是一种"父级驱动 + 子级反馈"的混合模型。

### 10.1.2 整体执行流程

```
UI 变化（内容/结构/尺寸改变）
  → 标记节点为 Dirty
  → CanvasUpdateRegistry 在 Layout Rebuild 阶段收集所有 dirty 节点
  → 对每个 dirty 节点，通过其 LayoutRebuilder 执行重建：
      阶段一：自下而上的尺寸计算（子节点 → 父节点）
      阶段二：自上而下的空间分配（父节点 → 子节点）
  → 将结果写入 RectTransform
```

> ⚠ **勘误 #1**：原文将 LayoutRebuilder 描述为"调度中心"，称其"统一遍历所有待重建节点"。实际上，**遍历和调度是由 `CanvasUpdateRegistry.PerformUpdate()` 完成的**。`LayoutRebuilder` 实现了 `ICanvasElement` 接口，它只负责**单个节点**的 `Rebuild()` 方法。当一个 RectTransform 需要重建时，`LayoutRebuilder.MarkLayoutForRebuild()` 会创建/获取对应的 LayoutRebuilder 实例并注册到 CanvasUpdateRegistry，后续由 CanvasUpdateRegistry 统一调度。

### 10.1.3 ILayoutElement — 尺寸数据提供层

`ILayoutElement` 是 Layout 系统最核心的接口之一，负责提供：

- `minWidth` / `minHeight` — 最小尺寸
- `preferredWidth` / `preferredHeight` — 首选尺寸
- `flexibleWidth` / `flexibleHeight` — 弹性拉伸权重
- `layoutPriority` — 优先级（当多个 ILayoutElement 共存时，优先级高的优先生效）

Text 组件的 preferredWidth 取决于文本内容长度、字体大小；LayoutElement 组件则可以手动指定这些值。Layout 系统综合所有子节点的 ILayoutElement 信息来计算最终尺寸。

### 10.1.4 ILayoutController — 布局控制层

`ILayoutController` 负责真正执行布局计算：

- `SetLayoutHorizontal()` — 水平方向布局
- `SetLayoutVertical()` — 垂直方向布局

HorizontalLayoutGroup、VerticalLayoutGroup、GridLayoutGroup 以及 ContentSizeFitter 都实现了 ILayoutController。

**职责分工**：`ILayoutElement` 描述"自己需要多少空间"，`ILayoutController` 决定"最终实际分配多少空间"。

### 10.1.5 Layout 与 RectTransform 的关系

所有布局计算结果，最终都转化为对 RectTransform 参数的修改：

- `anchoredPosition` — UI 相对于 Anchor 的位置
- `sizeDelta` — UI 相对于 Anchor 的尺寸偏移
- `offsetMin` / `offsetMax` — UI 边界的绝对偏移

一旦某个 RectTransform 被 Layout 控制，其对应方向的属性就不再适合手动修改——Layout 会在下次 Rebuild 时覆盖你的修改值。这就是"代码刚设置位置，下一帧又被自动改回去"的根本原因。

### 10.1.6 Layout 是树形递归系统

父节点负责布局规则，子节点提供尺寸需求，布局计算沿 UI 树传播。深层嵌套时容易出现 Rebuild 扩散、循环依赖和性能问题——因为整个 Layout 本质上是一个带有依赖关系的递归计算体系。

---

## 10.2 LayoutGroup 原理

LayoutGroup 是 UGUI 自动布局最核心的组件，它主动接管子节点的 RectTransform，按预设规则重新计算子节点的位置、尺寸、间距和对齐方式。

### 10.2.1 继承结构

```
LayoutGroup（基类：子节点收集、Padding、对齐、生命周期）
├── HorizontalOrVerticalLayoutGroup（封装水平/垂直公共逻辑）
│   ├── HorizontalLayoutGroup（X 轴排列）
│   └── VerticalLayoutGroup（Y 轴排列）
└── GridLayoutGroup（二维网格，独立体系）
```

### 10.2.2 执行流程

LayoutGroup 的执行分两个阶段：

**阶段一 — 尺寸计算**（`CalculateLayoutInputHorizontal` / `CalculateLayoutInputVertical`）：遍历所有参与布局的子节点，统计它们的 Min Size、Preferred Size、Flexible Size，结合 Padding、Spacing、Child Alignment 等参数，计算出容器整体所需空间。

**阶段二 — 子节点排列**（`SetLayoutHorizontal` / `SetLayoutVertical`）：真正修改子节点 RectTransform 的 `anchoredPosition` 和 `sizeDelta`。通过 `SetChildAlongAxis()` 方法将计算结果写入。

### 10.2.3 子节点收集

LayoutGroup 在执行布局前会刷新 `rectChildren` 列表（`protected List<RectTransform>`）。并不是所有子节点都参与布局——以下节点会被排除：

- **inactive 的 GameObject**
- **启用了 `ignoreLayout` 的节点**（通过 `ILayoutIgnorer` 接口标记）

这在实际开发中很常用：飘字、动画节点、独立定位的 UI 通常需要脱离 Layout 控制。

### 10.2.4 关键参数

| 参数 | 作用 |
|------|------|
| **Padding** | 容器内边距（Left / Right / Top / Bottom） |
| **Spacing** | 子节点之间的间距 |
| **Child Alignment** | 子节点在容器内的对齐方式（Start / Center / End / Stretch） |
| **Child Control Size** | 是否由 LayoutGroup 接管子节点对应方向的尺寸 |
| **Child Force Expand** | 是否将剩余空间强制按 Flexible 权重拉伸分配 |

`Child Control Size` 和 `Child Force Expand` 的区别：
- Control Size = true：LayoutGroup 会修改子节点的 `sizeDelta`
- Force Expand = true：即使子节点 flexible 为 0，也会获得额外的弹性空间

### 10.2.5 Flexible 尺寸分配

Flexible 决定剩余空间如何按比例分配。例如在 HorizontalLayoutGroup 中，所有子节点 preferredWidth 计算完毕后，若父容器还有剩余空间，会根据每个子节点的 `flexibleWidth` 权重进行二次分配：`flexibleWidth=2` 的节点获得 `flexibleWidth=1` 节点两倍的额外空间。

这是 UGUI 自适应布局的核心机制。

### 10.2.6 GridLayoutGroup 的特殊性

GridLayoutGroup 采用固定网格结构，子节点尺寸统一（由 Cell Size 决定），而非动态计算。通过 Constraint 参数控制行列模式：

- **Fixed Column Count** — 固定列数，自动扩展行
- **Fixed Row Count** — 固定行数，自动扩展列
- **Flexible** — 根据容器宽度自动换行

适用场景：背包系统、图标列表等规则化布局。

### 10.2.7 性能问题

LayoutGroup 的性能开销来自：
- 每次变化触发子节点遍历和尺寸重新计算
- 深层嵌套时形成递归重建链（子 → 父 → 祖父）
- 与 ScrollView、ContentSizeFitter、动态 Text 组合时尤为明显

**优化方向**：减少嵌套层级、将静态 UI 和动态 UI 分离到不同 Canvas、避免在热更新路径上依赖 Layout 自动计算。

---

## 10.3 ContentSizeFitter

### 10.3.1 核心原理

ContentSizeFitter 实现了 `ILayoutSelfController` 接口，它的特殊之处在于：**根据内容反向调整自身尺寸**。普通 LayoutGroup 是"父 → 子"的方向，ContentSizeFitter 是"子 → 父"的方向。

> ⚠ **勘误 #2**：原文称"它会读取当前节点上的 Layout 尺寸信息"。实际上，ContentSizeFitter 在其 `SetLayoutHorizontal()` / `SetLayoutVertical()` 方法中，通过 `LayoutUtility` 工具类**实时遍历**当前节点及子节点上的所有 `ILayoutElement` 组件，计算 min/preferred/flexible 值，然后根据 Fit 模式写回 RectTransform。尺寸是**重新计算**的，不是"读取已有信息"。

### 10.3.2 Fit 模式

每个方向（Horizontal / Vertical）有三种模式：

| 模式 | 行为 |
|------|------|
| **Unconstrained** | 不接管该方向，尺寸由其他系统决定 |
| **Min Size** | 设置为所有 ILayoutElement 提供的最小尺寸 |
| **Preferred Size** | 设置为所有 ILayoutElement 提供的首选尺寸 |

Preferred Size 是最常用的模式——根据文本内容、布局结果或子节点尺寸自动调整自身 RectTransform。

### 10.3.3 经典场景：文本自适应

Text 组件根据字符内容计算出 `preferredWidth` 和 `preferredHeight`，ContentSizeFitter 读取这些值后自动调整自身 RectTransform。这就是聊天气泡、动态标签、自适应按钮的实现基础。

代价：文本变化会频繁触发 Layout Rebuild，大量动态 Text 会带来明显 CPU 开销。

### 10.3.4 与 LayoutGroup 的组合

VerticalLayoutGroup + ContentSizeFitter 是最常见的自动布局结构：LayoutGroup 负责排列子节点，ContentSizeFitter 根据排列结果自动调整容器高度。这适用于动态列表、ScrollView Content、聊天窗口等场景。

其本质是"先计算子节点布局，再反向驱动父节点尺寸"。

### 10.3.5 与 RectTransform 的驱动

一旦启用 ContentSizeFitter，对应方向上的 `sizeDelta` 就不再适合手动修改——ContentSizeFitter 会在 Layout 阶段重新覆盖。

### 10.3.6 循环依赖问题（详见 10.5）

ContentSizeFitter 是 Layout 系统中最容易引发循环依赖的组件。典型场景：父节点用 VerticalLayoutGroup，子节点用 ContentSizeFitter，而父节点自己也用 ContentSizeFitter。此时父节点尺寸依赖子节点，子节点布局又依赖父节点，形成闭环。

### 10.3.7 官方不推荐滥用

ContentSizeFitter 让 Layout 树从"父 → 子单向"变成"父子双向依赖"，层级一深就容易出现重复 Rebuild 和布局震荡。大型 UI 系统通常尽量减少深层 ContentSizeFitter 嵌套。

---

## 10.4 Layout 与 RectTransform 驱动关系

### 10.4.1 Layout 的最终目标

Layout 本身不渲染 UI——它的所有工作最终都体现为对 RectTransform 的修改。最常被驱动的属性：

- `anchoredPosition` — 相对于 Anchor 的位置
- `sizeDelta` — 尺寸偏移
- `offsetMin` / `offsetMax` — 边界偏移
- `anchorMin` / `anchorMax` — Anchor 设置

### 10.4.2 为什么手动修改会被覆盖

当 RectTransform 被 Layout 驱动时，手动修改和 Layout 自动控制是冲突的。LayoutGroup 会在下一次 Layout Rebuild 时重新计算并覆盖你手动设置的值。**一旦 Layout 接管了某个方向，该方向就不适合再手动控制。**

### 10.4.3 DrivenRectTransformTracker

Unity 提供了一套驱动标记系统 `DrivenRectTransformTracker`。LayoutGroup、ContentSizeFitter 等组件通过它注册正在驱动的属性（如 Position、SizeDelta、Anchors），告诉 Unity："这些属性当前由系统控制，不是用户手动设置的。"

Inspector 中某些 RectTransform 属性变灰/锁定的原因就在这里。

### 10.4.4 Layout 如何修改 RectTransform

Layout 不直接操作 Transform.position（UGUI 使用 Anchored Layout 体系）。LayoutGroup 在排列子节点时，核心调用路径是：

```
LayoutGroup.SetChildAlongAxis(child, axis, pos, size)
  └─ 修改 child.anchoredPosition（对应轴）
  └─ 修改 child.sizeDelta（对应轴）
```

### 10.4.5 RectTransform 坐标体系

普通 Transform 偏向三维世界坐标，RectTransform 额外引入 Anchor、Pivot、Offset、Anchored Position 等参数。UI 不再是"绝对位置"，而是"相对于父节点的布局关系"——这就是 UI 自动布局能适配不同分辨率的基础。

### 10.4.6 Layout 与 Anchor 的关系

Anchor 影响 Layout 写入结果的最终视觉效果：同样的 `sizeDelta`，在不同 Anchor 下表现完全不同。

> ⚠ **勘误 #3**：原文称"某些 LayoutGroup 会自动强制修改 Anchor，因为只有统一 Anchor 规则，布局计算才能稳定成立"。**UGUI 内置的 HorizontalLayoutGroup、VerticalLayoutGroup、GridLayoutGroup 以及 ContentSizeFitter 都不会修改子节点或自身 Anchor。** Anchor 始终由用户设置，Layout 仅通过 `SetChildAlongAxis()` 修改 `anchoredPosition` 和 `sizeDelta`。作者可能混淆了某个第三方 UI 扩展的行为。

### 10.4.7 驱动传播

Layout 修改 RectTransform → RectTransform 变化可能触发 Layout Rebuild → 再次修改 RectTransform，形成"修改 → Dirty → Rebuild → 再修改"的循环。这种递归更新链在复杂 Layout 中会被放大。

---

## 10.5 循环依赖问题

### 10.5.1 什么是循环依赖

循环依赖不是某个组件的 bug，而是 Layout "父子双向驱动机制"在复杂嵌套下的自然产物。当 ContentSizeFitter、LayoutGroup 与 LayoutElement 在特定结构下组合时，正常单向的分层链被打破，形成无法稳定收敛的闭环。

### 10.5.2 典型结构

```
父节点 VerticalLayoutGroup（依赖子节点高度计算自身尺寸）
  └─ 子节点 ContentSizeFitter（根据父节点宽度计算自身 preferredHeight）
```

任何一方变化 → 另一方重新计算 → 反馈回第一方 → 再次触发计算 → 无法稳定收敛。

### 10.5.3 跨层级递归

不止父子之间：父节点 VerticalLayoutGroup，子节点 HorizontalLayoutGroup，孙子节点 ContentSizeFitter + 动态 Text。孙子节点的尺寸依赖父级宽度，父级高度又依赖孙子节点内容——形成跨层级的递归依赖。

### 10.5.4 实际表现：Layout Rebuild 抖动

通常不表现为无限循环，而是 Layout Rebuild 抖动：
- UI 尺寸在多帧间轻微变化
- Layout 持续标记 Dirty
- Canvas 不断触发重建
- CPU 占用持续升高

根源往往是浮点误差或反馈不稳定——preferredHeight 在不同帧计算结果略有差异，导致父节点不断微调自身尺寸。

### 10.5.5 Unity 的保护机制

LayoutRebuilder 内部有一定保护（避免重复无意义计算、跳过相同结果），但这些机制只能降低概率，不能从根本上消除循环依赖。当结构足够复杂时仍会出现问题。

### 10.5.6 如何避免

核心原则：**尽量保持 Layout 单向驱动结构**。

- 避免父子节点同时使用 ContentSizeFitter
- 减少 LayoutGroup 嵌套层级
- 在复杂 UI 中用固定尺寸或脚本预计算替代实时 Layout 回流
- ScrollView 中避免完全依赖动态 Layout 驱动高度

---

## 10.6 自定义布局

### 10.6.1 为什么需要自定义布局

内置 LayoutGroup（Horizontal / Vertical / Grid）都是线性规则布局。遇到环形菜单、技能轮盘、瀑布流、曲线路径排列等需求时，需要在 UGUI Layout 框架内自定义批量驱动 RectTransform 的逻辑。

### 10.6.2 基础接口体系

- **ILayoutElement**：提供 minWidth / preferredWidth / flexibleWidth 等尺寸参数
- **ILayoutController**：`SetLayoutHorizontal()` / `SetLayoutVertical()`，执行真正的布局计算

实际开发中通常不直接从接口开始实现，而是**继承 LayoutGroup**——它已经封装了布局生命周期、子节点管理（`rectChildren`）、Padding 计算、Alignment 对齐和 Dirty 标记逻辑。你只需要关注布局规则本身。

### 10.6.3 生命周期

自定义 Layout 运行在 Layout Rebuild 生命周期中，关键回调：

```
CalculateLayoutInputHorizontal()  // 阶段一：统计子节点水平尺寸需求
CalculateLayoutInputVertical()    // 阶段一：统计子节点垂直尺寸需求
SetLayoutHorizontal()             // 阶段二：执行水平布局，修改 RectTransform
SetLayoutVertical()               // 阶段二：执行垂直布局，修改 RectTransform
```

### 10.6.4 环形布局示例

```csharp
using UnityEngine;
using UnityEngine.UI;

public class CircleLayoutGroup : LayoutGroup
{
    [SerializeField] private float radius = 200f;
    [SerializeField] private float startAngle = 0f;

    private float centerX, centerY, angleStep;
    private int count;

    public override void CalculateLayoutInputHorizontal()
    {
        base.CalculateLayoutInputHorizontal();
        count = rectChildren.Count;
        if (count == 0) return;
        centerX = rectTransform.rect.width * 0.5f;
        angleStep = 360f / count;
    }

    public override void CalculateLayoutInputVertical()
    {
        if (count == 0) return;
        centerY = rectTransform.rect.height * 0.5f;
    }

    public override void SetLayoutHorizontal()
    {
        // 仅计算水平方向，避免与 SetLayoutVertical 重复计算
        for (int i = 0; i < count; i++)
        {
            RectTransform child = rectChildren[i];
            float angle = (startAngle + angleStep * i) * Mathf.Deg2Rad;
            float x = centerX + Mathf.Cos(angle) * radius - child.rect.width * 0.5f;
            SetChildAlongAxis(child, 0, x, child.rect.width);
        }
    }

    public override void SetLayoutVertical()
    {
        for (int i = 0; i < count; i++)
        {
            RectTransform child = rectChildren[i];
            float angle = (startAngle + angleStep * i) * Mathf.Deg2Rad;
            float y = centerY - Mathf.Sin(angle) * radius - child.rect.height * 0.5f;
            SetChildAlongAxis(child, 1, y, child.rect.height);
        }
    }
}
```

关键点：
- 不是直接操作 `Transform.position`，而是通过 `SetChildAlongAxis()` 在 RectTransform 布局体系中修改 UI
- `SetChildAlongAxis` 的 `pos` 参数相对于父容器左/上边界，而三角函数算出的坐标相对于父容器中心，因此必须加上 `centerX`/`centerY` 偏移
- 水平和垂直计算拆入各自的 `SetLayout` 方法，利用 LayoutRebuilder 先水平、后垂直的两轮调用顺序，避免在同一个方法中重复计算两个方向

### 10.6.5 自定义布局的本质

自定义布局不是脱离 UGUI 的独立系统，它仍然运行在 Layout Rebuild 生命周期、RectTransform 驱动体系以及 Canvas 更新流程中。它改变的只是 UI 元素的排列规则——用数学规则替代传统线性规则。

自定义布局的性能责任由开发者自己承担：需要控制 Dirty 标记频率，避免不必要的重复布局计算和复杂数学运算。

---

## 10.7 ScrollRect 深度解析

### 10.7.1 基本结构

一个标准 ScrollRect 由四个核心部分组成：

| 部分 | 角色 |
|------|------|
| **Viewport** | 可视区域，带有 RectMask2D 或 Mask 用于裁剪 |
| **Content** | 承载 UI 元素的容器，其 RectTransform 随滚动偏移变化 |
| **Scrollbar** | 滚动条，提供视觉反馈（不参与布局计算） |
| **ScrollRect 组件** | 控制中心，协调输入、位移与内容更新 |

### 10.7.2 核心驱动机制

ScrollRect 的本质不是"滚动"，而是**持续修改 Content 的 RectTransform**。当用户拖动或滚轮输入时，ScrollRect 计算 `normalizedPosition` 或 `anchoredPosition` 的变化量，应用到 Content 的 `anchoredPosition` 上。因此 ScrollRect 本质上是"基于 RectTransform 位移控制的视图窗口系统"。

### 10.7.3 ScrollRect 与 Layout 的关系

典型结构：Content 使用 VerticalLayoutGroup 自动排列子节点，ScrollRect 控制 Content 的位置变化。

LayoutGroup 计算 Content 的总尺寸 → ScrollRect 控制 Content 在 Viewport 中的显示位置 → Content 尺寸变化影响 ScrollRect 的滚动范围 → 形成 Layout 与 ScrollRect 之间的双向依赖。

### 10.7.4 ContentSizeFitter + ScrollRect 的问题

```
Content: VerticalLayoutGroup + ContentSizeFitter
ScrollRect: 依赖 Content 高度计算滚动范围
```

链路：子节点变化 → Layout 更新 → ContentSizeFitter 调整 Content 尺寸 → ScrollRect 更新滚动范围 → 可能触发 Layout 重新计算。控制不当会产生 Layout 抖动或持续 Rebuild。

### 10.7.5 视口裁剪机制

Viewport 依赖 **RectMask2D** 或 **Mask** 进行区域限制：
- **RectMask2D**：基于 UI 裁剪矩形计算像素可见性（性能更好，仅限矩形区域）
- **Mask**：基于模板缓冲区（Stencil）进行像素级裁剪（支持任意形状，但增加 DrawCall）

ScrollRect 并不真正"隐藏"内容——超出 Viewport 的部分在渲染阶段被裁剪掉。

### 10.7.6 性能瓶颈

ScrollRect 的性能问题通常不来自"滚动本身"，而来自 Content 内部结构：

- LayoutGroup 嵌套过深导致 Rebuild 成本过高
- 大量 Text 更新触发布局重新计算
- ContentSizeFitter 与 LayoutGroup 形成循环依赖
- Content 规模大时 RectTransform 更新的 CPU 开销

ScrollRect 内部采用惰性更新策略（只在输入或内容变化时重新计算偏移），但如果 Content 内部 Layout 过于复杂，Layout Rebuild 本身就是瓶颈。

---

## 10.8 大规模列表优化

### 10.8.1 问题的本质

大规模列表（背包、排行榜、聊天记录等）的性能瓶颈通常不是来自"元素数量本身"，而是来自 **Layout 系统的递归计算范围**。

当列表使用 VerticalLayoutGroup / GridLayoutGroup 时，每个新增或修改的元素都触发 Layout Rebuild，递归遍历整棵 Content 树重新计算所有子节点位置。叠加 ContentSizeFitter 或动态 Text 后，Rebuild 范围进一步放大。

### 10.8.2 对象池（Object Pool）

最基础的优化：复用已有 UI 节点，避免频繁 Instantiate / Destroy 带来的 GC 和 Layout 压力。列表项被缓存在对象池中，滚动时只更新数据而不重新创建 UI。

### 10.8.3 虚拟列表（Virtualized List）

规模更大时需要虚拟列表：**只渲染当前可视区域内的 UI 元素**。1000 条数据、屏幕只显示 10 条时，实际只需创建 10-15 个 UI 节点，通过复用和偏移计算模拟完整列表的滚动效果。

ScrollRect 不再依赖完整 Content，而是通过偏移计算 + 数据绑定实现"视觉上的完整列表"。

### 10.8.4 绕过 Layout 系统

大规模列表中 Layout 系统往往是主要瓶颈。常见做法：

- **用脚本直接计算 RectTransform 位置**，替代 LayoutGroup 自动排列
- **预计算或固定 item 高度**，避免 ContentSizeFitter 参与实时计算
- 本质上用"手动布局"替代"自动布局"，以灵活性换性能

### 10.8.5 局部更新与 Canvas 优化

- ScrollRect 滚动时只更新当前可视区域内的 item，不触发整棵 Content 的 Layout 更新
- 通过缓存滚动偏移量与可视索引范围，减少不必要的 RectTransform 变更
- 拆分静态 UI 和动态 UI 到不同 Canvas，将频繁变化的列表从主 Canvas 中隔离
- 减少动态 Text 更新频率，使用固定尺寸容器防止文本变化引发布局连锁反应

---

## 本章总结

Layout 系统是 UGUI 的"空间计算层"——它**不参与渲染，不生成像素**，而是通过持续驱动 RectTransform，为 Graphic 系统提供稳定可预测的布局结果。

### 核心认知

| 维度 | 要点 |
|------|------|
| **架构模型** | "父级驱动 + 子级反馈"混合模型，ILLayoutElement 提供尺寸，ILayoutController 执行分配 |
| **执行时机** | CanvasUpdateRegistry 统一调度的 Layout Rebuild 阶段 |
| **最终输出** | 所有计算结果转化为 RectTransform 的 `anchoredPosition` / `sizeDelta` 修改 |
| **与渲染的关系** | Layout 与渲染分层独立——Layout 负责空间计算，渲染负责像素输出 |

### 关键组件关系

```
LayoutGroup（父 → 子：排列规则 + 空间分配）
ContentSizeFitter（子 → 父：内容反向驱动容器尺寸）
DrivenRectTransformTracker（标记被驱动的属性，防止手动冲突）
ScrollRect（视图窗口：Content 位移 + Viewport 裁剪）
```

### 性能核心原则

1. **保持单向驱动**：避免父子同时使用 ContentSizeFitter，减少循环依赖风险
2. **控制嵌套深度**：Layout 是递归系统，嵌套越深，Rebuild 传播范围越大
3. **大规模列表去 Layout 化**：用脚本计算 + 对象池 + 虚拟列表替代实时 Layout 计算
4. **分离静态与动态**：不同更新频率的 UI 放到不同 Canvas

---

## 勘误汇总

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🟡 中等 | 10.1.5 | LayoutRebuilder 是"调度中心"，统一遍历所有待重建节点 | `CanvasUpdateRegistry` 负责统一调度；LayoutRebuilder 只负责单个节点的 Rebuild |
| 2 | 🟡 中等 | 10.3.1 | ContentSizeFitter "读取当前节点上的 Layout 尺寸信息" | ContentSizeFitter 通过 `LayoutUtility` 实时遍历 ILayoutElement 计算尺寸，不是"读取已有信息" |
| 3 | 🟡 中等 | 10.4.6 | "某些 LayoutGroup 会自动强制修改 Anchor" | UGUI 内置 LayoutGroup 和 ContentSizeFitter 都不修改 Anchor，仅通过 `SetChildAlongAxis()` 修改 `anchoredPosition` 和 `sizeDelta` |
| 4 | 🟢 轻微 | 10.2.3 | "内部通常会维护一个 rectChildren 列表" | rectChildren 是 LayoutGroup 的 protected 属性，通过 `GetRectChildren()` 每帧刷新，用"通常会维护"描述过于模糊 |
| 5 | 🟡 中等 | 10.6.4 | 环形布局代码中 `SetChildAlongAxis` 直接使用 `x` 和 `-y` 作为位置 | 三角函数算出的坐标相对于父容器中心，而 `SetChildAlongAxis` 的 `pos` 参数相对于父容器左/上边界，缺失 `centerX`/`centerY` 偏移，导致圆环定位到容器左上角附近 |
| 6 | 🟡 中等 | 10.6.4 | `SetLayoutHorizontal()` 和 `SetLayoutVertical()` 都调用同一个 `ArrangeChildren()` 全量计算 | LayoutRebuilder 分两轮先后调用这两个方法，每次全量计算意味着每帧多算了一遍；应将水平和垂直计算分别放入各自方法中，复用已计算的公共数据（角度、圆心坐标等） |
