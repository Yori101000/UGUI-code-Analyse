# 第17章 UI Mesh 扩展机制

> 本文是对原文第 16 章的结构化重写，合并了原文 16.1、16.2、16.4 中重复的链式流程描述（"OnPopulateMesh→ModifyMesh→SetMesh"同一链路讲了三次）。补充了原文缺失的 GC 分配问题和 `UIVertex` 结构体大小说明。对照 UGUI 源码修正了 5 处错误/遗漏。

---

## 概述

UGUI 的渲染管线允许开发者在 Mesh 提交 GPU 之前介入修改顶点数据。这一机制打破了"UI 只能是矩形"的限制——描边、阴影、渐变、波浪、扭曲、流光等效果，底层都是通过修改顶点数据实现的。

核心路径：

```
Graphic 重建
  → OnPopulateMesh()（生成原始顶点）
    → [ModifyMesh 链]（叠加多个顶点特效）
      → CanvasRenderer.SetMesh() → Canvas.BuildBatch() → GPU
```

---

## 17.1 ModifyMesh：顶点后处理入口

### 17.1.1 完整链路的执行顺序

对于任意一个 Graphic（Image、Text 等），重建时的完整流程：

```
Graphic.Rebuild()
  → OnPopulateMesh(VertexHelper)         ← 生成原始顶点
    → GetComponents<IMeshModifier>()      ← 获取所有 Mesh 特效组件
      → 特效 A.ModifyMesh(vh)            ← 修改顶点
        → 特效 B.ModifyMesh(vh)          ← 修改已修改过的顶点
          → ...                          ← 链式传递
            → CanvasRenderer.SetMesh(mesh) ← 提交最终 Mesh
```

**关键点**：所有特效共用同一个 `VertexHelper` 实例。前一个特效的输出就是后一个特效的输入。特效的执行顺序 = Inspector 中组件的排列顺序。

### 17.1.2 IMeshModifier 接口与 BaseMeshEffect 基类

ModifyMesh 的底层来自 `IMeshModifier` 接口：

```csharp
public interface IMeshModifier
{
    void ModifyMesh(VertexHelper vh);
}
```

Unity 推荐继承 `BaseMeshEffect` 而不是直接实现接口：

```csharp
public abstract class BaseMeshEffect : UIBehaviour, IMeshModifier
{
    public abstract void ModifyMesh(VertexHelper vh);

    // 内部自动处理了：
    // - OnEnable/OnDisable 时通知 Graphic 标记 Dirty
    // - 编辑器下自动刷新
}
```

**重要**：`BaseMeshEffect` 本身已经在 `OnEnable` / `OnDisable` / `OnDidApplyAnimationProperties` 中调用了 `graphic.SetVerticesDirty()`。因此继承 `BaseMeshEffect` 时只需**保留 `base.OnEnable()` 调用链**；只有**直接实现 `IMeshModifier`**（不继承 BaseMeshEffect）的组件，才需要自己通知：

```csharp
protected override void OnEnable()
{
    base.OnEnable();               // ← BaseMeshEffect 内部已调用 SetVerticesDirty()
    // 参数变化时仍需主动标记：
    if (graphic != null)
        graphic.SetVerticesDirty();  // 标记顶点已脏，下一帧重建
}
```

忘记通知 `SetVerticesDirty()` 是自定义 Mesh Effect 最常见的 bug——改了参数但画面不变，因为 ModifyMesh 根本没重新执行。

### 17.1.3 ModifyMesh 的标准写法

```csharp
public override void ModifyMesh(VertexHelper vh)
{
    List<UIVertex> verts = new List<UIVertex>();  // ⚠ 每次调用 new 分配

    vh.GetUIVertexStream(verts);  // 读取所有顶点到列表

    // 修改顶点
    for (int i = 0; i < verts.Count; i++)
    {
        UIVertex v = verts[i];
        v.position += Vector3.right * 10f;  // 修改位置
        v.color = Color.red;                 // 修改颜色
        verts[i] = v;                        // 写回（UIVertex 是结构体，必须写回）
    }

    vh.Clear();                     // 清空原始顶点
    vh.AddUIVertexTriangleStream(verts);  // 写回修改后的顶点
}
```

四个步骤：**读取 → 修改 → 清空 → 写回**。

**关于 GC 压力**：这段代码每次调用都会 `new List<UIVertex>()`——这是自定义 Mesh Effect 最常见的性能陷阱。`UIVertex` 结构体包含 9 个字段（position、normal、tangent、color、uv0~uv3、prevPosition），共 **124 字节**，IL2CPP 下对齐后约 **128 字节**（见第 2 章 2.7）。一个文本 1000 个顶点，Outline 处理后变成 5000 个顶点，单次分配就是 5000×128 ≈ **640KB**。如果频繁触发 `SetVerticesDirty()`，GC 压力会非常明显。

**优化方案（main 官方写法）**：使用 `ListPool<UIVertex>.Get() / Release()` 复用列表（main 的 Shadow / Outline 就是这么做的）：

```csharp
public override void ModifyMesh(VertexHelper vh)
{
    var verts = ListPool<UIVertex>.Get();
    vh.GetUIVertexStream(verts);
    // ... 修改 ...
    vh.Clear();
    vh.AddUIVertexTriangleStream(verts);
    ListPool<UIVertex>.Release(verts);  // 归还池，零 GC 分配
}
```

如果不想引入对象池，也可以把 `List<UIVertex>` 缓存为类成员字段、每次 `Clear()` 复用——注意列表不得跨帧持有。

### 17.1.4 CPU 顶点修改 vs Shader 特效

| 维度 | CPU 顶点修改（ModifyMesh） | Shader |
|------|:---:|:---:|
| 能否新增顶点 | ✅ 是 | ❌ 否 |
| 能否改变几何形状 | ✅ 是（新增/删除顶点/重组三角形） | ❌ 仅像素级变换 |
| 执行位置 | CPU（重建时） | GPU（每帧） |
| 对顶点数的影响 | ❌ 可能膨胀（Shadow=2×, Outline=5×） | 不增加 |
| 对合批的影响 | 不影响（材质不变） | 看情况 |
| 适合效果 | 阴影、描边、波浪、扭曲、顶点动画 | 颜色变化、UV 动画、渐变、流光 |

**选型原则**：如果效果不涉及新增顶点（颜色渐变、UV 偏移），优先用 Shader——不影响合批、不增加顶点、不产生 GC。涉及新增几何（描边、阴影、轮廓扩展）才用 ModifyMesh。

---

## 17.2 顶点复制：Shadow 与 Outline 的实现

### 17.2.1 Shadow（阴影）

Shadow 不修改原始顶点——它**复制**一份，偏移位置，改颜色，追加回去。

核心逻辑：

```csharp
public override void ModifyMesh(VertexHelper vh)
{
    var verts = ListPool<UIVertex>.Get();   // main 官方写法，避免每次 new
    vh.GetUIVertexStream(verts);

    int originalCount = verts.Count;  // ⚠ 必须提前缓存

    for (int i = 0; i < originalCount; i++)
    {
        UIVertex vt = verts[i];        // 值复制（结构体）

        Vector3 pos = vt.position;
        pos.x += effectDistance.x;     // 偏移位置
        pos.y += effectDistance.y;
        vt.position = pos;

        vt.color = effectColor;        // 改颜色（半透明的阴影色）

        verts.Add(vt);                 // 追加回去
    }

    vh.Clear();
    vh.AddUIVertexTriangleStream(verts);
    ListPool<UIVertex>.Release(verts);
}
```

> 说明：上面的 `ListPool` 写法与 main 的 `Shadow.cs` 一致；为便于理解，省略了 `IsActive()` 判断等细节。

效果：

```
原始：4 顶点（Image）
  → 复制 4 顶点 → 偏移 (x, y) → 改阴影色 → 追加
最终：8 顶点 = 原始 4 + 阴影 4 → 同时渲染两个 Quad
```

**注意**：`UIVertex` 是结构体，`verts[i]` 取值时发生值复制。如果只改 `v` 不写回 `verts[i] = v`，修改不生效。追加时同理——追加的也是结构体副本。

### 17.2.2 Outline（描边）

Outline 继承自 Shadow，向**多个方向**重复复制顶点：

```csharp
public override void ModifyMesh(VertexHelper vh)
{
    var verts = ListPool<UIVertex>.Get();   // 与 main 的 Outline.cs 一致
    vh.GetUIVertexStream(verts);

    // 每次 ApplyShadow 都从 start 复制到当前末尾，并把 start 推到新的末尾，
    // 这样四次复制各自只处理"原始那一批"，不会复制到自己刚追加的顶点
    int start = 0;
    int end = verts.Count;

    ApplyShadow(verts, effectColor, start, verts.Count,
        effectDistance.x, -effectDistance.y);  // 下
    start = end; end = verts.Count;
    ApplyShadow(verts, effectColor, start, verts.Count,
        -effectDistance.x, effectDistance.y);  // 上
    start = end; end = verts.Count;
    ApplyShadow(verts, effectColor, start, verts.Count,
        effectDistance.x, effectDistance.y);   // 右
    start = end; end = verts.Count;
    ApplyShadow(verts, effectColor, start, verts.Count,
        -effectDistance.x, -effectDistance.y); // 左

    vh.Clear();
    vh.AddUIVertexTriangleStream(verts);
    ListPool<UIVertex>.Release(verts);
}
```

效果：

```
原始：4 顶点
  → 向左复制 → 向右复制 → 向上复制 → 向下复制
最终：20 顶点 = 原始 4 + 4 个方向各 4
```

**顶点爆炸的实际影响**：

| 场景 | 无特效 | + Shadow | + Outline |
|------|:---:|:---:|:---:|
| Image（4 顶点） | 4 | 8 | 20 |
| Text 100 字（400 顶点） | 400 | 800 | 2000 |
| Text 1000 字（4000 顶点） | 4000 | 8000 | 20000 |
| 1000 字 + Shadow + Outline | 4000 | 8000（×2） | **40000**（8000 ×5） |

这就是为什么移动端项目通常禁止 Text 使用 Outline——文本本身就几千顶点，Outline 的 ×5 乘法效应直接爆炸。而且所有顶点都在 CPU 侧处理完才上传 GPU，重建时的 CPU 开销、GC 压力、上传带宽都会放大。

### 17.2.3 顶点复制的典型陷阱

**① 忘记缓存原始数量 → 无限递归**

```csharp
// ❌ 错误：verts.Count 会随着追加不断增长
for (int i = 0; i < verts.Count; i++)
    verts.Add(verts[i]);

// ✅ 正确：先缓存原始数量
int count = verts.Count;
for (int i = 0; i < count; i++)
    verts.Add(verts[i]);
```

**② 三角形顺序破坏**

`GetUIVertexStream` 返回的顶点按三角形流排列，每 3 个顶点一个三角形。追加时必须按同样 3 顶点一组追加，否则 Mesh 出现撕裂/翻转。

**③ 顶点膨胀不是加法是乘法**

多个特效叠加时，顶点数不是相加，而是依次乘法：

```
无特效：       4 顶点
+ Shadow：     4 × 2 = 8 顶点
  + Outline：  8 × 5 = 40 顶点（Outline 对已有 8 顶点做 4 方向复制）
    + 另一特效：继续乘法膨胀
```

---

## 17.3 多效果叠加

### 17.3.1 顺序决定结果

多个 BaseMeshEffect 按 Inspector 顺序依次执行，共用同一份 VertexHelper。顺序不同，结果不同：

```
顺序：Shadow → Outline
  1. Shadow 把 4 顶点变成 8 顶点（原始 + 阴影）
  2. Outline 对 8 顶点全部做 4 方向复制 → 40 顶点
  结果：阴影也有描边

顺序：Outline → Shadow
  1. Outline 把 4 顶点变成 20 顶点（原始 + 4 方向描边）
  2. Shadow 对 20 顶点全部做偏移复制 → 40 顶点
  结果：描边全部偏移，阴影"扩散"到整个描边范围
```

**链式污染**：后执行的特效看到的不是原始 Mesh，而是前面所有特效修改过的结果。如果渐变组件写在 Outline 后面，它会连描边顶点一起渐变——这经常导致意外效果。

### 17.3.2 多特效的 GC 成本

每次 Graphic 重建，UGUI 都会调用 `GetComponents<IMeshModifier>()`——这是一个数组分配。加上每个 ModifyMesh 内部的 new List，一个带有 3 个特效的 Text 在重建时可能产生：

```
GetComponents<IMeshModifier>()   → 1 次数组分配
特效 1（Shadow）GetUIVertexStream  → 1 次 List 分配
特效 2（Outline）GetUIVertexStream → 1 次 List 分配
特效 3（Self）GetUIVertexStream    → 1 次 List 分配
```

如果这个 Text 每帧更新（实时刷新的数值、倒计时），GC Alloc 会迅速累积。这就是动态文本 + 多层特效往往成为性能热点的原因——不仅仅是顶点多，GC 分配也频繁。

### 17.3.3 优化建议

1. **用 `ListPool<UIVertex>` 或缓存列表**：main 官方做法是 `ListPool.Get()/Release()` 复用（Shadow/Outline 即如此）；也可声明类字段 + `Clear()` 复用，避免每次 new 分配
2. **减少特效层数**：特别是 Outline + Shadow 同时使用时，评估是否真正需要
3. **Text 用 Shader 替代 Mesh Effect**：Text 顶点基数大，Mesh Effect 的乘法效应太严重——能用 Shader 做描边/阴影就优先用 Shader
4. **避免每帧 SetVerticesDirty**：动态内容用缓存判断内容是否真的变了再标记 Dirty
5. **优先考虑 `DrawMesh` 或自定义 Shader**：如果效果不涉及新增顶点，不走 ModifyMesh 更高效

---

## 本章总结

```
顶点生成 → 顶点修改 → Mesh 提交 → GPU

原始 OnPopulateMesh
  → 4 顶点（Image）/ N×4 顶点（Text）
    → ModifyMesh 链
      → Shadow：复制 ×2（偏移 + 改色）
      → Outline：复制 ×5（作用于已扩展的顶点）
      → 其他 Effect：继续乘法膨胀
        → CanvasRenderer.SetMesh(finalMesh)
```

UI Mesh 扩展机制本质上是一条**可编程几何流水线**。它让 UGUI 从"固定矩形渲染器"变成了"可编程二维几何系统"——代价是 CPU 开销、顶点爆炸和 GC 压力，需要在使用时做出权衡。

**核心选型原则**：

```
需要新增顶点 → ModifyMesh（阴影、描边、波浪、扭曲）
不需要新增顶点 → Shader（颜色渐变、UV 动作、流光）
```

---

### 推荐的源码阅读路径

```
IMeshModifier.cs → BaseMeshEffect.cs → Shadow.cs → Outline.cs → PositionAsUV1.cs
配合 VertexHelper.GetUIVertexStream / AddUIVertexTriangleStream 阅读
```

---

## 勘误汇总

| # | 严重程度 | 原文章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🟡 中等 | 全文 | 未提及 `GetUIVertexStream()` 每次调用 new List 的 GC 分配问题 | 每次 ModifyMesh 调用都分配 List，UIVertex 约 124~128 字节（9 字段），多层特效 + 高频重建时 GC 压力显著。应用 `ListPool` 复用（main 官方写法） |
| 2 | 🟡 中等 | 全文 | 未提及 `GetComponents<IMeshModifier>()` 每次 Graphic 重建时的数组分配 | `GetComponents` 在每次重建时分配数组，这也是一个频繁触发的 GC 点 |
| 3 | 🟢 轻微 | 17.3.2 | "UIVertex 是结构体，因此发生了值复制" | 正确，但未说明 UIVertex 约 124~128 字节。几千个顶点的结构体复制本身就有可观的 CPU 缓存压力，不仅仅是 GC 问题 |
| 4 | 🟢 轻微 | 17.3.3 / 17.3.5 / 17.4.3 | 顶点爆炸数字（4→8→40、Text 1000→Shadow 2000→Outline 5000）在三节中重复出现 | 同一组数字反复出现，仅上下文不同 |
| 5 | 🟢 轻微 | 17.1.4 / 17.4.6 | "CPU Mesh 修改与 Shader 的区别"与"优先使用 Shader 特效" | 两个节都在对比 CPU vs Shader，论点相同 |
| 6 | 🔴 严重 | 17.1.3 / 17.3.2 | UIVertex 是 8 个字段、约 52 字节 | Unity 6 / main 为 9 个字段（含 `prevPosition`），共 124 字节，IL2CPP 对齐后约 128 字节 |
| 7 | 🟡 中等 | 17.1.2 | 继承 BaseMeshEffect 时必须在 OnEnable 手动调用 `SetVerticesDirty()` | `BaseMeshEffect` 已自动处理；只需保留 `base.OnEnable()`。直接实现 `IMeshModifier` 才需要自己通知 |
| 8 | 🟡 中等 | 17.1.3 / 17.3.3 | 优化方案为缓存成员 `List<UIVertex>` | main 官方写法是 `ListPool<UIVertex>.Get()/Release()`（Shadow/Outline 即如此），成员字段缓存只是替代方案 |
| 9 | 🟢 轻微 | 17.2.2 表格 | 1000 字 + Outline + Shadow = **28000** 顶点 | 按本章 17.2.3③ 自述的乘法规则，两种叠加顺序都得 **40000**（4000 ×2 ×5）；第 20 章 20.4 给的也是 40000。28000 无法由文中任何规则推出，已更正 |
| 10 | 🟢 轻微 | 17.2.2 代码 | Outline 示例使用了从未声明的 `startIndex` | 已改为显式维护 `start` / `end` 的写法，并说明为什么每轮要推进起点（避免复制到自己刚追加的顶点） |
