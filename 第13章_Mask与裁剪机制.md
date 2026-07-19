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

> 如果你已经理解 UGUI 渲染管线（第 9 章：Graphic → CanvasRenderer → Canvas.BuildBatch → GPU），这一节可以在已有知识上直接建立 Mask 的心智模型。

### 13.1.0 从一个具体问题出发

你在 Unity 里给一个 GameObject 挂上 Mask 组件，子节点是一张超出父节点边界的图片。运行后，超出部分看不到了。

Mask 既不砍顶点（对比 RectMask2D 干掉整个元素的顶点），也不改布局。它做的是更底层的事：**在 GPU 里给每个像素打标记，后续渲染子节点时检查这个标记——标记不对的像素直接扔掉。**

要理解这个过程，首先得知道 GPU 里有一块叫 Stencil Buffer 的东西。

### 13.1.1 Stencil Buffer：GPU 给你的草稿纸

GPU 渲染时维护着几块"画布"，每块画布的每个像素位置都存着数据：

| 画布 | 每个像素存什么 | 用途 |
|------|-------------|------|
| Color Buffer | RGBA 颜色值 | 最终显示在屏幕上的画面 |
| Depth Buffer | 深度值（离相机多远） | 判断哪个物体在前面 |
| **Stencil Buffer** | 一个整数（0-255） | **你想存什么就存什么，GPU 不关心含义** |

**Stencil Buffer 就是 GPU 给你的一张草稿纸。** 每个像素对应一个格子，你可以往格子里写数字，之后读这个数字来做判断。它不影响颜色，不影响深度——纯粹给你做标记用的。

类比你用一张镂空卡片盖在纸上画画：只往镂空的地方涂色，被卡片挡住的地方不涂。Stencil Buffer 就是这张卡片——但不是物理盖住，而是每画一笔之前，GPU 先检查"这个像素在我的标记范围内吗？"。

### 13.1.2 Mask 需要两个组件在同一个 GameObject 上

```
GameObject (带 Mask)
  ├── Mask 组件          ← 控制器：修改自己和子节点的材质
  └── Image 组件         ← 渲染器：实际绘制遮罩形状 + 写入 Stencil 标记
```

**Mask 组件本身不渲染任何像素。** 渲染遮罩形状的是同 GameObject 上的 Graphic（通常是 Image）。Mask 组件做的是：在这两个组件（自己和子节点）渲染之前，替换它们用的材质。

遮罩形状由 Image 的 Alpha 通道决定——圆形图片的圆形区域不透明→写 Stencil，四个角透明→不写 Stencil。这就是为什么圆形图片能做出圆形遮罩。没有 Graphic 的 Mask 是无效的（没有形状可渲染，就没有东西写 Stencil）。

### 13.1.3 核心机制：先写标记，再检查

Mask 的工作分两步，按 GPU 实际执行顺序来理解。

#### 阶段一：渲染 Mask 本体 → 顺便写标记

Mask 上的 Image 走正常的渲染流程：Graphic.OnPopulateMesh() → CanvasRenderer.SetMesh() → Canvas.BuildBatch() → 提交 GPU。

但在 GPU 渲染这个 Image 时，它的材质已经被 Mask 替换成特殊版本，带了这样一条指令：

> **"画这个像素的同时，往 Stencil Buffer 的同一位置写一个 1。"**

执行过程（在 GPU 片元着色器之后）：

```
片段着色器输出 Image 像素颜色
  → Stencil 测试：Always（始终通过，不检查旧值）
    → Stencil 操作：Replace（把参考值 1 写入 Stencil Buffer 对应位置）
      → 颜色写入：正常输出到 Color Buffer
```

Image 渲染完成后：
- Color Buffer 里有了 Image 的颜色（`showMaskGraphic = true` 能看到；`false` 它全透明但标记照样写了）
- **Stencil Buffer 里，Image 不透明像素覆盖的位置被标记为 1，透明像素位置还是 0**

#### 阶段二：渲染子节点 → 检查标记

Mask 下面的子节点（Button、Text、各种 Image）开始渲染。它们的材质也被 Mask 替换成另一个特殊版本：

> **"画这个像素之前，先检查 Stencil Buffer 同一位置的值是不是 1。是 → 正常画。不是 → 丢弃。"**

执行过程：

```
片段着色器输出子节点像素颜色
  → Stencil 测试：Equal（要求 Stencil 值 == 参考值 1）
    → 通过 → 写入 Color Buffer（可见 ✅）
    → 失败 → 丢弃片元（不可见 ❌）
```

结果：子节点像素位置 Stencil=1（被 Mask Image 覆盖过）→ 正常显示。Stencil=0（在 Mask Image 范围外）→ GPU 扔掉了，屏幕上看不到。

### 13.1.4 一个具体帧的完整例子

假设场景结构：
```
Mask（挂 Mask 组件 + 圆形 Alpha 图）
  └── Background（Image，一张大地图）
```

这一帧发生了什么：

1. **Canvas 收集**需要渲染的 Graphic：Mask 上的 Image、子节点 Background Image
2. **Canvas 排序**：Mask Image 先渲染，Background 后渲染（同一层级下按深度排列）
3. **渲染 Mask Image**：片元着色器输出每个像素颜色 → Stencil Replace，圆形区域内 Stencil 写为 1，圆形外透明像素不写 → Stencil 保持 0 → 正常输出颜色到 Color Buffer（如果 showMaskGraphic=true）
4. **渲染 Background**：片元着色器算出每个像素颜色 → Stencil Equal 测试 → 圆形区域内 Stencil=1 通过，写入 Color Buffer → 圆形外 Stencil=0 失败，丢弃
5. **最终屏幕**：Background 只在圆形区域内可见

### 13.1.5 Mask 组件在 CPU 侧具体做了什么

上面讲的是 GPU 行为。Mask 组件在 CPU 侧做了三件事来让这一切发生：

#### ① 通知子节点"你被遮罩了"

```
Mask.OnEnable() / OnDisable() / OnTransformParentChanged()
  → MaskUtilities.NotifyStencilStateChanged(this)
    → 遍历 transform 子树中所有实现 IMaskable 的组件
      → 每个 IMaskable.RecalculateMasking()
        → 子 Graphic 内部标记"遮罩状态变了，我需要新材质"
```

`IMaskable` 是 Mask 和子节点之间的通信接口。Image、Text、RawImage 都实现了它。

#### ② 替换自己的材质（Mask 本体）→ 注入"写入 Stencil"指令

Mask 重写了同 GameObject 上 Graphic 的 `GetModifiedMaterial()`。当这个 Graphic 要渲染时，拿到的不是默认 UI 材质，而是 `StencilMaterial.Add()` 生成的新材质：

```csharp
// Mask 自身用的是"写入 Stencil"材质
StencilMaterial.Add(
    baseMaterial,            // 原始材质（Default UI Material）
    stencilRef,              // 要写入的参考值（如 1）
    StencilOp.Replace,       // 写入操作：替换
    CompareFunction.Always,  // 测试：始终通过
    ColorWriteMask.All       // 正常输出颜色
);
```

#### ③ 替换子节点的材质 → 注入"测试 Stencil"指令

通过 IMaskable → MaskUtilities 链路，子 Graphic 的 `GetModifiedMaterial()` 被重新调用，返回另一个 `StencilMaterial.Add()` 生成的材质：

```csharp
// 子节点用的是"测试 Stencil"材质
StencilMaterial.Add(
    baseMaterial,            // 原始材质
    stencilRef,              // 测试参考值（与父 Mask 写入的值相同）
    StencilOp.Keep,          // 不修改 Stencil Buffer
    CompareFunction.Equal,   // 测试：只有 Stencil == ref 才通过
    ColorWriteMask.All
);
```

#### Mask 的作用范围：Transform 层级决定

`MaskUtilities.GetStencilDepth(transform, stopAfter)` 通过遍历 `transform.parent` 向上查找 Mask 祖先来计算嵌套深度。这是纯粹的 Transform 树逻辑，不是"Canvas 渲染层级"——原文此处有严重错误。

### 13.1.6 嵌套 Mask

Mask 可以嵌套。每次嵌套，深度 +1，Stencil 参考值随之变化：

```
Mask A（深度 1，写 ref=1，子节点测 ref=1）
  └── Mask B（深度 2，写 ref=3¹，子节点测 ref=3）
       └── Image C（测 ref=3 → 必须同时通过 A 和 B 的裁剪区域）
```

¹ `(1 << depth) - 1` 计算 Stencil ref：depth=1 → ref=1，depth=2 → ref=3。这是位掩码编码，具体由 `StencilMaterial.Add()` 的 `stencilID` 参数决定，各 Unity 版本可能有所不同。

嵌套的效果：**每一层都在收紧条件——Image C 最终只显示在 A ∩ B 的交集区域。**

每层嵌套至少创建一个新的材质实例 → Stencil 状态不同 → 打断合批。

### 13.1.7 为什么 Mask 会打断合批（以及为什么 RectMask2D 不会）

UGUI 合批的条件是"使用完全相同的 Material 实例"（同一个 C# 对象引用，不是"相似的"材质）。

**`StencilMaterial.Add()` 有内部缓存**——它用 `(baseMaterial, stencilID, operation, compareFunction, colorWriteMask, readMask, writeMask)` 做 key。相同参数第二次调用时直接返回缓存中的实例，**不会重复创建**。因此同一 Mask 层级下的所有子 Graphic（Stencil 参数完全相同）共享同一个材质实例，**它们之间仍然可以合批**。

Mask 断批的真正原因不在"每个 Graphic 一个实例"，而在"不同 Mask 层级的 Stencil 参数不同"：

```
正常 UI（默认材质实例）                    → Batch 1
Mask A 自己的 Image（Replace ref=1）       → Batch 2（材质实例 ≠ Batch 1）
  |── 100 个子 Graphic（Equal ref=1）      → Batch 3（共享同一个实例，内部可以合批 ✅）
  |
  Mask B 自己的 Image（Replace ref=3）     → Batch 4（材质实例 ≠ Batch 3）
     |── 50 个子 Graphic（Equal ref=3）    → Batch 5（共享同一个实例，内部可以合批 ✅）
```

**关键结论**：Mask 增加的 Batch 数 = 不同 Stencil 参数组合的数量（嵌套深度 × 2），不是子 Graphic 的数量。一层 Mask 把 Batch 从 1 变成 3（自身 + 子节点群 + 外部 UI），但子节点群内部 100 个元素仍然可以共享 1 个 Batch。

而 RectMask2D 完全不修改材质——它通过 CanvasRenderer 的裁剪矩形实现，所有 Graphic 保持原始材质实例不变 → 可以跨裁剪边界合批。这是两者最本质的差异。

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

合批的前提是**相同的 Material 实例**。`StencilMaterial.Add()` 内部有缓存：**相同 Stencil 参数返回同一实例，不同参数才创建新实例。**

```
Mask 自身 Image（Replace + ref=N）    → 1 个新材质实例
Mask 所有直属子 Graphic（Equal + ref=N）→ 1 个新材质实例（但共享，内部可合批）
N 层嵌套 Mask                         → 2N 个不同 Stencil 参数组合的材质实例

结论：
  → Mask 内外的 UI 使用不同材质 → 无法跨边界合批
  → 不同深度的 Mask 有不同的 ref → 彼此无法合批
  → 同一深度内的子 Graphic 共享材质 → 可以合批 ✅
  → Batch 数 = 不同 Stencil 参数组合数，不是 Graphic 数量
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
