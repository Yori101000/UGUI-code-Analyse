# 第13章 Mask 与裁剪机制

> 本文是对原文的结构化重写，合并了原文 13.1、13.2、13.3 中大量重叠的 Stencil 讲解内容（"写入+测试"机制原文重复了三次）。补充了原文缺失的 `IMaskable` 接口机制和 `StencilMaterial.Add()` 源码分析。对照 UGUI 源码修正了 7 处错误/不准确表述。

---

## 概述

UGUI 提供两种裁剪机制：

| | Mask | RectMask2D |
|------|------|------|
| 底层机制 | GPU Stencil Buffer | CanvasRenderer 矩形裁剪 |
| 裁剪形状 | 任意（由 Graphic 的 Alpha 决定） | 固定矩形 |
| 断批 | 是（不同 Stencil 状态 = 不同材质实例） | 通常不断批 |
| 性能代价 | GPU 片元阶段 + 材质实例化 | CPU 剔除 + CanvasRenderer 裁剪数据 |
| 典型场景 | 圆形头像、不规则遮罩 | ScrollView、列表、规则裁剪区域 |

本章自底向上讲：先分别剖析两种机制的底层实现，再对比性能影响，最后给出选择指南。

---

## 13.1 Mask：基于 Stencil Buffer 的裁剪

### 13.1.1 Mask 的组件结构

一个有效的 Mask 需要两个部分在**同一个 GameObject** 上：

```
GameObject (带 Mask)
  ├── Mask 组件          ← 控制器：修改材质参数
  └── Image 组件         ← 渲染器：实际绘制 + 写入 Stencil（提供遮罩形状）
```

**Mask 组件本身不渲染任何像素。** 没有 Graphic（Image/RawImage）的 Mask 组件是无效的——它不会产生任何视觉效果。Mask 的遮罩形状由同 GameObject 上 Graphic 的渲染内容决定（包括其 Alpha 通道——这就是为什么圆形图片可以做出圆形遮罩）。

Mask 组件做的事情：重写同 GameObject 上 Graphic 的 `GetModifiedMaterial()`，在返回的材质中注入 Stencil 写入操作。

### 13.1.2 IMaskable：Mask 如何影响子节点

**这是原文完全缺失的一环。** 挂上 Mask 组件后，子节点的材质为什么会自动变？

答案在 `IMaskable` 接口：

```csharp
// Graphic 基类实现了 IMaskable
public interface IMaskable
{
    void RecalculateMasking();  // Mask 状态变化时被调用
}
```

整个传播链路：

```
Mask.OnEnable() / OnDisable() / OnTransformParentChanged()
  → MaskUtilities.NotifyStencilStateChanged(this)
    → 遍历 transform 子树中所有实现 IMaskable 的组件
      → 对每个 IMaskable 调用 RecalculateMasking()
        → 子 Graphic 的 GetModifiedMaterial() 被重新调用
          → 返回注入了 Stencil 测试的材质（来自 StencilMaterial.Add()）
```

**Mask 的作用范围精确遵循 Transform 层级。** `MaskUtilities.GetStencilDepth()` 通过遍历 `transform.parent` 向上查找 Mask 祖先来计算当前元素处于第几层 Mask 嵌套——这是纯粹的 Transform 树逻辑，不是"Canvas 渲染层级"决定的。

### 13.1.3 Stencil Buffer 机制：写入与测试

Stencil Buffer 是 GPU 的一块逐像素缓冲区（通常 8 位），与颜色缓冲、深度缓冲并列。每个像素位置存储一个整数值，供后续渲染做条件判断。

Mask 的裁剪分两个阶段，**都发生在 GPU 片元着色器之后**：

**阶段一：Mask 自身渲染 → 写入 Stencil 标记**

Mask GameObject 上的 Image 渲染时，其材质已被 `Mask.GetModifiedMaterial()` 修改为"写入 Stencil"模式：

```
片段着色器输出颜色
  → Stencil 测试：Always（始终通过，不检查之前的值）
    → Stencil 操作：Replace（将参考值写入 Stencil Buffer）
      → 颜色写入：正常输出颜色到帧缓冲
```

结果：Image 的**不透明像素**覆盖区域在 Stencil Buffer 中被标记为参考值（如 1）。Image 的透明区域不写入 Stencil——这就是为什么遮罩形状由 Image 的 Alpha 决定。

**阶段二：子节点渲染 → 测试 Stencil 标记**

子节点的 Graphic 渲染时，其材质已被修改为"测试 Stencil"模式：

```
片段着色器输出颜色
  → Stencil 测试：Equal（只允许 Stencil 值 == 参考值的像素通过）
    → 通过 → 写入颜色缓冲（可见）
    → 失败 → 丢弃片元（不可见）
```

结果：只有当像素位置的 Stencil 值等于 Mask 写入的参考值时，子节点的颜色才被写入帧缓冲。这就形成了"只在遮罩区域内可见"的效果。

### 13.1.4 StencilMaterial.Add()：材质的 Stencil 注入

UGUI 通过 `StencilMaterial.Add()` 创建带 Stencil 配置的材质实例：

```csharp
// Mask 自身: 写入 Stencil
Material maskMaterial = StencilMaterial.Add(
    baseMaterial,              // 原始材质（如 Default UI Material）
    (1 << stencilDepth) - 1,   // 参考值（基于嵌套深度计算）
    StencilOp.Replace,         // 写入操作：直接替换
    CompareFunction.Always,    // 测试函数：始终通过
    ColorWriteMask.All         // 同时正常输出颜色
);

// 子节点: 测试 Stencil
Material childMaterial = StencilMaterial.Add(
    baseMaterial,              // 原始材质
    (1 << stencilDepth) - 1,   // 参考值（与父 Mask 相同）
    StencilOp.Keep,            // 不修改 Stencil Buffer
    CompareFunction.Equal,     // 测试：只允许 Stencil == ref 的像素通过
    ColorWriteMask.All
);
```

**注意**：`StencilMaterial.Add()` 每次调用都会创建新的材质实例——即使传入了相同的参数。这意味着不同 Mask 层级有不同的材质实例 → 无法合批。

`MaskUtilities.GetStencilDepth()` 计算嵌套深度：

```csharp
// 向上遍历 transform.parent，统计遇到的 Mask 组件数量
public static int GetStencilDepth(Transform transform, Transform stopAfter)
{
    int depth = 0;
    while (transform != stopAfter && transform != null)
    {
        if (transform.GetComponent<Mask>() != null)
            ++depth;
        transform = transform.parent;
    }
    return depth;
}
```

### 13.1.5 嵌套 Mask

Mask 可以嵌套：

```
Mask A（深度 1，写 ref=1，子节点测 ref=1）
  └── Mask B（深度 2，写 ref=3¹，子节点测 ref=3）
       └── Image C（测 ref=3 → 必须同时通过 A 和 B 的裁剪区域）
```

¹ 注：UGUI 实际使用 `(1 << depth) - 1` 计算 Stencil ref，`depth=1 → ref=1`，`depth=2 → ref=3`。这是位掩码编码——具体细节由 `StencilMaterial.Add()` 的 `stencilID` 参数决定，各 Unity 版本可能有所不同。

每层嵌套至少创建一个新的材质实例 → Stencil 状态不同 → 打断合批。

---

## 13.2 RectMask2D：基于矩形裁剪

### 13.2.1 工作原理

RectMask2D **完全不经过 Stencil**。它的裁剪分为两个阶段：

**CPU 端 —— ClipperRegistry.Cull()**：

在 Canvas 重建阶段，RectMask2D 计算自身 RectTransform 的**屏幕空间矩形**。对于每个受其影响的子 Graphic：

- **Graphic 完全在矩形外** → 剔除：`CanvasRenderer.cull = true`，不提交顶点数据
- **Graphic 与矩形有交集** → 设置裁剪矩形：`CanvasRenderer.SetClipRect(clipRect, true)`

**GPU 端**：

被设置了裁剪矩形的 Graphic，GPU 在光栅化阶段只在矩形范围内输出片元。具体实现取决于图形后端——可能映射为 scissor rect 或 shader 中的 clip 指令。

### 13.2.2 关键代码逻辑

```csharp
// RectMask2D.PerformClipping() 的核心逻辑（简化）
private void PerformClipping()
{
    Rect clipRect = /* 从 RectTransform 计算屏幕空间矩形 */;
    
    for (int i = 0; i < m_ClipTargets.Count; ++i)
    {
        Rect targetBounds = /* Graphic 的包围盒 */;
        
        // 完全在裁剪区域外 → 剔除
        if (!clipRect.Overlaps(targetBounds))
            m_ClipTargets[i].canvasRenderer.cull = true;
        else
            // 部分在裁剪区域内 → 设置裁剪矩形
            m_ClipTargets[i].canvasRenderer.SetClipRect(clipRect, true);
    }
}
```

### 13.2.3 与 Mask 的本质区别

| 维度 | Mask | RectMask2D |
|------|------|------|
| 是否修改材质 | ✅ `StencilMaterial.Add()` 创建新材质实例 | ❌ 不修改材质 |
| 裁剪判断地点 | GPU 片元阶段（逐像素 Stencil 测试） | CPU 阶段剔除 + GPU scissor/clip |
| 对合批的影响 | 打断（不同材质实例不能合批） | 通常不断批（材质不变） |
| 裁剪形状 | 任意（由 Alpha 决定） | 只能是矩形 |
| 软遮罩（渐变边缘） | ✅（Alpha 半透明区域写入部分 Stencil） | ❌ |
| 所需依赖 | Image+Mask 组件 | 仅 RectMask2D 组件 |

---

## 13.3 裁剪对合批与性能的影响

### 13.3.1 Mask 为什么必然打断合批

合批的前提是**相同的 Material 实例**。`StencilMaterial.Add()` 为每个不同的 Stencil 状态（参考值、比较函数、写入操作）创建新的材质实例。

```
一个 Mask = 至少 1 个新材质实例（Mask 自身 Image 用）
N 层嵌套 Mask = N 个不同 Stencil 状态的材质实例

结论：
  → Mask 内外的 UI 使用不同材质 → 无法合批
  → 不同深度的 Mask 使用不同材质 → 彼此也无法合批
  → 即使使用相同的原始材质和纹理，只要 Stencil 参数不同就分属不同 Batch
```

### 13.3.2 RectMask2D 对合批的影响

RectMask2D 不修改材质——它只在 CanvasRenderer 上设置裁剪矩形：

- **同一个 RectMask2D 内的多个 Graphic**：材质相同 → 可以合批
- **RectMask2D 内和 RectMask2D 外的 Graphic**：材质相同 → 可以合批

这就是为什么 ScrollView 必须用 RectMask2D 而不是 Mask——列表中的几十个 Item 可以共享同一个 DrawCall，而 Mask 会把每个深度的元素都拆到不同的 Batch。

### 13.3.3 性能对比

| | Mask | RectMask2D |
|------|------|------|
| DrawCall | 至少 +1（Mask Image）+ 内部可能按 Stencil 状态分裂 | 通常不增加 |
| CPU | 材质创建/缓存（`StencilMaterial` 池） | `ClipperRegistry.Cull()` 每帧计算裁剪关系 |
| GPU | 片元阶段额外 Stencil 读写 | scissor/clip（硬件支持，成本极低） |
| 内存 | 额外材质实例（`StencilMaterial.Add()` 生成） | 无额外分配 |

> ⚠ **共同局限**：Mask 和 RectMask2D 都不会减少 Overdraw。被遮罩遮挡的像素仍然执行片元着色器——裁剪控制的是"是否写入帧缓冲"，不是"片元着色器是否执行"。Stencil 测试发生在片元着色器之后，所以着色器跑完了才决定丢弃。减少 Overdraw 需要靠减少 UI 元素重叠，不是靠裁剪系统。

### 13.3.4 嵌套的代价

- **嵌套 Mask**：每层创建一个新 Stencil 状态材质 → 材质实例数 = 嵌套深度 → Batch 按深度分裂。实际开发应严格控制深度，通常不超过 2 层。
- **嵌套 RectMask2D**：每个 RectMask2D 独立执行裁剪，子 RectMask2D 的有效裁剪区域是自身矩形与父裁剪矩形的交集。

---

## 13.4 实战选择指南

```
需要不规则形状遮罩？
  ├── 是 → Mask
  │      代价：额外 DrawCall + 材质实例
  │      优化：用简单 Graphic 做遮罩（减少 Mask Image 顶点数）
  │      Trick：Mask Image 的 color.a = 0 → 遮罩不可见但 Stencil 正常工作
  │
  └── 否 → RectMask2D
           场景：ScrollView、列表裁剪、规则矩形
           注意：裁剪区域由 RectTransform 决定
```

**常见问题速查**：

| 现象 | 可能原因 | 排查方向 |
|------|---------|---------|
| 加 Mask 后 DrawCall 暴涨 | Mask 把原本能合批的 UI 拆成了多个材质组 | Frame Debugger 看 Stencil 引用值变化 |
| 圆形遮罩不生效 | Mask 上的 Image 没有有效的 Alpha 通道 | 检查 Image 的 Source Image 是否有透明镂空区域 |
| RectMask2D 裁剪不了 | 子 Graphic 和 RectMask2D 不在同一个 Canvas | RectMask2D 的裁剪仅对同一 Canvas 内的 Graphic 生效 |
| 嵌套 Mask 后某些 UI 消失 | 子 UI 的 Stencil 参考值不匹配 | `GetStencilDepth()` 计算深度是否跟预期一致 |
| Mask 内 UI 和 Mask 外 UI 不同批 | 不同 Stencil 状态 = 不同材质实例 | 这是 Mask 的固有行为，不是 bug |

---

## 本章总结

```
Mask 系统
├── Mask（GPU Stencil Buffer）
│   ├── 机制：写入 → 测试（全部在 GPU 片元阶段）
│   ├── 实现：IMaskable 传播 → StencilMaterial.Add() → GetModifiedMaterial()
│   ├── 优点：任意形状（由 Image Alpha 决定）
│   └── 代价：新材质实例 → 断批
│
└── RectMask2D（矩形裁剪）
    ├── 机制：CPU 剔除 + CanvasRenderer.SetClipRect()
    ├── 优点：不断批、GPU 开销极低（硬件 scissor）
    └── 限制：只能是矩形
```

两者不是替代关系，是互补关系。选型的核心判断只有一个：**是否需要不规则形状？** 需要 → Mask，不需要 → RectMask2D。

本章与第 9 章（渲染管线）、第 12 章（图集）之间的关系：渲染管线决定了 UI 的顶点生成和 Batch 构建顺序；图集解决的是 Texture 统一问题（减少纹理切换）；Mask/RectMask2D 则是在这两个基础上引入了额外的渲染状态条件（Stencil 状态或裁剪矩形），它们是渲染状态下沉到 DrawCall 层面时的关键影响因素。

---

## 勘误汇总

| # | 严重程度 | 原文章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 严重 | 13.1.1 | "Mask 的作用范围是基于 Canvas 渲染层级的，而不是 Transform 空间意义上的父子关系" | Mask 作用范围精确基于 Transform 层级。`MaskUtilities.GetStencilDepth()` 逐级遍历 `transform.parent` 查找 Mask 祖先 |
| 2 | 🟡 中等 | 13.1.1 | "Mask 自身所在的 UI 节点，该节点负责定义裁剪区域的形状" | Mask 组件本身不渲染任何东西——渲染遮罩形状的是同 GameObject 上的 Graphic。没有 Graphic 的 Mask 无效 |
| 3 | 🟡 中等 | 13.3.6 | RectMask2D "发生在 CPU 剔除阶段，基于包围盒判断" | 不准确——RectMask2D 同时做 CPU 剔除（完全在外）+ GPU 裁剪（部分在内），不是单纯的"包围盒判断" |
| 4 | 🟡 中等 | 全文 | 大量讲解 Stencil 原理但完全不提 `IMaskable` 接口和 `StencilMaterial.Add()` | `IMaskable.RecalculateMasking()` 是 Mask 向子节点传递材质修改的桥梁——理解 Mask 如何生效的核心机制 |
| 5 | 🟢 轻微 | 13.1.2 / 13.2.2 / 13.2.3 | "Mask 的工作方式：从'写入'到'测试'"与"Stencil 的写入过程""Stencil 的测试过程" | 三节大量重叠——"写入+测试"机制讲了三次，只精细度不同 |
| 6 | 🟢 轻微 | 13.1.4 / 13.2.4 / 13.3.4 | 嵌套 Mask 的 Stencil 状态管理 | 同一主题分散在三处，信息重复 |
| 7 | 🟢 轻微 | 13.1.5 / 13.3.1 | "Mask 的渲染阶段位置"与"Mask 对 Canvas 渲染阶段的插入位置" | 内容高度重叠——都在描述 Mask 在渲染管线中的位置 |
