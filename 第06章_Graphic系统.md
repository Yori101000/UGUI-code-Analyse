# 第6章 Graphic 系统

> 本章对应原书结构中的第5章（Graphic 系统）。Graphic 是 UGUI 中所有可渲染 UI 组件的抽象基类，理解 Graphic 的设计是理解 UGUI 渲染流程的核心关键。

---

## 6.1 Graphic 的定位

在 UGUI 的三层架构（组件层 - 系统层 - 渲染层）中，Graphic 位于**组件层的核心位置**。它既是一个"数据生产者"，负责生成 UI 元素的顶点数据；又是一个"连接器"，连接了 UI 的空间信息（RectTransform）与 GPU 渲染（CanvasRenderer）。

一句话总结 Graphic 的职责：

> **Graphic 负责将子类定义的几何数据转换为 Mesh，并通过 CanvasRenderer 提交给渲染管线。**

在 UGUI 中，所有"看得见"的东西——Image、RawImage、Text——都是 Graphic 的子类。

---

## 6.2 类继承层级

### 6.2.1 完整层级

```
UnityEngine.EventSystems.UIBehaviour
  └── UnityEngine.UI.Graphic
       （实现 ICanvasElement, IMaterialModifier, ILayoutElement）
       └── UnityEngine.UI.MaskableGraphic（实现 IClippable）
            ├── UnityEngine.UI.Image
            ├── UnityEngine.UI.RawImage
            └── UnityEngine.UI.Text
```

### 6.2.2 各层级职责

| 类 | 职责 |
|------|------|
| **UIBehaviour** | 封装 Awake / OnEnable / OnDisable / OnDestroy 等生命周期 |
| **Graphic** | 定义 Dirty 标记机制、Rebuild 流程、OnPopulateMesh 虚方法 |
| **MaskableGraphic** | 添加 MaterialPropertyBlock 和 Stencil 裁剪支持 |
| **Image** | 四种 ImageType：Simple / Sliced / Tiled / Filled |
| **RawImage** | 显示不带九宫格裁剪的原始纹理 |
| **Text** | 逐字符生成 Quad，支持富文本和字体回退 |

### 6.2.3 Graphic 实现的接口

Graphic 实现了三个接口，分别对应三类职责：

```csharp
public class Graphic : UIBehaviour,
    ICanvasElement,       // 可被 CanvasUpdateRegistry 调度重建
    IMaterialModifier,    // 可在材质链中被修改（如 Mask 替换材质）
    ILayoutElement        // 可向 Layout 系统反馈尺寸需求
{
    // ...
}
```

#### ICanvasElement

`ICanvasElement` 是 Graphic 参与 Canvas 更新的"门票"。它定义了：

```csharp
public interface ICanvasElement {
    void Rebuild(CanvasUpdate executing);   // 重建入口
    void GraphicUpdateComplete();           // 重建完成回调
    bool IsDestroyed();                     // 是否已销毁
}
```

任何实现了 `ICanvasElement` 的对象都可以通过 `CanvasUpdateRegistry` 注册，并在 Canvas 的更新循环中被调度执行。

#### IMaterialModifier

`IMaterialModifier` 允许在渲染前对材质进行修改。只有一个方法：

```csharp
public interface IMaterialModifier {
    Material GetModifiedMaterial(Material baseMaterial);
}
```

这个接口的核心使用场景是 **Mask 裁剪**：当 Graphic 被 Mask 裁剪时，`MaskableGraphic` 通过实现此方法，将原始材质替换为带 Stencil 裁剪的材质实例。

#### ILayoutElement

在 Layout 系统中，`ILayoutElement` 描述"我需要多少空间"。Graphic 实现了此接口并返回`0`作为默认值，这意味着**子类如果不重写，就不会主动影响 Layout 系统的尺寸计算**。Text 重写了 `preferredWidth` 和 `preferredHeight` 来告诉布局系统它的内容尺寸。

#### 类继承全览

```
UIBehaviour             ← 生命周期（Awake / OnEnable / OnDisable）
  └── Graphic           ← Dirty 标记 + Rebuild 流程
       ├── ICanvasElement      ← 可被 CanvasUpdateRegistry 调度
       ├── IMaterialModifier   ← 可被修改材质（Mask 裁剪）
       └── ILayoutElement      ← 可反馈尺寸需求
```

---

## 6.3 渲染链路：从 UI 到屏幕像素

整个渲染链路由四个环节组成，每一层负责转换一种数据形态：

```
RectTransform            ← 空间结构（位置 + 尺寸 + 旋转 + 缩放）
  ↓ Canvas 空间转换
Graphic / OnPopulateMesh ← 几何生成（顶点 + 三角形 + UV + 颜色）
  ↓ canvasRenderer.SetMesh()
CanvasRenderer           ← 合批提交（存储本 UI 元素的 Mesh/Material）
  ↓ Canvas.BuildBatch()（引擎 native 方法）
GPU                      ← 绘制（Vertex Shader → 光栅化 → Fragment Shader）
```

### 6.3.1 RectTransform：空间结构

每个 UI 元素都有一个 RectTransform，定义了它在 Canvas 空间中的位置、尺寸、旋转和缩放。Graphic 的 `OnPopulateMesh` 需要读取 `rectTransform.rect` 获取坐标数据。

```csharp
// Graphic 读取 RectTransform 信息
public Rect rect {
    get { return rectTransform.rect; }
}
```

Image 的 Simple 类型会生成一个以 `rectTransform.rect` 为范围的四边形网格（Quad），四个顶点的坐标直接来自 RectTransform 的宽度和高度。

### 6.3.2 Graphic：几何生成

这是 UGUI 中最关键的一步——**将抽象的"UI 组件"转换为具体的"几何数据"**。这个过程发生在 `OnPopulateMesh` 中，输出是 `VertexHelper`，最终被写入一个 `Mesh`。

### 6.3.3 CanvasRenderer：合批提交

每个 Graphic 都对应一个 CanvasRenderer 组件。CanvasRenderer 从 Graphic 接收 Mesh 和 Material，但不立即提交——它会等 Canvas 在 `BuildBatch` 阶段统一合并。

```csharp
// CanvasRenderer 的核心字段
private Mesh m_Mesh;           // 本 UI 元素的网格
private Material mMaterial;    // 本 UI 元素的材质
private Texture mTexture;      // 本 UI 元素的纹理
```

### 6.3.4 GPU：最终绘制

Canvas 在 `BuildBatch` 阶段遍历所有 CanvasRenderer，将同材质 + 同纹理 + 同 Stencil 的网格合并为一个 DrawCall，提交给 GPU 绘制。

> ⚠ **勘误 #1**：常见的"UGUI 渲染链路图"中常把 RectTransform 列为渲染链路的起点，但严格来说 RectTransform 不参与渲染——它只提供空间数据。真正启动渲染流程的是 Canvas 的 `WillRenderCanvases` 事件。这个事件由 Unity 引擎的 Canvas 模块在每帧渲染前触发，是 CanvasUpdateRegistry 执行所有 UI 更新的信号源。

---

## 6.4 Dirty 标记机制

这是 UGUI 性能优化的基石。UI 变化不立即重建，而是通过 **Dirty 标记** 延迟到下一帧统一处理。

### 6.4.1 三种 Dirty

Graphic 定义了三种 Dirty 标记，分别对应三种不同的更新需求：

```csharp
// Graphic.cs（UGUI 核心文件）
public class Graphic : UIBehaviour, ICanvasElement {
    private bool m_VertsDirty;    // 顶点 Dirty：布局变化、颜色变化
    private bool m_MaterialDirty; // 材质 Dirty：材质属性变化
    // m_LayoutDirty 继承自 ILayoutElement 的潜在影响
}
```

| Dirty 类型 | 触发方法 | 触发条件 | 重建结果 |
|-----------|---------|---------|---------|
| **Vertices Dirty** | `SetVerticesDirty()` | 尺寸、颜色、图片、文本等变化 | `UpdateGeometry()` → 重新生成 Mesh |
| **Material Dirty** | `SetMaterialDirty()` | 材质、纹理、Tiling/Offset 变化 | `UpdateMaterial()` → 重新设置材质 |
| **Layout Dirty** | `SetLayoutDirty()` | 布局属性变化 | 由 LayoutRebuilder 处理，影响 RectTransform |

### 6.4.2 SetVerticesDirty()

```csharp
public virtual void SetVerticesDirty() {
    if (!IsActive()) return;

    m_VertsDirty = true;
    // 注册到 CanvasUpdateRegistry 的 Graphic 重建队列
    CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);

    // 如果当前节点参与了布局，通知 Layout 系统也需要重建
    if (m_OnDirtyLayoutCallback != null)
        m_OnDirtyLayoutCallback();
}
```

关键逻辑：
1. 只有处于 Active 状态的组件才需要处理
2. 将 `m_VertsDirty` 标记为 true
3. 将自己注册到 CanvasUpdateRegistry——相当于说"我在下一帧需要重建"
4. 如果当前 Graphic 影响了布局（例如 Text 尺寸变化后需要重新布局），触发布局重建回调

### 6.4.3 SetMaterialDirty()

```csharp
public virtual void SetMaterialDirty() {
    if (!IsActive()) return;

    m_MaterialDirty = true;
    // 注册到 CanvasUpdateRegistry 的材质重建队列
    CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
}
```

逻辑与 `SetVerticesDirty` 类似，但标记的是材质需要更新。

### 6.4.4 SetLayoutDirty()

```csharp
// 当影响了布局时调用
protected virtual void SetLayoutDirty() {
    // 触发 LayoutRebuilder 的布局重建
    if (m_OnDirtyLayoutCallback != null)
        m_OnDirtyLayoutCallback();
}
```

`m_OnDirtyLayoutCallback` 通常由 Layout 系统在初始化时注册，指向 `LayoutRebuilder.MarkLayoutForRebuild`。当 Text 的文本内容变化导致尺寸改变时，`SetVerticesDirty` 会触发布局重建，通知父 LayoutGroup 重新排列所有子元素。

### 6.4.5 触发场景

以 Image 组件为例，常见的触发场景：

```csharp
// Image.cs 中的触发点
public override void OnRectTransformDimensionsChange() {
    // 尺寸变化 → 顶点 Dirty
    SetVerticesDirty();
}

public override void OnDidApplyAnimationProperties() {
    // 动画改变颜色 → 顶点 Dirty
    SetVerticesDirty();
}

public void SetSprite(Sprite s) {
    // 替换图片 → 顶点 Dirty + 材质 Dirty（纹理变化）
    SetVerticesDirty();
    SetMaterialDirty();
}

public void SetNativeSize() {
    // 设置为原始尺寸 → 顶点 Dirty + 布局 Dirty
    SetVerticesDirty();
    SetLayoutDirty();
}
```

> 💡 **最佳实践**：当你需要大量批量修改 UI 时（例如 ListView 的 Item 更新），因为 CanvasUpdateRegistry 会在下一帧统一合并，你可以在同一帧内多次调用 `SetVerticesDirty`，系统只会重建一次，不必手动去重。

---

## 6.5 Rebuild 流程详解

这是 Graphic 最核心的流程，也是理解 UGUI 渲染管线的入口。

### 6.5.1 调度机制

重建由 `CanvasUpdateRegistry` 在 `PerformUpdate` 中统一触发：

```csharp
// CanvasUpdateRegistry.cs（伪代码，描述调度逻辑）
private void PerformUpdate() {
    // 阶段一：Layout Rebuild（ICanvasElement 中的 Layout 队列）
    for (int i = 0; i < m_LayoutRebuildQueue.Count; i++) {
        m_LayoutRebuildQueue[i].Rebuild(CanvasUpdate.Layout);
    }

    // 阶段二：PreRender Rebuild（Graphic 重建）
    for (int i = 0; i < m_GraphicRebuildQueue.Count; i++) {
        m_GraphicRebuildQueue[i].Rebuild(CanvasUpdate.PreRender);
    }

    // 阶段三：LatePreRender（Clipping 更新）
    ClipperRegistry.instance.Cull();
}
```

Graphic 的 `Rebuild` 被调用的时机是 `CanvasUpdate.PreRender` 阶段——这个阶段在 Layout 重建完成之后、GPU 绘制之前，确保布局结果已经写入 RectTransform，Graphic 可以读取正确的尺寸生成顶点。

### 6.5.2 Graphic.Rebuild() 源码

```csharp
// Graphic.cs
public virtual void Rebuild(CanvasUpdate update) {
    if (canvasRenderer == null || canvasRenderer.cull) return;

    // 只有在 PreRender 阶段才处理
    if (update == CanvasUpdate.PreRender) {
        // 阶段一：更新顶点几何数据
        if (m_VertsDirty) {
            UpdateGeometry();
            m_VertsDirty = false;
        }

        // 阶段二：更新材质
        if (m_MaterialDirty) {
            UpdateMaterial();
            m_MaterialDirty = false;
        }
    }
}
```

两次 `if` 判断的逻辑：

1. **先判断 CanvasRenderer 是否有效**：如果 `canvasRenderer` 不存在或被裁剪（`cull`），跳过所有操作
2. **只响应 `PreRender` 阶段**：即使 Graphic 也被其它 `CanvasUpdate` 值调用，只处理 `PreRender`
3. **先 Geometry 再 Material**：因为材质可能依赖于顶点数据（如果子类在顶点中编码了 UV/颜色等需要材质配合的信息）
4. **重置 Dirty 标记**：处理完后立即重置，避免重复重建

### 6.5.3 UpdateGeometry() — 顶点重建

```csharp
// Graphic.cs
private void UpdateGeometry() {
    // 步骤一：调用 DoMeshGeneration 生成/更新内部 Mesh
    DoMeshGeneration();

    // 步骤二：将 Mesh 提交给 CanvasRenderer
    canvasRenderer.SetMesh(m_WorkerMesh);
}

private void DoMeshGeneration() {
    // 创建一个 VertexHelper 实例
    using (var vh = new VertexHelper()) {
        // 调用虚方法 → 子类填充顶点数据
        OnPopulateMesh(vh);

        // 将 VertexHelper 的数据写入 m_WorkerMesh
        vh.FillMesh(m_WorkerMesh);
    }
}
```

`DoMeshGeneration` 是 `UpdateGeometry` 的内部实现，它做了三件事：

1. 创建 `VertexHelper`——这是一个临时的顶点数据容器
2. 调用 `OnPopulateMesh(vh)`——子类在此过程中向 `vh` 中添加顶点、三角形等
3. 通过 `vh.FillMesh(m_WorkerMesh)`——将临时数据写入 Graphic 持久的 `m_WorkerMesh` 中

`m_WorkerMesh` 是一个内部重用的 Mesh 对象：

```csharp
[NonSerialized]
private Mesh m_WorkerMesh;  // 不会序列化，每帧重用

private void EnsureWorkerMesh() {
    if (m_WorkerMesh == null) {
        m_WorkerMesh = new Mesh();
        m_WorkerMesh.name = "Graphic Worker Mesh";
        m_WorkerMesh.hideFlags = HideFlags.HideAndDontSave;
        // 标记为动态——提示 Unity 此 Mesh 会被频繁更新
        m_WorkerMesh.MarkDynamic();
    }
}
```

### 6.5.4 UpdateMaterial() — 材质更新

```csharp
// Graphic.cs
private void UpdateMaterial() {
    // 步骤一：选择材质
    // 如果有 modifiedMaterial（通过 IMaterialModifier 链修改过），
    // 使用它；否则使用原始 material
    Material mat = material;
    if (m_ShouldRecalculateStencil) {
        // MaskableGraphic 会通过 IMaterialModifier 生成 Stencil 材质
        var modifiedMat = GetModifiedMaterial(mat);
        mat = modifiedMat;
    }

    // 步骤二：设置材质和纹理到 CanvasRenderer
    canvasRenderer.SetMaterial(mat, mainTexture);

    // 步骤三：设置 Texture 的 Tiling 和 Offset（平铺参数）
    // RawImage 就是通过此参数实现 UV 平铺的
    canvasRenderer.SetTexture(mainTexture);
}
```

> 💡 **注意**：`SetMaterial` 和 `SetTexture` 是 CanvasRenderer 的方法，它们只负责将材质/纹理绑定到当前 UI 元素，真正的 DrawCall 合并是在 Canvas 的 `BuildBatch` 阶段发生的。

### 6.5.5 完整重建时序

```
Frame N:
  1. UI 状态变化 → SetVerticesDirty() 被调用
     → m_VertsDirty = true
     → CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this)

Frame N (当帧渲染前):
  2. Canvas.WillRenderCanvases 事件触发
  3. CanvasUpdateRegistry.PerformUpdate() 执行
     a. Layout Rebuild（布局重建）
     b. PreRender Rebuild（Graphic 重建）
        → 遍历 m_GraphicRebuildQueue
        → 调用每个 Graphic 的 Rebuild(CanvasUpdate.PreRender)
        → 调用 UpdateGeometry()
           → DoMeshGeneration()
              → OnPopulateMesh(vh)  ← 子类重写
              → vh.FillMesh(m_WorkerMesh)
           → canvasRenderer.SetMesh(m_WorkerMesh)
        → 调用 UpdateMaterial()
           → canvasRenderer.SetMaterial(mat, texture)
     c. LatePreRender（裁剪更新）
  4. Canvas.BuildBatch()（引擎 native 方法）
     → 遍历所有 CanvasRenderer
     → 合并同材质 + 同纹理的 Mesh
     → 提交 DrawCall
```

---

## 6.6 OnPopulateMesh —— 子类重写的入口

`OnPopulateMesh` 是 Graphic 提供给子类的**虚方法**。它是所有 UI 几何数据的来源。

### 6.6.1 方法签名

```csharp
// Graphic.cs
protected virtual void OnPopulateMesh(VertexHelper vh) {
    // 默认实现：生成一个覆盖整个 RectTransform 范围的白色 Quad
    var r = GetPixelAdjustedRect();
    var v = new Vector4(r.x, r.y, r.x + r.width, r.y + r.height);
    // 四个顶点
    vh.Clear();
    vh.AddVert(new Vector3(v.x, v.y), color, new Vector2(0f, 0f));
    vh.AddVert(new Vector3(v.x, v.w), color, new Vector2(0f, 1f));
    vh.AddVert(new Vector3(v.z, v.w), color, new Vector2(1f, 1f));
    vh.AddVert(new Vector3(v.z, v.y), color, new Vector2(1f, 0f));
    // 两个三角形
    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

默认实现很简单：以 `RectTransform.rect` 为范围，生成一个白色不透明的四边形网格。如果子类不重写 `OnPopulateMesh`，所有的 Graphic 都会显示为一个白色方块——就像在一个空的 Image 组件上不指定 Sprite 时看到的样子。

### 6.6.2 Image 的 OnPopulateMesh 实现

Image 的 `OnPopulateMesh` 根据 `Image.type`（Simple / Sliced / Tiled / Filled）有不同的实现：

```csharp
// Image.cs（简化示意）
protected override void OnPopulateMesh(VertexHelper toFill) {
    if (activeSprite == null) {
        // 没有 Sprite 时，用 Graphic 的默认实现生成白色方块
        base.OnPopulateMesh(toFill);
        return;
    }

    switch (type) {
        case Type.Simple:
            GenerateSimpleSprite(toFill, preserveAspect);
            break;
        case Type.Sliced:
            GenerateSlicedSprite(toFill);
            break;
        case Type.Tiled:
            GenerateTiledSprite(toFill);
            break;
        case Type.Filled:
            GenerateFilledSprite(toFill, preserveAspect);
            break;
    }
}
```

以 `GenerateSimpleSprite` 为例：

```csharp
// Image.cs
private void GenerateSimpleSprite(VertexHelper vh, bool lPreserveAspect) {
    var r = GetDrawingDimensions(lPreserveAspect);  // 计算最终绘制区域
    var spriteRect = activeSprite.rect;              // Sprite 的原始矩形
    var uv = Sprites.DataUtility.GetOuterUV(activeSprite); // 获取 UV 坐标

    // 清空旧的顶点数据
    vh.Clear();

    // 添加四个顶点（位置、颜色、UV）
    vh.AddVert(
        new Vector3(r.x, r.y),           // 位置：左下角
        color,                           // 颜色
        new Vector2(uv.x, uv.y)          // UV：纹理的左下角
    );
    vh.AddVert(
        new Vector3(r.x, r.w),           // 位置：左上角
        color,
        new Vector2(uv.x, uv.w)          // UV：纹理的左上角
    );
    vh.AddVert(
        new Vector3(r.z, r.w),           // 位置：右上角
        color,
        new Vector2(uv.z, uv.w)          // UV：纹理的右上角
    );
    vh.AddVert(
        new Vector3(r.z, r.y),           // 位置：右下角
        color,
        new Vector2(uv.z, uv.y)          // UV：纹理的右下角
    );

    // 两个三角形构成一个 Quad
    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

**关键区别**：
- `Graphic.OnPopulateMesh` 使用 `GetPixelAdjustedRect()`（考虑 Canvas 缩放偏移）
- `Image.OnPopulateMesh` 使用 `GetDrawingDimensions()`（考虑 Sprite 的 Pixels Per Unit）
- Image 额外计算了 Sprite 的 UV 坐标，不再是简单的 (0,0)-(1,1)

### 6.6.3 Text 的 OnPopulateMesh 实现

Text 的 `OnPopulateMesh` 是最复杂的——它需要将字符串中的每个字符转换为一个 Quad（四个顶点 + 两个三角形）：

```csharp
// Text.cs（简化示意）
protected override void OnPopulateMesh(VertexHelper toFill) {
    // 使用 TextGenerator（Unity 内置的文本生成器）生成字形数据
    IList<UIVertex> verts = cachedTextGenerator.verts;
    // 遍历每个字形
    for (int i = 0; i < verts.Count; i++) {
        // 对每个字形，添加四个顶点
        toFill.AddVert(verts[i]);
        // ... 处理字形的渲染顺序
    }

    // 生成三角形索引
    for (int i = 0; i < verts.Count / 4; i++) {
        toFill.AddTriangle(i * 4, i * 4 + 1, i * 4 + 2);
        toFill.AddTriangle(i * 4 + 2, i * 4 + 3, i * 4);
    }
}
```

Text 的重建有一个额外的影响：当文本内容变化时，除了 `SetVerticesDirty`，它还会通过 `SetLayoutDirty` 通知 Layout 系统重新计算尺寸。这是因为文本的尺寸取决于字体大小和文本长度，父容器需要根据 Text 的 `preferredWidth` 重新布局。

---

## 6.7 VertexHelper —— 顶点构建工具类

`VertexHelper` 是 `OnPopulateMesh` 中使用的核心工具类，它封装了顶点数据的收集过程。

### 6.7.1 VertexHelper 的职责

```
VertexHelper 的作用：
  收集顶点数据 → 整理为 Mesh 可接受的格式 → 填充到 Mesh
```

```csharp
// VertexHelper.cs
public class VertexHelper : IDisposable {
    private List<Vector3> m_Positions;       // 顶点位置（三维坐标）
    private List<Color32> m_Colors;          // 顶点颜色
    private List<Vector2> m_Uv0S;            // UV 通道 0（主纹理）
    private List<Vector2> m_Uv1S;            // UV 通道 1（光照贴图等）
    private List<Vector2> m_Uv2S;            // UV 通道 2
    private List<Vector2> m_Uv3S;            // UV 通道 3
    private List<Vector3> m_Normals;         // 法线
    private List<Vector4> m_Tangents;        // 切线
    private List<int> m_Indices;             // 三角形索引
}
```

`VertexHelper` 维护了 9 个内部列表，覆盖了 Mesh 需要的全部顶点属性。这些数据最终会被写入到一个 `Mesh` 对象中。

### 6.7.2 核心方法

#### AddVert —— 添加一个顶点

```csharp
// VertexHelper.cs
// 方式一：指定位置、颜色、UV
public void AddVert(Vector3 position, Color32 color, Vector2 uv0) {
    m_Positions.Add(position);
    m_Colors.Add(color);
    m_Uv0S.Add(uv0);
    // 法线和切线用默认值
    m_Normals.Add(k_TempNormal);    // 默认 (0, 0, -1)
    m_Tangents.Add(k_TempTangent);  // 默认 (1, 0, 0, -1)
}

// 方式二：直接添加一个 UIVertex 结构体
public void AddVert(UIVertex v) {
    m_Positions.Add(v.position);
    m_Colors.Add(v.color);
    m_Uv0S.Add(v.uv0);
    m_Normals.Add(v.normal);
    m_Tangents.Add(v.tangent);
}
```

注意，`AddVert` 不会自动追踪当前顶点的索引编号——调用者需要按添加顺序记住索引号，以便在 `AddTriangle` 中引用。

#### AddTriangle —— 添加一个三角形

```csharp
// VertexHelper.cs
public void AddTriangle(int idx0, int idx1, int idx2) {
    m_Indices.Add(idx0);
    m_Indices.Add(idx1);
    m_Indices.Add(idx2);
}
```

三个参数分别对应三角形的三个顶点在 `m_Positions` 列表中的索引值。**顶点顺序必须为顺时针方向**，否则三角形面向背面，会被 GPU 裁剪掉。

#### FillMesh —— 填充到 Mesh

```csharp
// VertexHelper.cs
public void FillMesh(Mesh mesh) {
    mesh.Clear();        // 先清空旧数据

    // 将列表数据设置到 Mesh 的各个属性通道
    mesh.SetVertices(m_Positions);
    mesh.SetColors(m_Colors);
    mesh.SetUVs(0, m_Uv0S);
    mesh.SetUVs(1, m_Uv1S);
    mesh.SetNormals(m_Normals);
    mesh.SetTangents(m_Tangents);
    mesh.SetTriangles(m_Indices, 0);  // 0 = submesh 索引
}
```

#### Clear —— 清空数据

```csharp
public void Clear() {
    m_Positions.Clear();
    m_Colors.Clear();
    m_Uv0S.Clear();
    m_Uv1S.Clear();
    m_Uv2S.Clear();
    m_Uv3S.Clear();
    m_Normals.Clear();
    m_Tangents.Clear();
    m_Indices.Clear();
}
```

> ⚠ **勘误 #2**：部分资料将 `Clear()` 描述为"仅重置索引计数器，不清除数据"，这在较新版本的 UGUI 中是不准确的。实际上 `VertexHelper.Clear()` 会清空所有列表——从 2019.x 版本起就是这样实现的。但清空时不会释放容量（`List.Clear()` 不释放 `Capacity`），所以重新添加顶点的过程中不会触发内存分配。

### 6.7.3 一个完整的 Quad 生成示例

```csharp
// 手动创建一个覆盖 RectTransform 的 Quad
void GenerateQuad(VertexHelper vh, Rect rect, Color32 color) {
    vh.Clear();

    // 添加四个顶点
    vh.AddVert(new Vector3(rect.xMin, rect.yMin, 0), color, new Vector2(0, 0));  // 0: 左下
    vh.AddVert(new Vector3(rect.xMin, rect.yMax, 0), color, new Vector2(0, 1));  // 1: 左上
    vh.AddVert(new Vector3(rect.xMax, rect.yMax, 0), color, new Vector2(1, 1));  // 2: 右上
    vh.AddVert(new Vector3(rect.xMax, rect.yMin, 0), color, new Vector2(1, 0));  // 3: 右下

    // 两个三角形
    vh.AddTriangle(0, 1, 2);  // 左下 → 左上 → 右上
    vh.AddTriangle(2, 3, 0);  // 右上 → 右下 → 左下
}
```

生成的拓扑结构：

```
  1 ─────── 2           1 ─────── 2
  │         │            │  ╲     │
  │         │     →      │   ╲    │
  │         │            │    ╲   │
  0 ─────── 3           0 ─────── 3
  四个顶点               两个三角形（6 个索引）
```

### 6.7.4 VertexHelper 的 IDisposable

`VertexHelper` 实现了 `IDisposable` 接口。在 `DoMeshGeneration` 中，通过 `using` 语句确保其被正确释放：

```csharp
private void DoMeshGeneration() {
    using (var vh = new VertexHelper()) {
        OnPopulateMesh(vh);
        vh.FillMesh(m_WorkerMesh);
    }
}
```

不过由于 `VertexHelper.Dispose()` 只执行了 `Clear()`，本质上和手动调用 `Clear()` 没有区别。使用 `using` 的主要目的是代码语义清晰——表示"这个 VertexHelper 的生命周期仅限于当前函数"。

---

## 6.8 Graphic 与 CanvasRenderer 的关系

Graphic 是 C# 层面的组件，负责生成数据；CanvasRenderer 是引擎 native 层面的"呈现器"，负责存储和提交数据。

### 6.8.1 自动创建

Graphic 在 `Awake` 阶段会确保同级 GameObject 上存在一个 CanvasRenderer：

```csharp
// Graphic.cs
protected override void Awake() {
    base.Awake();
    // 确保同级有 CanvasRenderer 组件
    if (canvasRenderer == null) {
        gameObject.AddComponent<CanvasRenderer>();
    }
}
```

### 6.8.2 数据流向

```
Graphic（C# 层）
  │ 调用 SetVerticesDirty()
  │ → Rebuild() → UpdateGeometry()
  │ → OnPopulateMesh(vh) → vh.FillMesh(mesh)
  │ → canvasRenderer.SetMesh(mesh)   ← 将 Mesh 推入 CanvasRenderer
  │ → canvasRenderer.SetMaterial(mat, tex)  ← 将材质推入 CanvasRenderer
  ▼
CanvasRenderer（C# 层封装，内部调用引擎 native 方法）
  │ SetMesh → 引擎层保存 Mesh 引用
  │ SetMaterial → 引擎层保存 Material 引用
  ▼
Canvas.BuildBatch()（引擎 native 方法）
  │ 遍历所有 CanvasRenderer
  │ 合并同材质+同纹理 Mesh
  ▼
GPU DrawCall
```

### 6.8.3 关键区别

| 维度 | Graphic | CanvasRenderer |
|------|---------|---------------|
| 所在程序集 | C#（UnityEngine.UI） | C# 封装 + 引擎 native |
| 主要职责 | 生成顶点数据、管理 Dirty 标记 | 存储 Mesh 和 Material、参与合批 |
| 是否可序列化 | 是（Component） | 是（Component） |
| 直接操作对象 | VertexHelper、m_WorkerMesh | Mesh、Material |
| 生命周期 | 开发者可控制（通过继承） | 引擎内部管理 |

---

## 6.9 自定义 Graphic 的实践路径

基于本章的分析，你可以通过继承 `MaskableGraphic` 来创建自定义 UI 组件：

```csharp
using UnityEngine;
using UnityEngine.UI;

public class CircleGraphic : MaskableGraphic {
    [SerializeField] private int segments = 32;
    [SerializeField] private float radius = 50f;

    protected override void OnPopulateMesh(VertexHelper vh) {
        vh.Clear();

        // 圆心顶点
        vh.AddVert(Vector3.zero, color, Vector2.one * 0.5f);

        // 圆周顶点
        float angleStep = 2 * Mathf.PI / segments;
        for (int i = 0; i <= segments; i++) {
            float angle = i * angleStep;
            float x = Mathf.Cos(angle) * radius;
            float y = Mathf.Sin(angle) * radius;
            float uvX = Mathf.Cos(angle) * 0.5f + 0.5f;
            float uvY = Mathf.Sin(angle) * 0.5f + 0.5f;
            vh.AddVert(new Vector3(x, y, 0), color, new Vector2(uvX, uvY));
        }

        // 三角形（扇形）
        for (int i = 0; i < segments; i++) {
            vh.AddTriangle(0, i + 1, i + 2);
        }
    }

    // 当 radius 或 segments 变化时通知系统重建
    public void SetRadius(float newRadius) {
        radius = newRadius;
        SetVerticesDirty();
        SetLayoutDirty();  // 尺寸变化也可能需要重新布局
    }

    public void SetSegments(int newSegments) {
        segments = newSegments;
        SetVerticesDirty();
    }
}
```

这个圆形控件的渲染链路：

```
CircleGraphic.SetRadius(100)
  → SetVerticesDirty()
  → CanvasUpdateRegistry 注册 → 下一帧 PreRender
  → Graphic.Rebuild() → UpdateGeometry()
  → DoMeshGeneration() → OnPopulateMesh(vh)
  → vh 中添加 segments+1 个顶点 + segments 个三角形
  → vh.FillMesh(m_WorkerMesh)
  → canvasRenderer.SetMesh(m_WorkerMesh)
  → Canvas.BuildBatch() → GPU 绘制
```

---

## 6.10 本章总结

### 核心要点

1. **Graphic 是 UGUI 可视 UI 的抽象基类**。所有能被渲染的 UI 元素（Image、RawImage、Text）都继承自 Graphic。

2. **类层级**：`UIBehaviour → Graphic（ICanvasElement / IMaterialModifier / ILayoutElement）→ MaskableGraphic → Image / RawImage / Text`。每一层添加了一种能力。

3. **渲染链路**：RectTransform（空间结构）→ Graphic / OnPopulateMesh（几何生成）→ CanvasRenderer.SetMesh（合批提交）→ GPU（绘制）。数据从 UI 组件逐级转换为 GPU 可理解的 Mesh 格式。

4. **OnPopulateMesh 是核心虚方法**。所有几何生成逻辑都在这里，子类通过重写此方法生成自定义的顶点数据。

5. **Dirty 标记机制**：`SetVerticesDirty` / `SetMaterialDirty` / `SetLayoutDirty` 三种标记分别控制顶点、材质、布局的延迟重建，通过 `CanvasUpdateRegistry` 统一调度。

6. **Rebuild 流程**：`Rebuild(PreRender)` → `UpdateGeometry()`（`OnPopulateMesh` + `SetMesh`）→ `UpdateMaterial()`（`SetMaterial` + `SetTexture`）。

7. **VertexHelper 是顶点构建工具**：管理位置、UV、颜色、法线、切线、三角形索引等数据，最终通过 `FillMesh` 写入 Mesh。

### 学习方法建议

想要真正掌握 Graphic 系统，建议在阅读完本章后打开 UGUI 源码（Unity 安装目录下的 `UnityEngine.UI` 包），依次查看以下文件：

```
1. Graphic.cs       → SetVerticesDirty() / Rebuild() / UpdateGeometry() / OnPopulateMesh()
2. MaskableGraphic.cs → GetModifiedMaterial() 理解 Stencil 材质如何注入
3. Image.cs         → OnPopulateMesh() 的四种 ImageType 实现
4. RawImage.cs      → 对比 Image 的差异（没有九宫格，只有简单的 UV 设置）
5. Text.cs          → OnPopulateMesh() 的逐字符 Quad 生成
6. VertexHelper.cs  → AddVert() / AddTriangle() / FillMesh() / Clear()
```

### 本章知识点自查清单

- [ ] Graphic 实现了哪三个接口？各自的职责是什么？
- [ ] 渲染链路中每个环节的数据形态是什么？
- [ ] 三种 Dirty 标记分别对应什么变化？它们如何被调度？
- [ ] Rebuild 流程中两个阶段（Geometry / Material）的顺序为什么重要？
- [ ] OnPopulateMesh 的参数 VertexHelper 管理了哪些数据？
- [ ] 一个 Image 生成 Quad 时，需要几个顶点？几个三角形？几个索引？
- [ ] CanvasRenderer 和 Graphic 的分工是什么？
- [ ] 自定义 UI 组件时，需要重写哪个方法？修改属性后调用哪个方法通知重建？
