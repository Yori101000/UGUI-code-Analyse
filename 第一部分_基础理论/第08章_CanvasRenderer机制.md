# 第8章 CanvasRenderer 机制

> CanvasRenderer 是 UGUI 渲染体系中最容易被忽视、却又最关键的一层——它是连接 C# UI 数据与 Native 渲染引擎的桥梁。理解 CanvasRenderer，才能真正理解 UGUI 的 Mesh 是怎么"送到 GPU 面前"的。

---

## 8.1 定位：提交者，而非生成者

### 8.1.1 三层职责模型

UGUI 的渲染体系可以用三层模型来概括：

| 层级 | 组件 | 职责 | 类比 |
|------|------|------|------|
| 数据生成层 | Graphic（Image/Text/RawImage） | 生成顶点、UV、颜色、三角形索引 | 设计师画图纸 |
| 渲染调度层 | Canvas + CanvasUpdateRegistry | 调度重建、执行批处理、提交 DrawCall | 项目经理排工期 |
| **最终提交层** | **CanvasRenderer** | **保存单个 UI 元素的 Mesh 与材质，供 Canvas 读取** | **材料仓库** |

关键认知：**CanvasRenderer 不参与数据生成，不参与布局计算，不参与批处理逻辑。它的全部职责就是"存数据"和"交数据"——把 Graphic 生成好的 Mesh 和 Material 存起来，等着 Canvas.BuildBatch() 来取。**

```
Graphic.OnPopulateMesh()     → 生成顶点数据
       ↓
CanvasRenderer.SetMesh()      → 存入数据（仓库）
       ↓
Canvas.BuildBatch()           → 读取数据、合并、提交 DrawCall
       ↓
GPU                            → 渲染到屏幕
```

### 8.1.2 每个渲染元素对应一个 CanvasRenderer

在 Unity Editor 中选中任意一个 UI 元素（Image、Text、RawImage），可以在 Inspector 中看到它自动带有一个 **CanvasRenderer** 组件。这个组件不是开发者手动挂载的——当 UI 元素被创建为 Canvas 的子对象时，UGUI 会自动为每个需要渲染的 GameObject 添加 CanvasRenderer。

```csharp
// CanvasRenderer 被自动添加，开发者通常不需要直接操作它
// 但在 Inspector 中可以看到其属性：
//   - Cull Transparent Mesh（对应 cull 属性）
//   - Material Count（对应 materialCount）
```

### 8.1.3 不是所有的 UI 组件都有 CanvasRenderer

只有需要渲染的 UI 组件才需要 CanvasRenderer。纯逻辑组件（如 Button、LayoutGroup、EventSystem 等）没有 CanvasRenderer，因为它们不产生任何可视内容。

> **规则**：Graphic 的继承链上需要 CanvasRenderer；Selectable 和 LayoutGroup 的继承链上不需要。

---

## 8.2 核心接口：数据仓库的"存"与"取"

CanvasRenderer 暴露给 C# 层的核心方法就是一组 Set 方法。这些方法的共同特征是：**只负责存，不负责渲染**。

### 8.2.1 SetMesh：存入网格数据

`SetMesh(Mesh)` 是 CanvasRenderer 最重要的方法。它接收一个 Mesh 对象，存储为当前 UI 元素的网格数据。

```csharp
// CanvasRenderer（引擎内置组件，位于 UnityEngine.UIModule）
public void SetMesh(Mesh mesh);
```

当调用 `SetMesh` 时，CanvasRenderer 不会复制 Mesh 数据——它持有对 Mesh 对象的引用。这意味着：
- 如果后续修改了该 Mesh 的顶点，CanvasRenderer 中看到的是修改后的数据
- `Graphic.cs` 内部使用的 `m_WorkerMesh` 是同一个 Mesh 实例的反复重用

> 以上为引擎侧行为，C# 无源码，依据官方文档与实际渲染行为推断。

### 8.2.2 SetMaterial：存入材质

```csharp
public void SetMaterial(Material material, int index);
```

`index` 参数支持 CanvasRenderer 的多材质分配（`materialCount` > 1 时），但绝大多数 UI 元素只用 index = 0。

### 8.2.3 SetTexture：存入纹理

```csharp
public void SetTexture(Texture texture);
```

`SetTexture` 单独设置纹理，在 UGUI 中通常与 `SetMaterial` 配合使用。`Graphic.UpdateMaterial()` 中两者的调用顺序是：先 SetMaterial，后 SetTexture。

### 8.2.4 SetColor：存入顶点颜色

```csharp
public void SetColor(Color color);
```

这个颜色值最终会与 Mesh 中的顶点颜色进行混合（相乘），作为 UI 最终的显示颜色。`Graphic.color` 属性最终就是通过此方法传递给 CanvasRenderer。

> （引擎行为：`SetColor` 的混色方式由引擎与 UI Shader 共同决定，C# 侧无源码，依据官方文档与渲染结果推断。）

### 8.2.5 Clear：清空所有数据

```csharp
public void Clear();
```

当 UI 元素被禁用或销毁时，CanvasRenderer 需要调用 `Clear` 来释放对 Mesh 和 Material 的引用，避免渲染残留。

---

## 8.3 与 Graphic 的协作关系

CanvasRenderer 本身是"被动"的——它自己不生成任何数据，所有数据都由 Graphic 推送过来。

### 8.3.1 Graphic.UpdateGeometry 中的协作

```csharp
// Graphic.cs（UGUI main）：dirty 判断在 Rebuild 的 PreRender 阶段
private void Rebuild(CanvasUpdate update)
{
    if (update == CanvasUpdate.PreRender)
    {
        if (m_VertsDirty)
        {
            UpdateGeometry();   // → DoMeshGeneration()，生成并提交顶点
            m_VertsDirty = false;
        }
        if (m_MaterialDirty)
        {
            UpdateMaterial();
            m_MaterialDirty = false;
        }
    }
}

// UpdateGeometry 本身只负责"生成 + 提交"：
// DoMeshGeneration() → OnPopulateMesh(s_VertexHelper)
//                    → IMeshModifier 链
//                    → s_VertexHelper.FillMesh(workerMesh)
//                    → canvasRenderer.SetMesh(workerMesh)
```

流程解析：
1. `SetVerticesDirty()` 设置 `m_VertsDirty`，并把 Graphic 注册到 `CanvasUpdateRegistry` 的图形重建队列
2. `Canvas.willRenderCanvases` 触发后，`PerformUpdate()` 在 `PreRender` 阶段调用 `Graphic.Rebuild()`
3. `UpdateGeometry()` → `DoMeshGeneration()`：子类 `OnPopulateMesh(VertexHelper)` 生成顶点 → `FillMesh(workerMesh)` → `canvasRenderer.SetMesh(workerMesh)`
4. 重建结束清除 dirty 标记（main 中不存在 `m_DirtyVerts` / `DoMeshUpdate` 增量路径）

### 8.3.2 Graphic.UpdateMaterial 中的协作

```csharp
// Graphic.cs（UGUI main）中的材质更新
private void UpdateMaterial()
{
    if (canvasRenderer == null)
        return;

    canvasRenderer.materialCount = 1;
    canvasRenderer.SetMaterial(materialForRendering, 0);  // 经过 IMaterialModifier 链的最终材质
    canvasRenderer.SetTexture(mainTexture);      // 设置主纹理
}
```

### 8.3.3 Graphic 保存对 CanvasRenderer 的引用

```csharp
// Graphic.cs（UGUI main）：惰性获取，首次访问时才 GetComponent / AddComponent
public CanvasRenderer canvasRenderer
{
    get
    {
        if (ReferenceEquals(m_CanvasRenderer, null))
        {
            m_CanvasRenderer = GetComponent<CanvasRenderer>();
            if (ReferenceEquals(m_CanvasRenderer, null))
                m_CanvasRenderer = gameObject.AddComponent<CanvasRenderer>();
        }
        return m_CanvasRenderer;
    }
}
```

main 中 `Graphic` **不在 Awake 里主动获取** CanvasRenderer——通过上面的惰性属性，首次访问（通常是 `UpdateMaterial()` / `DoMeshGeneration()` 时）才创建。这与第 5 章 5.8.1 的描述一致。

### 8.3.4 Graphic 生命周期中的 CanvasRenderer 操作

| Graphic 生命周期事件 | CanvasRenderer 操作 | 说明 |
|---------------------|-------------------|------|
| 首次访问 `canvasRenderer` 属性 | `GetComponent` → 没有则 `AddComponent` | 惰性建立引用，**不在 Awake 中**（见 8.3.3） |
| OnEnable | - | CanvasRenderer 自动启用 |
| Rebuild(PreRender) | `SetMesh()` + `SetMaterial()` + `SetTexture()` | 数据更新 |
| OnDisable | `Clear()` | 释放引用，避免残留 |
| OnDestroy | - | CanvasRenderer 随 GameObject 销毁 |

---

## 8.4 C# 层与 Native C++ 渲染层的桥梁

CanvasRenderer 特殊之处在于：**它不是一个纯 C# 类。它的核心方法都是在 C# 侧声明，但实现位于 Unity 引擎的 Native C++ 层。**

### 8.4.1 [NativeMethod] 特性

```csharp
// CanvasRenderer 的核心方法标注了 [NativeMethod]
// 意味着这些方法在 C# 中只有 extern 声明，实现由引擎内部 C++ 代码提供

[NativeMethod("SetMesh")]
public extern void SetMesh(Mesh mesh);

[NativeMethod("SetMaterial")]
public extern void SetMaterial(Material material, int index);

[NativeMethod("SetTexture")]
public extern void SetTexture(Texture texture);

[NativeMethod("SetColor")]
public extern void SetColor(Color color);
```

`[NativeMethod]` 注解告诉 Unity 的 IL2CPP 或 Mono 运行时：这个方法对应的实现不在 C# 程序集中，而是在 Unity Player 引擎的 Native 代码中。标有 `extern` 关键字的这些方法，其函数体由引擎内部实现。

### 8.4.2 为什么需要 Native 实现

CanvasRenderer 的关键数据（Mesh 数据、Material 引用、Texture 引用）在 Native 侧也需要维护一份，因为：

1. **Canvas.BuildBatch 在 Native 侧执行**——它需要直接读取 CanvasRenderer 的 Native 数据来合并 Mesh
2. **渲染线程在 Native 侧**——DrawCall 的提交需要 Native 侧的 Mesh 和 Material 数据
3. **避免 C# 到 C++ 的频繁跨语言调用**——数据直接存储在 Native 侧，减少 P/Invoke 开销

所以 CanvasRenderer 的本质是：**在 C# 侧暴露 Set 方法，内部将数据同步到 Native 侧的同名数据结构**。Graphic 把数据 push 到 C# 侧的 CanvasRenderer，C# 侧的 CanvasRenderer 再同步到 Native 侧。

```
Graphic (C#)
  ↓ SetMesh(workerMesh)
CanvasRenderer.SetMesh (C# 声明)
  ↓ [NativeMethod]  →  P/Invoke 调用
CanvasRenderer.SetMesh (C++ 实现)
  ↓ 存储 Mesh 指针
Native Mesh Data（渲染线程可见）
```

### 8.4.3 其他 Native 方法

```csharp
// 裁剪相关的 Native 方法
[NativeMethod("EnableRectClipping")]
public extern void EnableRectClipping(Rect clipRect);

[NativeMethod("DisableRectClipping")]
public extern void DisableRectClipping();

[NativeMethod("SetClipping")]
public extern void SetClipping(bool performClipping);

// 查询方法
public extern float GetAlpha();
public extern Color GetColor();
public extern Material GetMaterial(int index);
```

### 8.4.4 C# 侧封装的属性

除了上面这些方法，CanvasRenderer 还暴露了一组属性。注意它们**同样由引擎侧维护**——C# 只是读写入口，真实状态存在 Native 侧（与 8.4.2 的结论一致）：

```csharp
// C# 侧可读写，实际状态由引擎维护
public bool cull { get; set; }              // 是否剔除渲染（RectMask2D 使用）
public int absoluteDepth { get; }            // 用于排序的深度值
public int materialCount { get; set; }       // 材质数量（多材质支持）
public bool hasMoved { get; }                // 上一帧以来是否移动过
```

---

## 8.5 裁剪支持（Clipping）

CanvasRenderer 提供了矩形裁剪的支持——这是 **RectMask2D** 的底层机制。

### 8.5.1 EnableRectClipping 与 DisableRectClipping

```csharp
// 启用矩形裁剪
public void EnableRectClipping(Rect clipRect);

// 禁用矩形裁剪
public void DisableRectClipping();
```

`EnableRectClipping` 接收一个 `Rect` 参数（裁剪矩形），设置后该 CanvasRenderer 提交的内容只在矩形区域内可见。**它不修改 Mesh 顶点数据**——裁剪由 UI Shader 完成：引擎把矩形传给材质的 `_ClipRect`，并开启 `UNITY_UI_CLIP_RECT` 关键字，片元阶段执行 `color.a *= UnityGet2DClipping(worldPosition.xy, _ClipRect)`（见第 18 章）。

### 8.5.2 与 RectMask2D 的关系

RectMask2D 通过在子节点的 CanvasRenderer 上调用 `EnableRectClipping` 来实现裁剪：

```csharp
// RectMask2D.cs（简化逻辑）
public void PerformClipping()
{
    // 计算裁剪区域（考虑父级 Mask 的合并）
    Rect clipRect = GetCombinedClipRect();

    // 遍历被裁剪的子节点，在它们的 CanvasRenderer 上启用裁剪
    foreach (var renderer in m_ClipTargets)
    {
        renderer.EnableRectClipping(clipRect);
    }
}
```

与 Mask（基于 Stencil Buffer）不同，RectMask2D **不创建任何材质实例**——这是它的核心性能优势。但"不创建材质实例"不等于"不断批"：裁剪矩形与 `UNITY_UI_CLIP_RECT` 关键字本身也是渲染状态，**同一裁剪矩形下的元素可以合批，分属不同 RectMask2D 的元素之间仍会断批**（Frame Debugger 显示 `Different RectMask2D`）。完整对比见第 15 章。

### 8.5.3 SetClipping

```csharp
public void SetClipping(bool performClipping);
```

一个开关方法，控制当前 CanvasRenderer 是否参与裁剪。当设为 `false` 时，即使调用了 `EnableRectClipping` 也不会生效。

---

## 8.6 其他重要属性

### 8.6.1 cull

```csharp
public bool cull { get; set; }
```

`cull` 控制是否对当前 CanvasRenderer 进行剔除。当设为 `true` 时，该 CanvasRenderer 对应的 UI 元素不会被渲染。

RectMask2D 通过此属性来剔除完全在裁剪区域之外的 UI 元素。如果一个 UI 元素完全在 RectMask2D 的裁剪区域之外，RectMask2D 会将该元素的 CanvasRenderer.cull 设置为 `true`，跳过该元素的渲染。

```
UI 元素与裁剪区域的关系       cull 值      渲染行为
完全在裁剪区域内              false        正常渲染
部分在裁剪区域内              false        正常渲染 + 裁剪
完全在裁剪区域外              true         跳过渲染（不提交 DrawCall）
```

### 8.6.2 absoluteDepth

```csharp
public int absoluteDepth { get; }
```

`absoluteDepth` 返回当前 CanvasRenderer 在 Canvas 渲染层级中的绝对深度值。这个值由引擎自动计算，用于排序——深度值越大的元素，在渲染时越靠后（覆盖在更上层）。

### 8.6.3 materialCount

```csharp
public int materialCount { get; set; }
```

有些 UI 元素需要多个材质（例如带 Outline 或 Shadow 特效的 Text），此时 `materialCount` 可以设为大于 1，然后通过 `SetMaterial(material, index)` 为每个索引设置不同的材质。

---

## 8.7 CanvasRenderer 的完整生命周期

从 UI 元素的创建到销毁，CanvasRenderer 经历以下阶段：

```
创建阶段：
  1. GameObject 实例化（new GameObject 或 Instantiate 预制体）
  2. Graphic 首次访问 canvasRenderer 属性
       → GetComponent<CanvasRenderer>()，没有则 AddComponent（惰性，见 8.3.3）

启用阶段：
  4. CanvasRenderer.enabled = true
  5. Graphic.OnEnable() → 注册到 CanvasUpdateRegistry

数据更新阶段（每帧可能触发）：
  6. UI 数据变化 → SetVerticesDirty() / SetMaterialDirty()
  7. 下一帧 CanvasUpdateRegistry.PerformUpdate()
  8. Graphic.Rebuild(PreRender)
  9. UpdateGeometry() → canvasRenderer.SetMesh(workerMesh)
  10. UpdateMaterial() → canvasRenderer.SetMaterial() + SetTexture()
  11. [Native] Canvas.BuildBatch() 读取 CanvasRenderer 数据 → 合并 → 提交 DrawCall

禁用阶段：
  12. Graphic.OnDisable() → canvasRenderer.Clear()
  13. CanvasRenderer 数据清空，不再参与渲染

销毁阶段：
  14. GameObject.Destroy() → CanvasRenderer 随组件销毁
```

### 核心时间线

```
帧 N                   帧 N+1                  帧 N+2
│                      │                      │
SetVerticesDirty() ──→ CanvasUpdateRegistry  │
（打标记）             PerformUpdate() ──────→ 渲染到屏幕
                       Graphic.Rebuild()     │
                       canvasRenderer        │
                       .SetMesh()            │
                       Canvas.BuildBatch() ──→ GPU 执行
```

---

## 8.8 源码级要点总结

### 8.8.1 CanvasRenderer 不是纯 C# 类

| 方面 | 说明 |
|------|------|
| 命名空间 | `UnityEngine`（不是 `UnityEngine.UI`） |
| 存储位置 | Unity 引擎内置组件（UnityEngine.UIModule） |
| 核心方法 | `extern` + `[NativeMethod]`，实现位于引擎 C++ 层 |
| C# 侧角色 | 仅仅是"前端接口"，真正的数据存储在 Native 侧 |

### 8.8.2 CanvasRenderer 是"数据容器"而非"执行者"

这是最重要的认知：

- **错误理解**：CanvasRenderer 负责把数据提交给 GPU
- **正确理解**：CanvasRenderer 只负责存储 Graphic 生成的数据。真正的"提交者"是 `Canvas.BuildBatch()`，它在 Native 侧从 CanvasRenderer 读取数据、合并、提交

### 8.8.3 与 Graphic 的引用关系

```
Graphic (挂载在 GameObject 上)
  ├── has a → CanvasRenderer（同 GameObject 上的组件）
  ├── SetVerticesDirty() → 标记顶点需要重建
  └── UpdateGeometry()  → canvasRenderer.SetMesh(m_WorkerMesh)
```

Graphic 持有 CanvasRenderer 的引用，但 CanvasRenderer 不持有 Graphic 的引用。数据流动是单向的：Graphic → CanvasRenderer。

---

## 8.9 本章总结

1. **三层职责模型**：Graphic（数据生成）→ Canvas（调度批处理）→ CanvasRenderer（数据存储）

2. **核心接口**：`SetMesh()`、`SetMaterial()`、`SetTexture()`、`SetColor()` —— 这四个方法构成了 CanvasRenderer 的"数据仓库"功能

3. **C# 与 Native 的桥梁**：CanvasRenderer 的核心方法标注 `[NativeMethod]`，实现位于 Unity 引擎的 C++ 层，这是 UGUI 中 C# 数据流向渲染线程的关键通道

4. **裁剪支持**：通过 `EnableRectClipping()` / `DisableRectClipping()` 实现矩形裁剪，这是 RectMask2D 底层机制，与 Mask 的 Stencil 方案不同

5. **被动的组件**：CanvasRenderer 不主动做任何事——它等待 Graphic 调用它的 Set 方法来存入数据，等待 Canvas.BuildBatch() 来读取数据

6. **每个渲染 UI 元素一个 CanvasRenderer**：Image、Text、RawImage 等所有可视组件都依赖 CanvasRenderer 来存储和传递它们的渲染数据

### 推荐的源码阅读路径

```
打开 CanvasRenderer API 文档 → 重点阅读：
  1. [NativeMethod] 标注的方法 ← 理解 C# 与 Native 的边界
  2. SetMesh()          ← 核心方法，数据存入的入口
  3. EnableRectClipping ← 裁剪机制的底层接口

打开 Graphic.cs → 对照阅读：
  4. UpdateGeometry()   ← 理解 SetMesh 的调用时机
  5. UpdateMaterial()   ← 理解 SetMaterial/SetTexture 的调用时机
  6. m_CanvasRenderer   ← 理解引用建立的方式
```

---

## 勘误汇总（对照 uGUI main）

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 | 8.3.1 | `UpdateGeometry()` 内有 `m_VertsDirty` / `m_DirtyVerts` 分支和 `DoMeshUpdate()` 增量路径 | main 中不存在 `m_DirtyVerts` / `DoMeshUpdate`；dirty 判断在 `Graphic.Rebuild` 的 `PreRender` 分支，`UpdateGeometry()` 只调用 `DoMeshGeneration()` |
| 2 | 🟡 | 8.3.2 | `canvasRenderer.SetMaterial(material, 0)` | main 使用经过 `IMaterialModifier` 链的 `materialForRendering`，并先设置 `materialCount = 1` |
| 3 | 🟡 | 8.3.3 | `Graphic` 在 `Awake` 中获取 `CanvasRenderer` 引用 | main 用惰性 `canvasRenderer` 属性，首次访问时 `GetComponent` / `AddComponent` |
| 4 | 🟡 | 8.2.1 / 8.8.1 | CanvasRenderer 位于 `UnityEngine.CoreModule` | 属 `UnityEngine.UIModule` |
| 5 | 🟡 | 8.3.4 / 8.7 | 生命周期表与创建阶段仍写「Awake → `GetComponent<CanvasRenderer>()` 建立引用」 | 与 8.3.3 及 main 一致：惰性 `canvasRenderer` 属性，首次访问时才 `GetComponent`/`AddComponent`。此处是勘误 #3 修正时的遗漏 |
| 6 | 🟡 | 8.4.4 | `cull` / `absoluteDepth` / `materialCount` 等属性「完全由 C# 管理，不涉及 Native 同步」 | 均为引擎侧属性，C# 只是读写入口，真实状态在 Native 侧——与 8.4.2 的结论一致 |
| 7 | 🔴 | 8.5.2 | RectMask2D「**不断批**——这是它的核心性能优势」 | 不创建材质实例 ≠ 不断批。裁剪矩形与 `UNITY_UI_CLIP_RECT` 关键字本身是渲染状态：同一裁剪矩形内可合批，不同 RectMask2D 之间断批（行为推断，Frame Debugger 显示 `Different RectMask2D`）。全书统一口径见第 15 章 |
