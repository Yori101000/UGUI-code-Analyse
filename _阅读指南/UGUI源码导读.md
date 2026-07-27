# UGUI 源码导读

> 本文档不是章节总结，而是**源码阅读指南**——告诉你每个知识点对应 UGUI 源码中的哪个文件、哪个类、哪个方法，以及怎么看懂它们。建议配合 UGUI 源码（Unity-Technologies/uGUI）一起阅读，源码位于 `UnityEngine.UI/UI/Core/` 目录下。

---

## 如何获取 UGUI 源码

```
方式一：GitHub（推荐）
https://github.com/Unity-Technologies/uGUI
分支选对应你的 Unity 版本

方式二：Unity 包管理器
Window → Package Manager → Unity UI → 在文件管理器中显示
```

源码核心目录结构：

```
UnityEngine.UI/UI/Core/
├── Canvas.cs              ← Canvas 组件
├── CanvasRenderer.cs       ← CanvasRenderer 组件（C# 层封装）
├── Graphic.cs              ← Graphic 抽象基类
├── MaskableGraphic.cs      ← 支持 Mask 裁剪的 Graphic
├── Image.cs                ← Image 组件
├── RawImage.cs             ← RawImage 组件
├── Text.cs                 ← 传统 Text 组件
├── VertexHelper.cs         ← 顶点构建工具类
├── Layout/
│   ├── LayoutGroup.cs      ← 布局基类
│   ├── LayoutRebuilder.cs  ← 布局重建执行器
│   ├── ContentSizeFitter.cs
│   ├── HorizontalOrVerticalLayoutGroup.cs
│   └── GridLayoutGroup.cs
├── Mask.cs                 ← Stencil Mask 组件
├── RectMask2D.cs           ← 矩形裁剪组件
├── EventSystem.cs          ← 事件系统核心
├── ExecuteEvents.cs        ← 事件分发
├── InputModules/
│   ├── BaseInputModule.cs
│   ├── StandaloneInputModule.cs
│   └── InputSystemUIInputModule.cs
├── Raycasters/
│   ├── GraphicRaycaster.cs
│   ├── PhysicsRaycaster.cs
│   └── Physics2DRaycaster.cs
├── Selectable.cs           ← 交互组件基类
├── Button.cs
├── ScrollRect.cs
└── StencilMaterial.cs      ← Stencil 材质管理
```

---

# 第一部分：基础认知（第1-4章）

## 第1章 UGUI 整体架构

### 源码文件

| 文件 | 说明 |
|------|------|
| `Canvas.cs` | 渲染入口，所有 UI 必须挂在 Canvas 下 |
| `CanvasRenderer.cs` | 每个 UI 元素对应一个，保存渲染数据 |
| `Graphic.cs` | 所有可视 UI 的抽象基类 |
| `RectTransform.cs`（引擎内置） | UI 空间定义（非开源，但可通过 API 理解） |

### 核心概念 vs 源码映射

**"保留模式（Retained Mode）" vs IMGUI 的"即时模式"**

IMGUI 在 `OnGUI()` 中每帧执行绘制代码；UGUI 则是"声明 UI 结构 → 系统在状态变化时自动更新"。

```csharp
// IMGUI：每帧绘制
void OnGUI() {
    GUI.Button(new Rect(10,10,100,50), "Click");
}

// UGUI：声明结构，系统维护
// 将 Button 拖到场景中 → 系统只在状态变化时重建
```

对应源码：**Graphic.cs** 中的 `SetVerticesDirty()` / `SetMaterialDirty()` / `SetLayoutDirty()`——这些方法不立即执行重建，而是打上标记，等 Canvas 统一调度。

### 源码阅读入口

打开 `Graphic.cs`，搜索以下内容建立初步认知：

```
1. SetVerticesDirty()     → 观察它调用了 CanvasUpdateRegistry.Register...
2. SetMaterialDirty()     → 同上
3. Rebuild()              → 观察它分两个阶段：UpdateGeometry + UpdateMaterial
4. OnPopulateMesh()       → 空的虚方法，子类实现
```

### 如何验证

```
创建一个 Button → Game 视图 Stats 观察 Batch/Verts
修改 Button 颜色 → 观察 Stats 变化
打开 Profiler → 观察 Canvas.BuildBatch 的触发时机
```

---

## 第2章 UI 的本质：从图形到网格

### 源码文件

| 文件 | 说明 |
|------|------|
| `VertexHelper.cs` | 顶点构建工具，管理 List<UIVertex> |
| `Graphic.cs` | OnPopulateMesh 的入口 |
| `Image.cs` | Image 的 OnPopulateMesh 实现 |

### 核心概念 vs 源码映射

**"UI 在 GPU 眼中是 Mesh，不是按钮"**

UGUI 中所有 UI 元素最终都生成 Mesh（顶点 + 三角形 + UV + 颜色）。

**UIVertex 结构体**（定义在 `VertexHelper.cs` 中）：

```csharp
public struct UIVertex {
    public Vector3 position;    // 顶点位置
    public Vector3 normal;      // 法线（UI 通常无关）
    public Vector4 tangent;     // 切线（TMP 用到）
    public Color32 color;       // 顶点颜色
    public Vector4 uv0;         // 主纹理 UV
    public Vector4 uv1;         // 额外 UV（PositionAsUV1 用）
    public Vector4 uv2;         // 额外 UV
    public Vector4 uv3;         // 额外 UV
}
```

**关键认知**：一个 Image = 4 个 UIVertex（4 个角）+ 2 个三角形（6 个索引）。

### 源码阅读路径

```
Image.cs → OnPopulateMesh()
  → 根据 Image.type（Simple/Sliced/Tiled/Filled）生成不同顶点
  → 使用 VertexHelper 添加顶点和三角形
  → vh.AddVert(position, color, uv)
  → vh.AddTriangle(a, b, c)
```

### 如何验证

```csharp
// 在某个 Image 的 Update 中观察顶点变化
void Update() {
    Debug.Log($"顶点数: {GetComponent<CanvasRenderer>().mesh.vertexCount}");
}
```

---

## 第3章 RectTransform 核心机制

### 源码文件

RectTransform 的 C# 源码是引擎内置的，不在 UGUI 仓库中。但可以通过 `RectTransformUtility.cs`（位于 `UI/Core/`）和 `LayoutRebuilder.cs` 间接理解。

| 文件 | 说明 |
|------|------|
| `RectTransformUtility.cs` | RectTransform 的工具方法 |
| `LayoutRebuilder.cs` | 布局重建时调用 RectTransform 的方法 |

### 核心概念 vs API 映射

**锚点（Anchor）**：`RectTransform.anchorMin` / `anchorMax`
**轴心（Pivot）**：`RectTransform.pivot`
**矩形信息**：`RectTransform.rect`（本地坐标）

**关键公式**（引擎内部计算，可从 API 行为反推）：

```
anchoredPosition = 父级矩形上的锚点位置 + 偏移
sizeDelta = 当前尺寸 - 锚点定义的尺寸
```

### 源码阅读路径

```
LayoutRebuilder.cs → 搜索 LayoutRebuilder.ValidateLayout
  → 观察它如何调用 rectTransform 的属性
  → 理解布局系统与 RectTransform 的交互
```

---

## 第4章 UI 更新与重建系统

### 源码文件

| 文件 | 说明 |
|------|------|
| `CanvasUpdateRegistry.cs` | **核心调度器**——这是整个 UGUI 更新系统的引擎 |
| `Graphic.cs` | Rebuild 方法的实现 |
| `LayoutRebuilder.cs` | Layout 重建的执行器 |

### 核心概念 vs 源码映射

**延迟重建（Deferred Rebuild）**

```csharp
// CanvasUpdateRegistry 的核心逻辑（简化）
public class CanvasUpdateRegistry {
    private List<ICanvasElement> m_LayoutRebuildQueue;
    private List<ICanvasElement> m_GraphicRebuildQueue;

    void OnCanvasWillRenderCanvases() {
        PerformUpdate();
    }

    public void PerformUpdate() {
        // 阶段 0-2：Layout Rebuild
        for (int i = 0; i <= 2; i++)
            for each element in m_LayoutRebuildQueue
                element.Rebuild((CanvasUpdate)i);

        // 阶段 3-5：Graphic Rebuild
        for (int i = 3; i <= 5; i++)
            for each element in m_GraphicRebuildQueue
                element.Rebuild((CanvasUpdate)i);
    }
}
```

**Dirty 标记传播**：

```csharp
// Graphic.cs 中
public virtual void SetVerticesDirty() {
    m_VertsDirty = true;
    CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
    // → 把自己加入 m_GraphicRebuildQueue
}
```

**完整链路**：

```
修改 text → Text.SetVerticesDirty()
  → CanvasUpdateRegistry.Register...
    → 下一帧 Canvas.SendWillRenderCanvases 触发
      → CanvasUpdateRegistry.PerformUpdate()
        → Graphic.Rebuild(PreRender)
          → UpdateGeometry()
            → OnPopulateMesh(vh)    ← 子类实现
            → canvasRenderer.SetMesh(mesh)
          → UpdateMaterial()
            → canvasRenderer.SetMaterial(mat, tex)
```

### 源码阅读路径

```
CanvasUpdateRegistry.cs → 完整看完
  → 理解 m_LayoutRebuildQueue 和 m_GraphicRebuildQueue 的区别
  → 理解 PerformUpdate 的阶段划分
Graphic.cs → Rebuild() / UpdateGeometry() / UpdateMaterial()
LayoutRebuilder.cs → Rebuild() / LayoutRebuilder.MarkLayoutForRebuild()
```

### 关键问题

**Q: 为什么每帧修改 `text.text = hp.ToString()` 会导致卡顿？**
A: 每帧调用 `SetVerticesDirty()` → 每帧加入 GraphicRebuild 队列 → 每帧触发 `OnPopulateMesh` → 每帧设置 Mesh → 严重 CPU 开销。

---

# 第二部分：渲染链路（第5-9章）

## 第5章 Graphic 系统

### 源码文件

| 文件 | 说明 |
|------|------|
| `Graphic.cs` | 抽象基类，核心渲染逻辑 |
| `MaskableGraphic.cs` | 继承 Graphic，添加 Mask 支持 |
| `Image.cs` | Image 的 OnPopulateMesh 实现 |
| `VertexHelper.cs` | 顶点构建工具 |
| `GraphicRegistry.cs` | Graphic ↔ Canvas 的映射关系 |

### 核心概念 vs 源码映射

**Graphic 类层级**：

```
UIBehaviour
  └── Graphic（实现 ICanvasElement, IMaterialModifier, ILayoutElement）
       └── MaskableGraphic（实现 IClippable, IMaskable）
            ├── Image
            ├── RawImage
            └── Text
```

**OnPopulateMesh**——每个子类重写此方法生成自己的网格：

```csharp
// Image.cs（简化）
protected override void OnPopulateMesh(VertexHelper vh) {
    vh.Clear();
    // 根据 Image.type 生成 4 个顶点和三角形
    AddQuad(vh, vertices, colors, uvs);
}
```

### 源码阅读路径

```
1. Graphic.cs → m_VertsDirty → SetVerticesDirty()
   → Rebuild(CanvasUpdate.PreRender) → UpdateGeometry()
2. Image.cs → OnPopulateMesh(vh) → 四种 ImageType 的顶点生成逻辑
3. VertexHelper.cs → AddVert / AddTriangle / AddUIVertexStream
```

---

## 第6章 Canvas 系统

### 源码文件

| 文件 | 说明 |
|------|------|
| `Canvas.cs` | Canvas 组件，核心调度入口 |
| `CanvasScaler.cs` | 分辨率适配 |

### 核心概念 vs 源码映射

**Canvas 的三种渲染模式**：对应 `Canvas.renderMode` 枚举。

**Canvas 的批处理**——核心方法 `Canvas.BuildBatch()` 是 **native 方法**（C++ 实现），C# 层看不到源码。但可以通过观察行为来理解：

```csharp
// Canvas.cs（C# 可见部分）
public class Canvas : Behaviour {
    public RenderMode renderMode;     // Overlay / Camera / WorldSpace
    public int sortingOrder;           // 同一 Canvas 层级的排序
    public float scaleFactor;          // CanvasScaler 修改此值

    // Native 方法（看不到实现）
    [NativeMethod] private extern void BuildBatch();
    [NativeMethod] private extern void SendWillRenderCanvases();
}
```

### 关键理解

`Canvas.BuildBatch()` 的内部逻辑（根据反推和官方文档）：
1. 遍历当前 Canvas 下的所有 `CanvasRenderer`
2. 读取每个 Renderer 的 Mesh、Material、Texture
3. 按渲染状态（材质/纹理）分组，同组合并为一个 DrawCall
4. 提交到 GPU Command Buffer

---

## 第7章 CanvasRenderer 机制

### 源码文件

| 文件 | 说明 |
|------|------|
| `CanvasRenderer.cs` | C# 层封装，核心方法几乎都是 native 调用 |

```csharp
public class CanvasRenderer : MonoBehaviour {
    public void SetMesh(Mesh mesh);          // 设置顶点数据
    public void SetMaterial(Material mat, Texture tex); // 设置材质
    public void SetColor(Color color);       // 设置颜色
    public void EnableRectClipping(Rect rect); // RectMask2D 裁剪
    public void DisableRectClipping();       // 关闭裁剪
    public bool cull { get; set; }           // RectMask2D 用于剔除
    public int absoluteDepth { get; }        // 用于排序
}
```

### 核心概念 vs 源码映射

**Graphic → CanvasRenderer → Canvas 的关系**：

```
Graphic（生成数据）
  → canvasRenderer.SetMesh(mesh)     ← Graphic.UpdateGeometry 中调用
  → canvasRenderer.SetMaterial(mat)  ← Graphic.UpdateMaterial 中调用
    → Canvas.BuildBatch() 从 CanvasRenderer 读取数据
      → 合并 → DrawCall
```

---

## 第8章 UI 批处理与 DrawCall

### 源码文件

无直接源码（BuildBatch 是 native 方法），但可通过 `Frame Debugger` 反推行为。

### 核心概念

**合批条件**（可在 Frame Debugger 中验证）：

```
必须完全一致才能合并：
  ├── Material（同一个实例）
  ├── Texture（同一张图）
  ├── Shader + Keywords
  └── Stencil 状态
中断批次的因素：
  ├── 材质变化       → Frame Debugger: Different Material
  ├── 纹理变化       → Different Texture
  ├── Mask 嵌套      → Different Stencil
  └── RectMask2D    → Different Clipping Rect
```

### 如何验证

```
1. 创建一个 Canvas，放两个 Image（同一图集）
2. 打开 Frame Debugger（Window → Analysis → Frame Debugger）
3. 观察两个 Image 是否在同一个 DrawCall 中
4. 改变其中一个 Image 的材质 → 观察 Batch 分裂
```

---

## 第9章 UI 与渲染管线

### 本章是全链路总结，无新源码

---

# 第三部分：三大子系统（第10-12章）

## 第10章 Layout 布局系统

### 源码文件

| 文件 | 说明 |
|------|------|
| `LayoutGroup.cs` | 布局基类，实现 ILayoutGroup |
| `HorizontalOrVerticalLayoutGroup.cs` | 水平/垂直布局通用逻辑 |
| `HorizontalLayoutGroup.cs` | 水平布局 |
| `VerticalLayoutGroup.cs` | 垂直布局 |
| `GridLayoutGroup.cs` | 网格布局 |
| `LayoutRebuilder.cs` | 布局重建 |
| `ContentSizeFitter.cs` | 自适应尺寸 |
| `LayoutElement.cs` | 布局元素配置 |
| `LayoutUtility.cs` | 布局计算工具 |

### 核心概念 vs 源码映射

**布局两阶段计算**（`ILayoutController` 接口）：

```csharp
public interface ILayoutController {
    void SetLayoutHorizontal();  // 阶段一：计算高度
    void SetLayoutVertical();    // 阶段二：计算宽度（依赖高度计算结果）
}
```

**LayoutRebuilder 的工作机制**：

```csharp
// LayoutRebuilder.cs（简化）
public class LayoutRebuilder : ICanvasElement {
    public static void MarkLayoutForRebuild(RectTransform rect) {
        // 向上遍历找到最近的 LayoutGroup
        // 注册到 CanvasUpdateRegistry
    }

    public void Rebuild(CanvasUpdate executing) {
        // 执行布局计算
        PerformLayoutCalculation();
        PerformLayoutControl();
    }

    private void PerformLayoutControl() {
        // 调用父级 LayoutGroup.SetLayoutHorizontal/Vertical
    }
}
```

### 源码阅读路径

```
1. LayoutGroup.cs → CalculateLayoutInputHorizontal/ Vertical
2. HorizontalLayoutGroup.cs → SetLayoutHorizontal/ SetLayoutVertical
3. LayoutRebuilder.cs → Rebuild() → MarkLayoutForRebuild()
4. ContentSizeFitter.cs → SetLayoutHorizontal/Vertical
```

### 关键问题

**ContentSizeFitter 的双向依赖**：ContentSizeFitter 依赖 LayoutGroup 的排布结果来计算自己的尺寸，但 LayoutGroup 又需要子元素的尺寸来做排布——这会导致同一帧内多次递归。

---

## 第11章 EventSystem 事件系统

### 源码文件

| 文件 | 说明 |
|------|------|
| `EventSystem.cs` | 核心调度器，每帧 Update |
| `BaseInputModule.cs` | 输入模块基类 |
| `StandaloneInputModule.cs` | 鼠标/键盘输入处理 |
| `TouchInputModule.cs` | 触摸输入处理 |
| `InputSystemUIInputModule.cs` | 新输入系统适配 |
| `ExecuteEvents.cs` | 事件分发核心，委托表 |
| `BaseRaycaster.cs` | 射线检测基类 |
| `GraphicRaycaster.cs` | UI 射线检测（矩形范围） |
| `PhysicsRaycaster.cs` | 3D 物理射线检测 |
| `Physics2DRaycaster.cs` | 2D 物理射线检测 |
| `PointerEventData.cs` | 指针事件数据 |
| `RaycasterManager.cs` | Raycaster 全局注册表 |

### 核心概念 vs 源码映射

**ExecuteEvents 的委托表机制**——这是 UGUI 事件系统的核心设计，**不依赖反射**：

```csharp
// ExecuteEvents.cs（核心设计）
public static class ExecuteEvents {
    // 事件类型 → 委托的静态映射表
    public static EventFunction<IPointerClickHandler> pointerClickHandler =
        (h, d) => h.OnPointerClick(d);

    public static EventFunction<IPointerDownHandler> pointerDownHandler =
        (h, d) => h.OnPointerDown(d);

    public static EventFunction<IBeginDragHandler> beginDragHandler =
        (h, d) => h.OnBeginDrag(d);

    // 事件执行方法
    public static bool Execute<T>(GameObject target, BaseEventData data,
        EventFunction<T> functor) where T : IEventSystemHandler {
        // 1. GetComponents<T>() 获取组件
        // 2. 遍历调用 functor(component, data)
        // 3. 有实现者时返回 true
    }

    // 事件冒泡
    public static GameObject ExecuteHierarchy<T>(GameObject root,
        BaseEventData data, EventFunction<T> functor) where T : IEventSystemHandler {
        // 从 target 开始向上遍历 transform.parent
        // 第一个实现接口的组件响应事件
    }
}
```

**事件链路**：

```
EventSystem.Update()
  → m_CurrentInputModule.Process()
    → StandaloneInputModule.ProcessMouseEvent()
      → eventSystem.RaycastAll()       ← 获取所有 Raycaster 执行检测
        → EventSystem.RaycastAll()
          → RaycasterManager.GetRaycasters()
            → 每个 Raycaster.Raycast()
              → 结果合并排序
      → ExecuteEvents.ExecuteHierarchy()  ← 分发事件
```

### 源码阅读路径

```
1. ExecuteEvents.cs → 完整看完，理解 delegate 表的设计
2. EventSystem.cs → Update() → TickModules()
3. StandaloneInputModule.cs → Process() → ProcessMouseEvent()
4. GraphicRaycaster.cs → Raycast()
   → 注意它和 Graphic.Raycast() 的区别
5. RaycasterManager.cs → AddRaycaster / GetRaycasters
```

### 勘误对照

```
原文错误：timeScale=0 时 UI 输入仍有效
→ EventSystem.Update() 是标准 MonoBehaviour Update，受 timeScale 控制
→ timeScale=0 时 EventSystem 不执行，UI 输入完全失效

原文错误：Physics2DRaycaster 直接继承 BaseRaycaster
→ public class Physics2DRaycaster : PhysicsRaycaster
→ 它继承自 PhysicsRaycaster，不是 BaseRaycaster
```

---

## 第12章 UI 资源与图集系统

### 源码文件

UGUI 本身的 SpriteAtlas 系统源码不在 UI/Core 目录中，而是在 `UnityEngine.UIModule` 中。但以下文件与图集使用直接相关：

| 文件 | 说明 |
|------|------|
| `Graphic.cs` | `mainTexture` 属性—最终用于合批的纹理 |
| `Image.cs` | 使用 Sprite 的入口 |
| `RawImage.cs` | 使用 Texture 的入口 |

### 关键理解

**SpriteAtlas 的工作原理**（非开源，从行为反推）：

```
构建时：多个 Sprite → 合并到一张 Atlas Texture
运行时：
  Sprite.GetTexture() 返回的不再是原始 Texture
  Sprint.GetUV() 返回 Atlas 中的 UV 区域
  → Image 无感知地使用 Atlas Texture 渲染
```

---

# 第四部分：高级机制（第13、15-18章）

## 第13章 Mask 与裁剪机制

### 源码文件

| 文件 | 说明 |
|------|------|
| `Mask.cs` | Stencil Mask 组件 |
| `RectMask2D.cs` | 矩形裁剪组件 |
| `MaskableGraphic.cs` | 支持裁剪的 Graphic 基类 |
| `StencilMaterial.cs` | Stencil 材质管理 |
| `MaskUtilities.cs` | Mask 工具方法 |
| `ClipperRegistry.cs` | RectMask2D 的裁剪注册器 |

### 核心概念 vs 源码映射

**Mask 的工作机制**：

```csharp
// Mask.cs（简化）
public class Mask : UIBehaviour, IMaterialModifier {
    public virtual Material GetModifiedMaterial(Material baseMaterial) {
        // 计算当前 Mask 的 Stencil 深度
        int stencilDepth = MaskUtilities.GetStencilDepth(transform, maskCanvas);
        // 返回带 Stencil 参数的新材质
        return StencilMaterial.Add(baseMaterial, stencilID, ...);
    }

    protected void OnEnable() {
        // 通知子树重新计算 Mask 状态
        MaskUtilities.NotifyStencilStateChanged(this);
    }
}
```

**StencilMaterial 的缓存机制**：

```csharp
// StencilMaterial.cs
public static class StencilMaterial {
    private class StencilMaterialEntry {
        public Material baseMat;    // 原始材质
        public Material customMat;  // 带 Stencil 参数的新材质
        public int stencilID;       // Stencil 参考值
        public StencilOp operation; // 写入操作（Replace/Keep）
        public CompareFunction compareFunction; // 比较函数
    }

    private static List<StencilMaterialEntry> m_List = new();

    public static Material Add(Material baseMat, int stencilID, ...) {
        // 查找缓存：相同参数返回同一个实例
        // 不同参数创建新实例并缓存
    }
}
```

**RectMask2D 的工作机制**：

```csharp
// RectMask2D.cs（简化）
public class RectMask2D : UIBehaviour, IClipper {
    private List<IClippable> m_ClipTargets = new();

    public virtual void PerformClipping() {
        Rect clipRect = GetClipRect();  // 从 RectTransform 计算
        foreach (var target in m_ClipTargets) {
            if (clipRect.Overlaps(targetBounds))
                target.SetClipRect(clipRect, true);  // → EnableRectClipping
            else
                target.Cull(clipRect, true);  // → cull = true
        }
    }
}
```

### 源码阅读路径

```
1. Mask.cs → GetModifiedMaterial() / OnEnable / OnDisable
2. StencilMaterial.cs → Add() 的缓存逻辑 ← 理解 Mask 断批的原因
3. MaskableGraphic.cs → GetModifiedMaterial() (Mask 和 RectMask2D 都改这里)
4. RectMask2D.cs → PerformClipping() / AddClippable / RemoveClippable
5. ClipperRegistry.cs → Cull() 的执行时机
```

### 关键问题

**为什么 Mask 会断批？**
→ `StencilMaterial.Add()` 为不同 Stencil 参数创建不同 Material 实例
→ 合批要求 Material 实例一致 → 不同实例 = 不同 Batch

**相同深度的 Mask 子节点之间能合批吗？**
→ 能，因为它们的 Stencil 参数完全相同 → 共享同一个 Material 实例

---

## 第15章 文本渲染系统

### 源码文件

| 文件 | 说明 |
|------|------|
| `Text.cs` | 传统 UGUI Text 组件（已标记为 Legacy） |
| `FontData.cs` | 字体参数序列化 |
| — | TextGenerator（引擎内置，C# 侧调用但不开源） |

### 核心概念 vs 源码映射

**Text 的 OnPopulateMesh——每个字符生成一个 Quad**：

```csharp
// Text.cs（简化）
protected override void OnPopulateMesh(VertexHelper vh) {
    // TextGenerator 是引擎内置的排版引擎
    TextGenerationSettings settings = GetGenerationSettings(rectTransform.rect.size);
    cachedTextGenerator.Populate(text, settings);  // 生成顶点数据

    IList<UIVertex> verts = cachedTextGenerator.verts;
    // 每个字符生成 4 顶点 + 6 索引
    for (int i = 0; i < verts.Count; i += 4) {
        vh.AddVert(verts[i]);
        vh.AddVert(verts[i+1]);
        vh.AddVert(verts[i+2]);
        vh.AddVert(verts[i+3]);
        vh.AddTriangle(i, i+1, i+2);
        vh.AddTriangle(i+2, i+1, i+3);
    }
}
```

**Font.textureRebuilt**——图集更新后的级联反应：

```csharp
// 监听 font.textureRebuilt 事件
// 当字体图集发生变化时，所有使用该字体的 Text 全部重建
Font.textureRebuilt += (font) => {
    // 遍历所有使用该字体的 Text
    // 调用 SetVerticesDirty()
};
```

---

## 第16章 UI Mesh 扩展机制

### 源码文件

| 文件 | 说明 |
|------|------|
| `BaseMeshEffect.cs` | Mesh 效果基类 |
| `IMeshModifier.cs` | Mesh 修改接口 |
| `Shadow.cs` | 阴影效果 |
| `Outline.cs` | 描边效果（继承 Shadow） |
| `PositionAsUV1.cs` | 位置→UV1 效果 |

### 核心概念 vs 源码映射

**完整调用链**：

```
Graphic.Rebuild()
  → UpdateGeometry()
    → DoMeshGeneration()
      → OnPopulateMesh(vh)     ← 生成原始顶点
      → GetComponents<IMeshModifier>()  ← 获取所有 BaseMeshEffect
        → 按顺序调用 ModifyMesh(vh)    ← 链式修改
          → CanvasRenderer.SetMesh()   ← 提交最终 Mesh
```

**Shadow 的实现——顶点复制**：

```csharp
// Shadow.cs（简化）
public class Shadow : BaseMeshEffect {
    public override void ModifyMesh(VertexHelper vh) {
        List<UIVertex> verts = GetVerticesFromHelper(vh);  // 读取
        int count = verts.Count;

        for (int i = 0; i < count; i++) {
            UIVertex vt = verts[i];
            vt.position += effectDistance;  // 偏移
            vt.color = effectColor;         // 改色
            verts.Add(vt);                  // 追加（复制）
        }

        vh.Clear();
        vh.AddUIVertexTriangleStream(verts);
    }
}
```

性能影响：原始 4 顶点 → Shadow 后 8 顶点 → Outline 后 40 顶点（×5）。

### 源码阅读路径

```
1. BaseMeshEffect.cs → 接口定义 + OnEnable 时 SetVerticesDirty
2. Shadow.cs → ModifyMesh 完整实现
3. Outline.cs → 4 次调用 base 的 ApplyShadow
```

---

## 第17章 UI Shader 机制

### 源码文件

Unity 内置的 UI Shader 不在 C# 仓库中，位于引擎内置资源中。可以通过以下方式查看：

```
Unity Built-in Shaders → UI → UI-Default.shader
```

位置：
- Windows: `Unity安装目录/Data/Resources/BuiltinShaders/UI-Default.shader`
- 或通过 GitHub 搜索 `UnityBuiltinShaders`

### UI-Default.shader 逐行解读

```glsl
Shader "UI/Default" {
    Properties {
        [PerRendererData] _MainTex ("Sprite Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
    }

    SubShader {
        Tags { "Queue"="Transparent" "RenderType"="Transparent" "CanUseSpriteAtlas"="True" }

        Stencil {
            Ref [_Stencil]           // ← Material 参数，不是硬编码
            Comp [_StencilComp]
            Pass [_StencilOp]
        }

        Cull Off
        Lighting Off
        ZWrite Off                  // UI 不写深度
        Blend SrcAlpha OneMinusSrcAlpha  // 标准 Alpha 混合

        Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            struct appdata_t {
                float4 vertex   : POSITION;   // ← 来自 UIVertex.position
                float4 color    : COLOR;      // ← 来自 UIVertex.color
                float2 texcoord : TEXCOORD0;  // ← 来自 UIVertex.uv0
            };

            v2f vert(appdata_t v) {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.texcoord = v.texcoord;
                o.color = v.color * _Color;   // 顶点颜色 × Tint
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed4 color = tex2D(_MainTex, i.texcoord) * i.color;
                return color;                 // ← 核心：就这么一行
            }
            ENDCG
        }
    }
}
```

### Stencil 参数传递机制

```csharp
// Mask 通过 StencilMaterial.Add() 修改 Material 参数
// Shader 通过 [_Stencil] 读取 Material 参数
// 所以同一个 Shader 可以服务于不同 Stencil 状态的 UI
```

---

## 第18章 UI 特效实现

### 相关源码

Shader 实现见第 17 章，CPU 特效见第 16 章（BaseMeshEffect）。

| 效果 | 实现方式 | 源码文件 |
|------|---------|---------|
| 渐变 | Shader（lerp） | 自定义 Shader |
| 流光 | Shader（时间+UV） | 自定义 Shader |
| 灰化 | Shader（dot 加权） | 自定义 Shader |
| 描边 | Shader（8邻域采样）或 ModifyMesh | Outline.cs（CPU）/ 自定义 Shader（GPU）|
| 发光 | Shader（多重采样+权重） | 自定义 Shader |
| 溶解 | Shader（噪声+discard） | 自定义 Shader |
| 毛玻璃 | Shader（GrabPass+高斯模糊） | 自定义 Shader |

---

# 第五部分：性能与工具（第19、28章）

## 第19章 UI 特效与性能

### 源码文件

| 文件 | 说明 |
|------|------|
| `CanvasUpdateRegistry.cs` | Rebuild 调度器 |
| `Graphic.cs` | Dirty 标记 |
| `LayoutRebuilder.cs` | Layout 重建 |
| `StencilMaterial.cs` | 材质实例化计数器 |
| `MaskUtilities.cs` | Stencil 深度计算 |

### Profiler 中常见的函数对应源码

| Profiler 函数 | 对应源码 | 含义 |
|------|------|------|
| `Canvas.BuildBatch` | 引擎 native 方法 | Canvas 正在合批，顶点多了或变化频繁 |
| `LayoutRebuilder.Rebuild` | `LayoutRebuilder.cs` | Layout 正在递归计算 |
| `Graphic.Rebuild` | `Graphic.cs` | Graphic 正在重建顶点 |
| `Canvas.SendWillRenderCanvases` | `Canvas.cs` | 所有 UI 更新的入口 |

### 如何通过源码定位问题

```csharp
// 方法：在 SetVerticesDirty 中加日志（需要 clone 源码）
public virtual void SetVerticesDirty() {
    Debug.Log($"{name} SetVerticesDirty", this);  // ← 加这行
    m_VertsDirty = true;
    CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
}
```

---

## 第28章 调试与分析工具

### Frame Debugger 的使用

```
Window → Analysis → Frame Debugger → Enable
左侧：DrawCall 列表
右侧 Details：
  - Shader: UI/Default
  - Texture: AtlasName
  - Stencil Ref: 0
对比相邻两个 DrawCall 的 Details → 不同字段就是断批原因
```

### Profiler 的使用

```
Window → Analysis → Profiler → UI Module
关键指标：
  - Vertices: UI 总顶点数
  - Batches: DrawCall 数
  - SetVerticesDirty 调用次数
  - Layout Rebuild 次数
```

---

# 第六部分：工程实践（第25、26、29章）

## 第25章 UI 架构设计

### 无直接 UGUI 源码对应

本章是工程架构设计，非 UGUI 机制。核心是 MVP 模式 + UIManager + 生命周期标准化。可参考源码设计思路：

```csharp
// UIManager 的核心——对照本章理解
public interface IUIComponent {
    void OnCreate();     // 实例化后（一次）
    void OnBind(object data); // 绑定数据
    void OnClear();      // 回收前
}

public abstract class UIBase : MonoBehaviour {
    public virtual void OnCreate() { }
    public virtual void OnInit() { }
    public virtual void OnOpen(object param = null) { }
    public virtual void OnClose() { }
    public virtual void OnDestroyUI() { }
}
```

## 第26章 DOTween 原理与 UGUI 集成

### 无 UGUI 源码，DOTween 是第三方库

但与 UGUI 源码的接口关系：
- `CanvasGroup.DOFade()` → 修改 `CanvasGroup.alpha` → 不触发 Rebuild ✅
- `Graphic.DOFade()` → 修改 `Graphic.color.a` → 触发 `SetVerticesDirty()` ⚠️
- `RectTransform.DOSizeDelta()` → 触发 Layout Rebuild 🔴

## 第29章 综合案例分析

### 涉及源码

| 案例 | 对应源码文件 |
|------|-------------|
| 背包数据驱动 | `Graphic.cs`（SetVerticesDirty 触发刷新） |
| 虚拟列表 | `LayoutRebuilder.cs`（不用 LayoutGroup 避免 Layout Rebuild） |
| 弹窗系统 | 无直接 UGUI 源码，架构设计 |

---

# 附录：UGUI 源码阅读速查表

| 你想了解什么 | 读哪个文件 | 看什么方法 |
|------------|----------|----------|
| UI 怎么生成的 Mesh | `Graphic.cs` | `OnPopulateMesh` → 在 Image/Text 中看实现 |
| UI 何时重建 | `CanvasUpdateRegistry.cs` | `PerformUpdate` |
| 布局怎么工作 | `LayoutRebuilder.cs` | `MarkLayoutForRebuild`, `Rebuild` |
| LayoutGroup 怎么排子元素 | `HorizontalOrVerticalLayoutGroup.cs` | `SetLayoutHorizontal/Vertical` |
| 事件怎么分发 | `ExecuteEvents.cs` | `Execute`, `ExecuteHierarchy` |
| Mask 怎么裁剪 | `Mask.cs` | `GetModifiedMaterial` |
| | `StencilMaterial.cs` | `Add`（理解缓存→断批） |
| RectMask2D 怎么裁剪 | `RectMask2D.cs` | `PerformClipping`, `AddClippable` |
| Shadow/Outline 怎么实现 | `Shadow.cs`, `Outline.cs` | `ModifyMesh` |
| 什么打断 Batch | Frame Debugger | 对比相邻 DrawCall 的 Details |
| Canvas 怎么合批 | 引擎 native 方法 | `Canvas.BuildBatch`（不可见） |
