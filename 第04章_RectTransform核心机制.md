# 第4章 RectTransform 核心机制

> 本章对应原书结构中的第3章。RectTransform 是 UGUI 空间系统的基石，定义了 UI 元素如何在父级 Canvas 中进行定位、对齐和尺寸计算。与普通 Transform 不同，RectTransform 引入了锚点（Anchor）和轴心（Pivot）两大机制，使得 UI 可以以声明式的方式描述"相对父级什么位置、以什么为基准"。

---

## 4.1 RectTransform vs Transform

### 4.1.1 Transform 的定位方式

传统的 Transform 组件使用 **绝对坐标** 描述物体的空间状态：

```
Transform.position   → 世界坐标下的位置
Transform.rotation   → 旋转
Transform.localScale → 缩放
```

子物体的 `localPosition` 是相对于父物体的偏移，没有"对齐方式"的概念。如果要实现"子元素在父元素的右边"这样的布局，需要手动计算坐标，且父级尺寸变化时所有子元素都需要重新计算。

### 4.1.2 RectTransform 新增的能力

RectTransform 继承自 Transform（`RectTransform : Transform`），在 position/rotation/scale 的基础上，增加了 UI 布局所需的专属参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| **anchorMin** | `Vector2` (0~1) | 锚点矩形左下角，归一化到父级宽高 |
| **anchorMax** | `Vector2` (0~1) | 锚点矩形右上角，归一化到父级宽高 |
| **pivot** | `Vector2` (0~1) | 轴心，归一化到自身宽高 |
| **anchoredPosition** | `Vector2` | 轴心相对于锚点的偏移 |
| **sizeDelta** | `Vector2` | 当前尺寸与锚点矩形尺寸的差值 |
| **offsetMin** | `Vector2` | 矩形左下角相对于锚点左下角的偏移 |
| **offsetMax** | `Vector2` | 矩形右上角相对于锚点右上角的偏移 |
| **rect** | `Rect` (只读) | 计算出的本地空间矩形 |

> **重要**：RectTransform 是 Unity 引擎内置组件，C# 源码位于 `UnityEngine.CoreModule`，**不在 UGUI 开源仓库中**。但它的所有公开 API 和行为都可以通过官方文档和运行时反射验证。

### 4.1.3 核心区别总结

```
Transform:  position (绝对) + rotation + scale
RectTransform: anchorMin/Max (相对父级对齐) + pivot (自身基准) + anchoredPosition (相对锚点偏移)
```

**坐标含义完全不同**：Transform 的坐标是"物体在世界/父空间中的位置"；RectTransform 的坐标是"物体的轴心相对于锚点的偏移"。

---

## 4.2 锚点系统（Anchor）

### 4.2.1 锚点的本质

锚点（Anchor）定义了 **子元素相对于父级矩形** 的对齐参考系。`anchorMin` 和 `anchorMax` 共同定义了一个"锚点矩形"（Anchor Rect）：

```
anchorMin = (xMin, yMin)   // 父级矩形左下角的归一化位置
anchorMax = (xMax, yMax)   // 父级矩形右上角的归一化位置
```

取值范围为 `[0, 1]`，表示相对于父级宽高的比例：
- `(0, 0)` = 父级左下角
- `(1, 1)` = 父级右上角
- `(0.5, 0.5)` = 父级中心

### 4.2.2 锚点预设（Presets）

Unity 编辑器提供了一系列锚点预设，覆盖了最常见的对齐模式：

**四点合一**（锚点聚为一点，即 anchorMin == anchorMax）：
- 子元素以该点为参考基准
- 子元素位置 = 父级某点 + anchoredPosition 偏移
- 父级尺寸变化时，子元素**保持与锚点的绝对距离**

**两点水平线**（锚点水平展开，垂直合并）：
- 子元素宽度 = 锚点宽度 + sizeDelta.x
- 父级宽度变化时，子元素宽度**自动拉伸**

**两点垂直线**（锚点垂直展开，水平合并）：
- 子元素高度 = 锚点高度 + sizeDelta.y
- 父级高度变化时，子元素高度**自动拉伸**

**四点矩形**（锚点完全展开，anchorMin=(0,0), anchorMax=(1,1)）：
- 子元素边缘锚定在父级边缘
- sizeDelta 表示子元素四边距父级四边的偏移
- 父级尺寸变化时，子元素**自动跟随缩放**

### 4.2.3 锚点与父级尺寸变化的关系

这是理解 RectTransform 的关键——**锚点决定了父级尺寸变化时子元素的行为**：

```csharp
// 伪代码：RectTransform 位置计算逻辑
// 锚点矩形在父级空间中的坐标：
float anchorLeft   = parentWidth  * anchorMin.x;
float anchorRight  = parentWidth  * anchorMax.x;
float anchorBottom = parentHeight * anchorMin.y;
float anchorTop    = parentHeight * anchorMax.y;

// 子元素最终矩形：
// rect.x = anchorLeft + offsetMin.x
// rect.y = anchorBottom + offsetMin.y
// rect.width  = (anchorRight - anchorLeft) + (offsetMax.x - offsetMin.x)
// rect.height = (anchorTop  - anchorBottom) + (offsetMax.y - offsetMin.y)
```

当 `anchorMin.x == anchorMax.x`（锚点合并），子元素的宽度固定，x 位置相对于锚点水平偏移固定量。当 `anchorMin.x != anchorMax.x`（锚点展开），子元素的宽度会随父级宽度变化而拉伸。

---

## 4.3 轴心（Pivot）

### 4.3.1 轴心的定义

Pivot 同样是一个归一化坐标 `(0~1, 0~1)`，定义在 **自身矩形空间** 内：

| pivot 取值 | 含义 |
|-----------|------|
| (0, 0) | 左下角 |
| (1, 1) | 右上角 |
| (0.5, 0.5) | 中心（默认值） |
| (0.5, 1) | 顶部居中 |

### 4.3.2 轴心的作用

轴心影响三个方面的行为：

**1. 定位（anchoredPosition 的基准）**

`anchoredPosition` 表示的是 **轴心到锚点的向量**。修改 pivot 不改变 UI 的最终渲染位置，但会改变 `anchoredPosition` 和 `offsetMin/offsetMax` 的值。

**2. 旋转（rotation 的基准点）**

UI 元素旋转时，以 pivot 为中心旋转。例如 pivot=(0, 0) 会使 UI 绕左下角旋转。

**3. 缩放（scale 的基准点）**

UI 元素缩放时，以 pivot 为中心进行缩放。

```
pivot=(0.5, 0.5) → 从中心缩放（最常见的用法）
pivot=(0, 0)     → 从左下角缩放
pivot=(1, 0)     → 从右下角缩放
```

---

## 4.4 矩形计算详解

### 4.4.1 坐标空间

RectTransform 涉及三个坐标空间：

```
父级 RectTransform 矩形空间
  └─ 锚点矩形（Anchor Rect）—— 由 anchorMin/anchorMax 在父级空间定义
       └─ 子元素矩形（Child Rect）—— 由 offsetMin/offsetMax 相对于锚点定义
            └─ 轴心（Pivot）—— 子元素自身矩形内的基准点
```

### 4.4.2 offsetMin 和 offsetMax

`offsetMin` 和 `offsetMax` 是最直观的边界偏移量：

```
offsetMin = rect.min - anchorMin（子元素左下角 - 锚点左下角）
offsetMax = rect.max - anchorMax（子元素右上角 - 锚点右上角）
```

- `offsetMin` 表示子元素左/下边界到锚点左下角的方向向量
- `offsetMax` 表示子元素右/上边界到锚点右上角的方向向量

当锚点四点合一时（anchorMin == anchorMax），offsetMin 和 offsetMax 的含义退化为"子元素四边到锚点的距离"。

### 4.4.3 anchoredPosition

`anchoredPosition` 是 **轴心到锚点的偏移向量**。当锚点聚为一点时：

```
anchoredPosition = pivotInParentSpace - anchorPoint
```

其中 `pivotInParentSpace` 是轴心在父级空间中的位置，`anchorPoint` 是锚点在父级空间中的位置。

**这是开发中最常用的定位属性**——设置 `anchoredPosition` 就能控制 UI 元素相对于锚点的位置。

当锚点展开时，anchoredPosition 的计算变得更复杂，因为此时"锚点"不是一个点而是一个矩形区域。Unity 内部取锚点矩形的中心作为参考点：

```
// 锚点矩形中心（在父级空间中）
Vector2 anchorCenter = new Vector2(
    parentRect.width  * (anchorMin.x + anchorMax.x) / 2,
    parentRect.height * (anchorMin.y + anchorMax.y) / 2
);
```

### 4.4.4 sizeDelta

`sizeDelta` 是 **当前矩形尺寸与锚点矩形尺寸的差值**：

```
sizeDelta = rect.size - anchorRect.size
```

理解 sizeDelta 的关键在于锚点状态：

- **锚点合并**（anchorMin == anchorMax）：锚点矩形尺寸为零，`sizeDelta = rect.size`，即子元素的实际尺寸
- **锚点展开**（anchorMin != anchorMax）：`sizeDelta` 表示子元素尺寸相对于锚点矩形的"盈余"或"缺口"
  - `sizeDelta = (0, 0)` → 子元素与锚点矩形等大
  - `sizeDelta.x > 0` → 子元素比锚点矩形宽
  
### 4.4.5 rect（只读属性）

`rect` 属性返回的是 **本地空间** 中的矩形，以 pivot 为原点：

```csharp
// RectTransform.rect 的等效计算（本地空间）
Rect rect = new Rect(
    -pivot.x * width,    // x 最小值（轴心到左边界的向量）
    -pivot.y * height,   // y 最小值（轴心到下边界的向量）
    width,               // 实际宽度
    height               // 实际高度
);
```

这意味着 `rect.x` 和 `rect.y` 通常是负数（当 pivot 在中心时），而 `rect.width` 和 `rect.height` 始终为正。**rect 在 UI 重建过程中被大量使用**，特别是在 Mask 裁剪、布局计算时。

---

## 4.5 与 Canvas、Layout 系统的协作

### 4.5.1 RectTransform 在 UI 渲染链中的位置

从 UI 元素到最终渲染的完整链路中，RectTransform 处于顶层：

```
Canvas           → 渲染入口，定义渲染模式
  └─ RectTransform（根） → 定义屏幕/世界空间中的基础矩形
       ├─ RectTransform（子1） → 锚点定位，相对父级对齐
       │    └─ Graphic          → 读取 rect 生成网格
       └─ RectTransform（子2） → 锚点定位
            └─ Text             → 读取 rect 进行文本排布
```

每个 UI 元素都同时挂载 RectTransform 和 Graphic（或 Graphic 的子类）。Graphic 在 `OnPopulateMesh()` 中读取 RectTransform 的 `rect` 属性，将矩形区域转换为 Mesh 顶点。

### 4.5.2 Canvas 与 RectTransform

Canvas 组件本身带有一个 RectTransform（根 RectTransform），它定义了 UI 的"世界空间基准矩形"：

- **ScreenSpaceOverlay** 模式：rect 尺寸等于屏幕分辨率
- **ScreenSpaceCamera** 模式：rect 尺寸等于渲染摄像机的视口大小
- **WorldSpace** 模式：rect 尺寸由用户手动设置或通过碰撞器定义

所有子 UI 元素的 RectTransform 都相对于 Canvas 的根 RectTransform 进行定位。

### 4.5.3 Layout 系统与 RectTransform 的关系

Layout 系统（详见第10章）是 RectTransform 的主要驱动者之一。布局计算的本质就是 **向 RectTransform 参数写入计算结果**：

```csharp
// Layout 系统最终做的工作（简化）
void SetChildAlongAxis(RectTransform rect, int axis, float pos, float size) {
    if (axis == 0) {  // 水平方向
        rect.SetInsetAndSizeFromParentEdge(Edge.Left, pos, size);
    } else {          // 垂直方向
        rect.SetInsetAndSizeFromParentEdge(Edge.Top, pos, size);
    }
}
```

LayoutGroup 的子类（Horizontal/Vertical/Grid）遍历子节点，计算每个子元素应当占据的位置和尺寸，然后通过 RectTransform API 将其写入：

```
HorizontalLayoutGroup.SetLayoutHorizontal()
  → 遍历子元素
  → 计算每个子元素的 x 位置和宽度
  → 调用 rectTransform.SetInsetAndSizeFromParentEdge()
  → 内部修改 offsetMin/offsetMax/sizeDelta
```

**关键理解**：Layout 系统不参与渲染，它只负责操纵 RectTransform 的参数。渲染侧（Graphic）读取的是 RectTransform 最终计算出的 `rect` 值。

### 4.5.4 ContentSizeFitter

ContentSizeFitter 是另一个 RectTransform 的驱动者，它反向工作——根据子内容（如 Text 的文本宽度）计算出父级 RectTransform 的尺寸，然后修改其 `sizeDelta`：

```
ContentSizeFitter.SetLayoutHorizontal()
  → 读取 ILayoutElement.preferredWidth（如 Text 的首选宽度）
  → 设置父级 RectTransform 的 sizeDelta.x
```

---

## 4.6 RectTransformUtility 工具类

`RectTransformUtility` 是 UGUI 源码中的静态工具类文件（位于 `UnityEngine.UI/UI/Core/`），提供了 RectTransform 空间变换的核心辅助方法。

### 4.6.1 文件位置与作用

| 项目 | 说明 |
|------|------|
| 文件 | `RectTransformUtility.cs` |
| 命名空间 | `UnityEngine` |
| 关键特性 | 全部为 `public static` 方法，包含对 RectTransform 的多种空间坐标转换 |
| 核心私有方法 | `PixelAdjustPoint()` / `PixelAdjustRect()` 等被内部组件调用 |

### 4.6.2 核心 API

**屏幕坐标与 UI 坐标的转换**（最常用的功能）：

```csharp
// 屏幕坐标 → RectTransform 本地坐标
public static bool ScreenPointToLocalPointInRectangle(
    RectTransform rect,          // 目标 RectTransform
    Vector2 screenPoint,         // 屏幕坐标点
    Camera cam,                  // 参考 Camera（Overlay 模式传 null）
    out Vector2 localPoint       // 输出：转换后的本地坐标
);

// 屏幕坐标 → 世界空间射线
public static bool ScreenPointToWorldPointInRectangle(
    RectTransform rect,
    Vector2 screenPoint,
    Camera cam,
    out Vector3 worldPoint
);

// 世界坐标 → RectTransform 本地坐标
public static bool WorldToPointInRectangle(
    RectTransform rect,
    Vector2 worldPoint,
    out Vector2 localPoint
);
```

**矩形碰撞检测**：

```csharp
// 判断屏幕坐标是否在 RectTransform 矩形内（考虑射线检测）
public static bool RectangleContainsScreenPoint(
    RectTransform rect,
    Vector2 screenPoint,
    Camera cam
);
```

**像素对齐**（用于抗锯齿优化）：

```csharp
// 将点对齐到像素边界，防止子像素渲染造成的模糊
public static Vector2 PixelAdjustPoint(
    Vector2 point,
    Transform elementTransform,
    Canvas canvas
);

// 获取像素对齐后的矩形
public static Rect PixelAdjustRect(
    RectTransform rectTransform,
    Canvas canvas
);
```

### 4.6.3 典型使用场景

**场景1：点击检测**

EventSystem 的 GraphicRaycaster 使用 `RectTransformUtility.RectangleContainsScreenPoint()` 判断点击是否落在 UI 元素范围内。

```csharp
// GraphicRaycaster.cs（简化）
var pointerPosition = eventData.position;
if (RectTransformUtility.RectangleContainsScreenPoint(
        graphic.rectTransform, pointerPosition, eventCamera)) {
    // 点击命中该 UI 元素
}
```

**场景2：拖拽时屏幕坐标转 UI 坐标**

```csharp
// 拖拽处理（简化）
RectTransformUtility.ScreenPointToLocalPointInRectangle(
    parentRectTransform,
    Input.mousePosition,
    uiCamera,
    out Vector2 localPoint
);
dragItem.anchoredPosition = localPoint;
```

### 6.4 跨 Canvas 空间转换

当不同 Canvas（如 Overlay + Camera 混合模式）之间需要坐标转换时，RectTransformUtility 通过 Camera 参数来处理：

- Overlay 模式传 `cam = null`，使用屏幕空间直接映射
- Camera 模式传对应 Camera，通过视口变换计算
- WorldSpace 模式将世界坐标投影到目标 RectTransform 的本地空间

---

## 4.7 关键 API 速查表

| API | 类型 | 说明 |
|-----|------|------|
| `anchorMin` | `Vector2` | 锚点矩形左下角，父级归一化坐标 |
| `anchorMax` | `Vector2` | 锚点矩形右上角，父级归一化坐标 |
| `pivot` | `Vector2` | 轴心，自身归一化坐标 |
| `anchoredPosition` | `Vector2` | 轴心相对于锚点的偏移 |
| `anchoredPosition3D` | `Vector3` | 三维版本的 anchoredPosition |
| `offsetMin` | `Vector2` | 矩形左下角相对锚点左下角的偏移 |
| `offsetMax` | `Vector2` | 矩形右上角相对锚点右上角的偏移 |
| `sizeDelta` | `Vector2` | 自身尺寸与锚点矩形尺寸的差值 |
| `rect` | `Rect`（只读） | 本地空间中的矩形，以 pivot 为原点 |
| `GetLocalCorners()` | 方法 | 获取本地空间的四个角 |
| `GetWorldCorners()` | 方法 | 获取世界空间的四个角（常用于调试） |
| `SetSizeWithCurrentAnchors()` | 方法 | 保持当前锚点设置下设置尺寸 |
| `SetInsetAndSizeFromParentEdge()` | 方法 | 从父级某一边设置边距和尺寸 |
| `RectTransformUtility` | 工具类 | 屏幕/世界/本地坐标的互相转换 |

---

## 4.8 常见误区

**误区1：anchoredPosition 就是 UI 的位置**

不完全正确。`anchoredPosition` 是"轴心相对于锚点的偏移"，只有当锚点聚为一点时，它才等价于"UI 在父级中的位置"。锚点展开时，anchoredPosition 的含义是相对于锚点矩形中心点的偏移。

**误区2：sizeDelta 就是 UI 的尺寸**

只有当锚点合并（anchorMin == anchorMax）时，`sizeDelta` 才等于 UI 的实际尺寸。锚点展开时，`sizeDelta` 表示"超出锚点矩形"的部分。始终使用 `rect.size` 获取实际尺寸更安全。

**误区3：RectTransformUtility 只能用于 UI**

虽然 RectTransformUtility 专为 UI 设计，但它的坐标转换方法（如 `ScreenPointToLocalPointInRectangle`）同样适用于 WorldSpace Canvas 下的 3D 坐标转换。

---

## 总结

RectTransform 是 UGUI 整个布局和渲染系统的空间基础：

1. **锚点系统**实现了 UI 的**自适应布局**——通过 anchorMin/anchorMax 声明子元素与父级的对齐关系，父级尺寸变化时自动调整
2. **轴心**定义了旋转/缩放的基准点，同时影响 anchoredPosition 的计算方式
3. **四组关键参数**（anchorMin/Max、pivot、offsetMin/offsetMax、anchoredPosition/sizeDelta）共同定义了 UI 元素的"弹性空间"
4. **RectTransformUtility** 提供了屏幕坐标、世界坐标、本地坐标之间的互转能力，是事件检测和拖拽交互的底层支撑
5. **Layout 系统**本质上是 RectTransform 的"自动操纵者"，通过计算向 RectTransform 参数写入布局结果
6. **Graphic 系统**读取 RectTransform 的 `rect` 属性生成渲染网格，形成从"空间定义"到"视觉表现"的完整链路

> RectTransform 的 C# 源码虽然不在 UGUI 仓库中，但它的所有公开 API 都可以通过 Unity 官方文档和运行时反射进行验证。在实际开发中，理解锚点和轴心的工作机制远比背诵 API 参数更为重要——它们是 UI 自适应布局的理论基础。
