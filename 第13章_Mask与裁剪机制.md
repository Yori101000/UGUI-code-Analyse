# 第13章 Mask 与裁剪机制

> 本文是对原文第 13、14 章的合并重写——原文将 Mask 和 RectMask2D 分两章但共用大量重复的 Stencil 讲解内容（"写入+测试"机制在 13.1、13.2、13.3 中重复三次），且第 14 章 RectMask2D 独立成章后与第 13 章的性能对比节大量重叠。现合并为一章，按 Mask → RectMask2D → 性能对比 → 选择指南的结构组织。补充了原文缺失的 `IMaskable` 接口机制和 `StencilMaterial.Add()` 源码分析。对照 UGUI 源码修正了 7 处错误/不准确表述。

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

**① 裁剪矩形计算**：

在 Canvas 重建阶段，RectMask2D 计算自身 RectTransform 的有效裁剪区域：

```
RectTransform 四个角的世界坐标
  → 转换到 Canvas 的局部空间
    → 计算包围盒（axis-aligned bounding box）
      → 与父级 RectMask2D 的裁剪区域求交集（如有）
        → 得到最终裁剪矩形
```

如果存在多层 RectMask2D 嵌套，每层都与父级裁剪区域求交——有效裁剪区域只会越来越小。

**② CPU 端 —— ClipperRegistry.Cull()**：

对于每个受影响的子 Graphic（它们都实现了 `IClippable` 接口）：

- **Graphic 完全在矩形外** → 调用 `IClippable.Cull()` → 内部设置 `canvasRenderer.cull = true`，不提交顶点数据
- **Graphic 与矩形有交集** → 调用 `IClippable.SetClipRect()` → 内部调用 `canvasRenderer.EnableRectClipping(clipRect)`

> **关键**：RectMask2D 并不直接操作 CanvasRenderer，而是通过 `IClippable` 接口与子 Graphic（继承自 `MaskableGraphic`）通信。`MaskableGraphic` 实现了 `IClippable.SetClipRect()` 和 `IClippable.Cull()`，在其内部才真正调用 CanvasRenderer 的对应方法。这一层间接关系使裁剪逻辑与 Graphic 的材质管理（Stencil 状态）保持一致。

**③ GPU 端**：

被设置了裁剪矩形的 Graphic，GPU 在光栅化阶段只在矩形范围内输出片元。具体实现取决于图形后端——通常映射为硬件 Scissor Test（裁剪测试），其成本远低于 Stencil 的逐像素读写。

**④ Transform 变化时的重新计算**：

RectMask2D 不是一次性计算。当以下事件发生时，裁剪矩形会在下一帧的 `Canvas.BuildBatch()` 前重新计算：

- RectTransform 尺寸或位置变化
- 父级 RectTransform 变化
- Canvas 缩放变化

这是通过 `RectMask2D.OnRectTransformDimensionsChange()` 和 `SetDirty()` 触发 `ClipperRegistry` 重新调度实现的。在频繁滚动的列表中，这个 CPU 成本持续存在——但通常远小于 Stencil Mask 带来的 GPU 开销。

**⑤ 对子节点的影响**：

RectMask2D **不会修改子节点的顶点数据，不会裁剪 Mesh**。子节点仍然生成完整的几何结构——顶点数量不变、布局不变、mesh 不变。唯一的变化是片元输出阶段被裁剪矩形限制。这意味着：

- ✅ 顶点处理成本不增加（与无裁剪时相同）
- ✅ UI 布局不受影响
- ✅ Batch 结构通常可以保持（材质未变）
- ✅ 片元填充区域减少（超出矩形的像素被丢弃）

### 13.2.2 关键代码逻辑

RectMask2D 维护的 `m_ClipTargets` 是 `List<IClippable>`，不是 CanvasRenderer 列表。裁剪操作通过接口调用传递：

```csharp
// RectMask2D.PerformClipping() 的核心逻辑（简化）
private void PerformClipping()
{
    Rect clipRect = /* 从 RectTransform 计算屏幕空间矩形 */;
    
    for (int i = 0; i < m_ClipTargets.Count; ++i)
    {
        IClippable target = m_ClipTargets[i];
        Rect targetBounds = /* target 的包围盒 */;
        
        bool cull = !clipRect.Overlaps(targetBounds);
        if (cull)
            target.Cull(clipRect, true);       // IClippable.Cull() → canvasRenderer.cull = true
        else
            target.SetClipRect(clipRect, true); // IClippable.SetClipRect() → canvasRenderer.EnableRectClipping(clipRect)
    }
}
```

`IClippable` 由 `MaskableGraphic` 实现（Image、Text 等都继承自它）：

```csharp
// MaskableGraphic 内部的 IClippable 实现
public virtual void SetClipRect(Rect value, bool validRect)
{
    if (validRect)
        canvasRenderer.EnableRectClipping(value);   // 生效：启用矩形裁剪
    else
        canvasRenderer.DisableRectClipping();       // 禁用：恢复完整渲染
}

public virtual void Cull(Rect clipRect, bool validRect)
{
    bool cull = !validRect || /* 包围盒与 clipRect 无交集 */;
    canvasRenderer.cull = cull;  // 完全在裁剪区域外 → 不提交顶点
}
```

**总结调用链**：

```
RectMask2D.PerformClipping()
  → IClippable.Cull(clipRect, true)         // 完全在外：剔除顶点
       → canvasRenderer.cull = true
  
  → IClippable.SetClipRect(clipRect, true)  // 有交集：限制像素输出
       → canvasRenderer.EnableRectClipping(clipRect)
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

### 13.2.4 RectMask2D 的局限性

- **只能做矩形裁剪**——圆形头像、不规则遮罩、Alpha 通道裁剪全部无法支持
- **旋转的 RectTransform 裁剪不精确**——裁剪矩形是轴对齐包围盒（AABB），旋转后实际形状可能是菱形或不规则四边形，但裁剪区域始终是矩形
- **不同 Canvas 之间无法跨 Canvas 裁剪**——RectMask2D 仅影响同一 Canvas 内的子 Graphic
- **不会减少 Overdraw**——超出裁剪区域的片元着色器照常执行，只在输出阶段被丢弃（与 Mask 的 Stencil 测试在此问题上是相同的）

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

### 13.3.3 全维度性能对比

| 维度 | Mask | RectMask2D |
|------|------|------|
| **DrawCall** | 至少 +1（Mask Image）+ 内部按 Stencil 状态分裂 | 通常不增加 |
| **CPU** | 低（一次性材质创建/缓存） | 中等（`ClipperRegistry.Cull()` 每帧计算裁剪矩形） |
| **GPU 片元** | 高（Stencil 读写 + 比较，逐像素执行） | 极低（硬件 Scissor Test，固定功能管线） |
| **FillRate** | 差——被遮挡像素跑完片元着色器后才做 Stencil 测试 | 同左——裁剪测试也在着色器之后 |
| **内存** | 额外材质实例 + GPU Stencil Buffer | 无额外分配 |
| **嵌套表现** | Batch 按深度分裂，越深越差 | 仅做矩形求交，性能稳定 |
| **表达能力** | 任意形状 | 仅矩形 |

**CPU vs GPU 的取舍**：

RectMask2D 的 CPU 开销高于 Mask（每帧计算裁剪矩形），但 GPU 开销远低于 Mask（无 Stencil 读写）。在大多数场景中，GPU 是瓶颈——尤其是移动端——所以 RectMask2D 的 CPU 代价通常是值得的。

**关于 FillRate 的澄清**：

> ⚠ Mask 和 RectMask2D 在被遮挡像素的处理上是**相同的**——被裁剪的像素仍然执行片元着色器，裁剪（Stencil 或 Scissor）发生在着色器之后。两者都不会减少 Overdraw。减少 Overdraw 需要靠减少 UI 元素重叠，裁剪系统解决的是"可见性"而非"着色器执行次数"。

**移动平台的额外考量**：

在移动 GPU 上，Mask 的压力被进一步放大：
- 移动 GPU 的 FillRate 和带宽天然较低
- Stencil Buffer 在部分移动架构上与 Depth Buffer 共享存储，读写竞争更激烈
- 多重采样（MSAA）下 Stencil 成本随采样点数量等比例放大
- 全屏 UI + 大量半透明元素 + Stencil Mask = 移动端掉帧的高发组合

因此移动项目的经验法则是：**能用 RectMask2D 就不用 Mask，能减少嵌套就减少嵌套，能避免 Stencil 就避免 Stencil。**

### 13.3.4 嵌套的代价

- **嵌套 Mask**：每层创建一个新 Stencil 状态材质 → 材质实例数 = 嵌套深度 → Batch 按深度分裂。实际开发应严格控制深度，通常不超过 2 层。
- **嵌套 RectMask2D**：每个 RectMask2D 独立执行裁剪，子 RectMask2D 的有效裁剪区域是自身矩形与父裁剪矩形的交集。

---

## 13.4 实战选择指南

### 决策树

```
需要不规则形状遮罩（圆形、异形、Alpha 通道）？
  ├── 是 → Mask
  │      场景：圆形头像、异形按钮、动态镂空、UI 擦除、渐变遮罩
  │      代价：额外 DrawCall + 材质实例
  │      优化：用简单 Graphic 做遮罩（减少 Mask Image 顶点数）
  │      Trick：Mask Image 的 color.a = 0 → 遮罩不可见但 Stencil 正常工作
  │      注意：Mask 裁剪掉的区域仍可被点击（见第 11 章勘误）
  │
  └── 否 → RectMask2D
           场景：ScrollView、聊天/背包/排行榜列表、窗口裁剪
           注意：裁剪区域由 RectTransform 决定，不支持旋转形状
```

### 按场景选型速查

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| ScrollView 内容裁剪 | RectMask2D | 列表 Item 数量大，必须保持合批 |
| 圆形头像 | Mask | 形状不规则，RectMask2D 做不到 |
| 聊天列表 | RectMask2D | 规则矩形 + 大量 Item + 频繁滚动 |
| 异形弹窗 | Mask | 需要 Alpha 通道决定裁剪形状 |
| 背包/仓库格子 | RectMask2D | 规则矩形，性能优先 |
| UI 擦除/过渡动画 | Mask | 需要动态变化的像素级遮罩 |
| 多层嵌套裁剪 | RectMask2D（优先） | 嵌套 Mask 的 Batch 分裂不可控 |

### 项目规范参考

成熟 UI 项目常见的硬性规范：

- **禁止在 ScrollView 中使用 Mask**——滚动列表必须用 RectMask2D
- **限制 Mask 嵌套深度 ≤ 2 层**
- **圆形头像用 RectMask2D + 圆角 Shader 替代 Mask**（部分项目）
- **复杂视觉效果才允许使用 Mask**，需经性能评审

### 编辑器与真机的差异

PC 编辑器 GPU 性能充足 → Stencil Mask 开销几乎无感 → 容易误判。在移动设备上，同样 UI 的 DrawCall 增长和 FillRate 下降会明显得多。**所有裁剪相关的性能决策必须以目标设备 Profiler 数据为准，不能以编辑器运行效果为准。**

### 常见问题速查

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
│   ├── 代价：新材质实例 → 断批、GPU Stencil 读写、移动端 FillRate 压力
│   └── 陷阱：被裁剪的像素仍可被 Raycast 检测到
│
└── RectMask2D（矩形裁剪）
    ├── 机制：裁剪矩形计算 → IClippable.Cull() / SetClipRect() → CanvasRenderer.EnableRectClipping()
    ├── 优点：不断批、GPU 开销极低（硬件 Scissor）、移动端友好
    ├── 代价：每帧 CPU 裁剪计算、旋转时裁剪不精确、不支持不规则形状
    └── 陷阱：编辑器运行流畅 ≠ 真机运行流畅
```

两者不是替代关系，是互补关系。选型的核心判断只有一个：**是否需要不规则形状？** 需要 → Mask，不需要 → RectMask2D。

本章与第 9 章（渲染管线）、第 11 章（EventSystem 点击检测）、第 12 章（图集/合批）之间的关系：渲染管线决定了 UI 的顶点生成和 Batch 构建顺序；图集解决的是 Texture 统一（减少纹理切换）；EventSystem 的 Raycast 检测不受 Mask（Stencil）影响但受 RectMask2D（cull）影响；Mask/RectMask2D 则在合批基础上引入了额外的渲染状态条件（Stencil 状态或裁剪矩形）。

---

## 勘误汇总

| # | 严重程度 | 原文章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 严重 | 13.1.1 | "Mask 的作用范围是基于 Canvas 渲染层级的，而不是 Transform 空间意义上的父子关系" | Mask 作用范围精确基于 Transform 层级。`MaskUtilities.GetStencilDepth()` 逐级遍历 `transform.parent` 查找 Mask 祖先 |
| 2 | 🟡 中等 | 13.1.1 | "Mask 自身所在的 UI 节点，该节点负责定义裁剪区域的形状" | Mask 组件本身不渲染任何东西——渲染遮罩形状的是同 GameObject 上的 Graphic。没有 Graphic 的 Mask 无效 |
| 3 | 🟡 中等 | 13.3.6 | RectMask2D "发生在 CPU 剔除阶段，基于包围盒判断" | 不准确——RectMask2D 同时做 CPU 剔除（完全在外）+ GPU 裁剪（部分在内），不是单纯的"包围盒判断" |
| 4 | 🟡 中等 | 全文 | 大量讲解 Stencil 原理但完全不提 `IMaskable` 接口和 `StencilMaterial.Add()` | `IMaskable.RecalculateMasking()` 是 Mask 向子节点传递材质修改的桥梁——理解 Mask 如何生效的核心机制 |
| 5 | 🟡 中等 | 14.1.7 | "多层RectMask2D嵌套...最终裁剪区域为两者重叠部分" | 过于笼统——子 RectMask2D 的有效裁剪区域 = 自身裁剪矩形 ∩ 父裁剪矩形。嵌套时裁剪区域只会缩小不会扩大，这是关键行为约束 |
| 6 | 🟢 轻微 | 14.2 | 原文将 Mask 和 RectMask2D 分两章，但第 14 章的性能对比内容（14.2）与第 13 章高度重叠 | 两章共用大量重复的 Stencil 讲解，且 14.2 性能对比的表格与第 13 章已有的对比结构基本重复 |
| 7 | 🟢 轻微 | 14.3 | 大量使用场景小节（14.3.2~14.3.5）的核心论点重复——都是"能用 RectMask2D 就不用 Mask" | 各小节差异只在举例界面名称不同（商城/卡牌/技能树 vs 聊天/背包/排行榜），本质是同一论点的多次换皮 |
| 8 | 🟢 轻微 | 13.1.5 / 13.3.1 | "Mask 的渲染阶段位置"与"Mask 对 Canvas 渲染阶段的插入位置" | 内容高度重叠——都在描述 Mask 在渲染管线中的位置 |
