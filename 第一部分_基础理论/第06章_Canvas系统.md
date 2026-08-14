# 第6章 Canvas系统

> Canvas是UGUI中承上启下的核心组件——所有UI元素必须挂载在Canvas下才能被渲染，它是连接UI数据（Graphic生成的网格）与渲染管线（GPU DrawCall）的核心枢纽。

---

## 6.1 Canvas的核心定位

在UGUI的三层架构中，Canvas处于**系统层**的最核心位置：

```
┌──────────────────────────────────────┐
│           组件层（Component Layer）    │
│  Graphic / Image / Text / Button     │
│  负责：生成UI顶点数据                  │
├──────────────────────────────────────┤
│           系统层（System Layer）       │
│  Canvas  ←  本章主角                  │
│  CanvasUpdateRegistry                │
│  EventSystem / LayoutRebuilder       │
│  负责：调度、合批、提交渲染            │
├──────────────────────────────────────┤
│           渲染层（Renderer Layer）     │
│  CanvasRenderer                      │
│  负责：暂存数据，桥接Native层          │
└──────────────────────────────────────┘
```

### 6.1.1 没有Canvas会怎样

任何UI组件（Image、Text、Button等）如果不在Canvas的子级，在编辑器中会看到这样的提示：

```
"XYZ is not active. This object is not under a Canvas."
```

原因很简单：**没有Canvas，Graphic生成的Mesh就没人去收集、合并、提交给GPU。**

### 6.1.2 Canvas在UGUI中的角色

| 角色 | 对应机制 | 说明 |
|------|---------|------|
| **容器** | Hierarchy父子关系 | Canvas收集其下所有CanvasRenderer |
| **排程器** | Sorting Layer / Order / Hierarchy | 决定UI层级的上下关系 |
| **模式控制器** | renderMode | 决定UI以何种方式接入渲染管线 |
| **批处理器** | Canvas.BuildBatch() | 遍历CanvasRenderer，合并Mesh，提交DrawCall |

---

## 6.2 Canvas的三大职责

Canvas承担三个职责，缺一不可。

### 6.2.1 职责一：组织UI层级结构

Canvas下的所有UI元素构成一个**渲染层级树**。Canvas通过以下维度管理谁在上、谁在下：

- **Sorting Layer**（排序层）
- **Order in Layer**（层内排序）
- **Hierarchy顺序**（层级面板中的先后）
- **Camera Depth**（Camera模式下）

这些规则将在6.4节详细展开。

### 6.2.2 职责二：控制渲染模式

Canvas的`renderMode`属性决定了UI以何种方式接入Unity渲染管线。这种选择直接影响：

- UI是否依赖Camera
- UI是否受后处理影响
- UI是否参与深度排序
- UI是否受透视投影影响

三种模式（Overlay / Camera / WorldSpace）的详细差异见6.3节。

### 6.2.3 职责三：执行批处理与渲染提交

这是Canvas最底层、最关键的职责。在每帧渲染循环中，Unity引擎会调用`Canvas.BuildBatch()`方法（native实现，C#侧不可见），执行以下操作：

1. 遍历当前Canvas下的所有CanvasRenderer
2. 从每个CanvasRenderer读取已存储的Mesh、Material、Texture
3. 将材质和纹理相同的Mesh合并为一个大的合并Mesh
4. 将合并结果提交到GPU Command Buffer

> **重要理解**：Canvas不参与顶点数据的生成（那是Graphic的职责），也不直接暂存数据（那是CanvasRenderer的职责）。Canvas的职责是**收集、合批、提交**——它是整个UI渲染流程的终点协调者。

---

## 6.3 三种Render Mode详解

`Canvas.renderMode`是一个枚举类型，定义在引擎侧（`UnityEngine.RenderMode`，不在 uGUI 仓库中）：

```csharp
// Canvas（引擎内置组件）
public enum RenderMode
{
    ScreenSpaceOverlay,  // 默认模式
    ScreenSpaceCamera,   // 绑定到特定Camera
    WorldSpace           // 3D空间中的UI
}
```

### 6.3.1 Screen Space - Overlay（覆盖模式）

**Overlay是默认且最常用的模式。**

核心行为：

- **不依赖任何Camera**：UI不需要经过Camera的MVP矩阵变换，顶点直接使用屏幕空间坐标
- **在所有Camera之后渲染**：Overlay UI在所有Camera（包括后处理）都完成渲染后才绘制
- **不受后处理影响**：因为绘制在后处理之后，Bloom、Color Grading等效果不会作用于Overlay UI
- **不参与深度测试**：ZWrite Off，层级完全由Canvas排序决定

```csharp
// Overlay模式的典型配置
// renderMode = RenderMode.ScreenSpaceOverlay
// worldCamera = null（没有关联Camera）
```

**适用场景**：主界面、HUD、弹窗——几乎所有不需要被场景遮挡的UI。

**Inspector面板中的Overlay配置**：

```
Canvas 组件
├── Render Mode:  Screen Space - Overlay
├── Pixel Perfect: [ ]  （是否像素对齐）
├── Sorting Layer: Default
└── Order in Layer: 0
```

### 6.3.2 Screen Space - Camera（摄像机模式）

UI绑定到特定Camera，参与该Camera的完整渲染流程。

核心行为：

- **依赖Camera**：UI经过Camera的MVP矩阵变换
- **受后处理影响**：如果Camera开启了后处理（Bloom、Color Grading等），UI也会受到效果
- **参与Camera Stack**：在多Camera场景中，UI随所属Camera一起渲染
- **planeDistance控制距离**：UI距离Camera的平面距离，影响UI在透明队列中的排序

```csharp
// Camera模式的典型配置
// renderMode = RenderMode.ScreenSpaceCamera
// worldCamera = 指定的Camera引用
// planeDistance = 100（默认，UI距离Camera的平面距离）
```

**Inspector面板中的Camera配置**：

```
Canvas 组件
├── Render Mode:  Screen Space - Camera
├── Render Camera: Main Camera  （指定绑定Camera）
├── Plane Distance: 100
├── Sorting Layer: Default
└── Order in Layer: 0
```

**适用场景**：
- 需要与场景后处理风格统一的UI（如CG过场）
- 希望UI只被特定Camera渲染（如VR中的左右眼Camera）
- UI需要被Camera Stack中的Overlay Camera单独控制

### 6.3.3 World Space（世界空间模式）

Canvas完全进入三维世界坐标体系，UI作为3D物体存在于场景中。

核心行为：

- **三维空间定位**：使用Transform.position控制UI位置，不再是屏幕坐标
- **参与深度测试**：可以被其他3D物体遮挡（ZTest开启）
- **受透视投影影响**：近大远小，有透视效果
- **参与透明排序**：与场景中其他透明物体一起按距离排序

```csharp
// WorldSpace模式的典型配置
// renderMode = RenderMode.WorldSpace
// worldCamera = null（使用场景中渲染此Canvas的Camera）
```

**Inspector面板中的WorldSpace配置**：

```
Canvas 组件
├── Render Mode:  World Space
├── Event Camera: Main Camera  （用于事件投射的Camera）
├── Sorting Layer: Default
└── Order in Layer: 0
```

**适用场景**：
- 3D世界中的交互UI（如NPC头上的名字、场景中的操作面板）
- VR/AR中的空间UI
- UI与3D物体融合的特效场景

### 6.3.4 三种模式行为对比

| 维度 | Overlay | Camera | World Space |
|------|---------|--------|-------------|
| 依赖Camera | 否 | 是 | 是 |
| 坐标空间 | 屏幕空间 | 屏幕空间 | 世界空间 |
| 受后处理影响 | 否 | 是 | 是 |
| 参与深度测试 | 否 | 否（默认ZWrite Off） | 是 |
| 受透视投影影响 | 否 | 否 | 是 |
| 被场景物体遮挡 | 不可能 | 不可能 | 可能 |
| 随Camera Stack | 否（在所有Camera外） | 是 | 是 |
| 常用场景 | 主界面/HUD/弹窗 | 风格化UI/CG | 3D空间交互 |

---

## 6.4 Canvas的排序规则

UI的渲染层级是一个多因素共同决定的复杂问题。Canvas作为层级的组织者，其排序规则按渲染模式有所不同。

### 6.4.1 基本排序要素

所有Canvas都具备以下排序属性：

```csharp
// Canvas（引擎内置）
public class Canvas : Behaviour
{
    public RenderMode renderMode;      // Overlay / Camera / WorldSpace
    public int sortingOrder;           // Order in Layer（Inspector中配置）
    public int sortingLayerID;         // Sorting Layer的ID
    public int cachedSortingLayerValue; // 引擎缓存的排序层值（只读）

    // Camera模式专属
    public Camera worldCamera;         // 绑定的Camera
    public float planeDistance;        // UI距离Camera的平面距离
}
```

| 排序要素 | 对应属性 | 影响范围 |
|---------|---------|---------|
| **Sorting Layer** | sortingLayerID | 跨Canvas的主要排序维度 |
| **Order in Layer** | sortingOrder | 在同一Sorting Layer内排序 |
| **Camera Depth** | worldCamera.depth | Camera模式下，按Camera深度排序 |
| **Hierarchy顺序** | Hierarchy中的先后位置 | 同级Canvas的最终回退规则 |
| **planeDistance** | planeDistance | Camera/WorldSpace模式下，按距离排序 |

### 6.4.2 Overlay模式的排序

Overlay Canvas不依赖Camera，排序规则相对简单：

```
优先级：Sorting Layer → Order in Layer → Hierarchy顺序
```

1. **Sorting Layer**：在Project Settings → Tags and Layers中配置。值越大越靠上。
2. **Order in Layer**（sortingOrder）：在同一Sorting Layer内，值越大越靠上。
3. **Hierarchy顺序**：如果以上两者都相同，则Hierarchy中**靠后的**元素渲染在**靠前的**之上。

```csharp
// 示例：两个Overlay Canvas的排序
// Canvas A: Sorting Layer = Default(0), Order in Layer = 10
// Canvas B: Sorting Layer = Default(0), Order in Layer = 0
// 结果：Canvas A 在 Canvas B 之上

// Canvas C: Sorting Layer = UI(5), Order in Layer = 0
// Canvas D: Sorting Layer = Default(0), Order in Layer = 100
// 结果：Canvas C 在 Canvas D 之上（Sorting Layer优先级更高）
```

### 6.4.3 Camera/WorldSpace模式的排序

当Canvas绑定到Camera或处于WorldSpace时，排序规则多了一层Camera的介入：

```
优先级：Camera Depth → Sorting Layer → Order in Layer → planeDistance
```

1. **Camera Depth**：绑定Camera的depth值。**Camera的深度优先级高于所有UI的排序属性**。后渲染的Camera内容会覆盖先渲染的。
2. **Sorting Layer**：同Overlay模式，值越大越靠上。
3. **Order in Layer**：同Overlay模式。
4. **planeDistance**（仅Camera模式）：距Camera平面越近（值越小），渲染越靠上。因为UI走透明队列，近距离先绘制，远距离后绘制——后绘制的覆盖先绘制的。

> **关键结论**：Camera的叠加顺序优先级高于Canvas Sorting Order。例如Camera A（depth=0）上有一个sortingOrder=100的UI，Camera B（depth=1）上有一个sortingOrder=0的UI——最终Camera B上的UI会覆盖Camera A上的UI，因为Camera B执行渲染的顺序更晚。

### 6.4.4 Hierarchy顺序的作用范围

Hierarchy顺序只有在**同级Canvas**且所有其他排序因素都相同时才起作用。在以下情况下Hierarchy顺序被忽略：

- 不同Sorting Layer的Canvas之间
- 不同sortingOrder的Canvas之间
- 不同Camera下的Canvas之间

---

## 6.5 Canvas.BuildBatch()与批处理

### 6.5.1 BuildBatch的工作机制

`Canvas.BuildBatch()`是整个UI渲染流程的**最终提交阶段**。它在C#侧的定义非常简单：

```csharp
// Canvas（引擎内置组件）
public class Canvas : Behaviour
{
    // 其他属性...

    /// <summary>
    /// 构建并提交渲染批次（Native方法，C#侧看不到实现）
    /// </summary>
    [NativeMethod]
    private extern void BuildBatch();
}
```

这个`[NativeMethod]`标注告诉编译器：该方法的实现在Unity引擎的C++层。C#侧只能调用它，无法查看或修改它的实现。

根据Unity官方文档和Frame Debugger的反推观察，BuildBatch的内部逻辑大致如下：

```
Canvas.BuildBatch()
│
├─ Step 1: 遍历当前Canvas下的所有活跃CanvasRenderer
│
├─ Step 2: 从每个CanvasRenderer读取：
│   ├─ Mesh（通过CanvasRenderer.GetMesh()）
│   ├─ Material（通过CanvasRenderer.GetMaterial()）
│   └─ Texture（通过CanvasRenderer.GetTexture()）
│
├─ Step 3: 按渲染状态分组：
│   ├─ 相同Material → 同组
│   ├─ 相同Texture → 同组
│   └─ 相同Shader + 相同RenderState → 同组
│
├─ Step 4: 合并同组Mesh
│   └─ 将多个小Mesh合并为一个大的合并Mesh
│
└─ Step 5: 提交DrawCall到GPU Command Buffer
    └─ 每组产生一个DrawCall
```

### 6.5.2 Canvas是Batch的边界

**不同Canvas之间的UI元素永远不会合并在同一个Batch中。**

这意味着：
- Canvas A中的一个Image和Canvas B中的一个Image，即使使用完全相同的材质和纹理，也会产生**两个DrawCall**
- 每个Canvas独立执行自己的`BuildBatch()`
- 单个Canvas越大（包含的UI元素越多），单次BuildBatch的开销越大

```
场景中有两个Canvas：

Canvas A (Sorting Order = 0)
├── Image (相同材质, 相同纹理)
└── Text  (相同材质, 相同纹理)
     ↓ BuildBatch → 合并为一个DrawCall ✅

Canvas B (Sorting Order = 10)
├── Image (与Canvas A完全相同的材质和纹理)
└── Text  (与Canvas A完全相同的材质和纹理)
     ↓ BuildBatch → 合并为一个DrawCall ✅
     ↓ 但与Canvas A的DrawCall不能合并 ❌
```

### 6.5.3 为什么不同Canvas不能合批

这是Unity引擎的设计决策，原因有三：

1. **排序独立性**：每个Canvas有自己的Sorting Layer和Order in Layer。如果跨Canvas合批，排序逻辑会变得极其复杂（同一个Batch中的元素必须按相同顺序渲染）。

2. **更新隔离性**：一个Canvas下的UI变化不会触发另一个Canvas的BuildBatch。如果跨Canvas合批，一个UI元素的变化就会影响多个Canvas。

3. **渲染模式差异性**：不同Canvas可能使用不同的renderMode（一个Overlay，一个Camera），其顶点所处的坐标空间不同，无法直接合并。

### 6.5.4 BuildBatch的触发时机

Canvas在以下情况下会触发BuildBatch：

- Canvas下的UI元素被标记为dirty（顶点变化、材质变化等）
- Canvas自身属性变化（sortingOrder、renderMode等）
- Canvas启用/禁用
- 新UI元素被添加到Canvas下
- 每帧都会至少执行一次（即使没有变化，也会检查）

> 以上为引擎内部行为，C# 侧无源码可查，依据 Unity 官方文档与 Frame Debugger 观察推断（BuildBatch 本身是 native 方法）。

> **性能启示**：频繁触发BuildBatch是UI性能问题的常见来源。将**变化频繁**的UI（如实时更新的HUD）放在独立的Canvas中，可以避免它影响整个UI系统的Batch重建。

---

## 6.6 Canvas关键源码解析

### 6.6.1 Canvas核心属性

以下是Canvas中C#侧可见的核心成员：

```csharp
// Canvas（引擎内置组件，简化版）
namespace UnityEngine
{
    [RequireComponent(typeof(RectTransform))]
    public class Canvas : Behaviour
    {
        // ===== 渲染模式 =====
        public RenderMode renderMode { get; set; }
        // ScreenSpaceOverlay(0) / ScreenSpaceCamera(1) / WorldSpace(2)

        // ===== Camera模式关联 =====
        public Camera worldCamera { get; set; }
        // Overlay模式下为null；Camera模式下为绑定的Camera

        // ===== 排序 =====
        public int sortingOrder { get; set; }
        // Order in Layer，同一Sorting Layer内的排序值

        public int sortingLayerID { get; set; }
        // Sorting Layer的ID，对应Project Settings中的配置

        public int cachedSortingLayerValue { get; }
        // 引擎缓存的Sorting Layer数值（只读）

        // ===== 缩放 =====
        public float scaleFactor { get; set; }
        // UI缩放因子，CanvasScaler修改的就是这个值

        // ===== 像素对齐 =====
        public bool pixelPerfect { get; set; }
        // 是否启用像素对齐，开启后UI边缘不会出现模糊

        // ===== 世界空间参数 =====
        public float planeDistance { get; set; }
        // Camera模式下UI距离Camera的平面距离
        // WorldSpace模式下此值由RectTransform的Z位置决定

        // ===== Native方法 =====
        [NativeMethod] private extern void BuildBatch();
        [NativeMethod] private extern void SendWillRenderCanvases();

        // ===== 静态事件 =====
        // 委托是无参的：public delegate void WillRenderCanvases();
        public static event WillRenderCanvases willRenderCanvases;
        public static event WillRenderCanvases preWillRenderCanvases;
        // 渲染前的回调事件，CanvasUpdateRegistry / CanvasScaler 依赖它驱动
        // 用法见第 4 章 4.2.1（+= PerformUpdate）与第 21 章 21.4.1
    }
}
```

属性使用示例：

```csharp
// 运行时修改Canvas属性
Canvas canvas = GetComponent<Canvas>();

// 切换到Camera模式
canvas.renderMode = RenderMode.ScreenSpaceCamera;
canvas.worldCamera = mainCamera;
canvas.planeDistance = 50;

// 提高排序层级
canvas.sortingOrder = 100;

// 启用像素对齐
canvas.pixelPerfect = true;
```

### 6.6.2 CanvasScaler与scaleFactor

`CanvasScaler`是挂载在Canvas上的常见组件，它的核心职责就是**修改Canvas.scaleFactor**来实现UI适配。

```csharp
// CanvasScaler.cs（UGUI源码，核心逻辑简化）
public class CanvasScaler : UIBehaviour
{
    private Canvas m_Canvas;

    protected override void OnEnable()
    {
        m_Canvas = GetComponent<Canvas>();
        // 在Canvas渲染前执行缩放计算
        Canvas.preWillRenderCanvases += Handle;
    }

    protected virtual void Handle()
    {
        if (m_Canvas == null || !IsActive())
            return;

        switch (m_ScaleMode)
        {
            case ScaleMode.ConstantPixelSize:
                HandleConstantPixelSize();   // 固定scaleFactor
                break;
            case ScaleMode.ScaleWithScreenSize:
                HandleScaleWithScreenSize(); // 根据分辨率动态计算
                break;
            case ScaleMode.ConstantPhysicalSize:
                HandleConstantPhysicalSize(); // 根据DPI计算
                break;
        }
    }

    // 三种模式最终都调用此方法设置Canvas的scaleFactor
    protected void SetScaleFactor(float scaleFactor)
    {
        m_Canvas.scaleFactor = scaleFactor;  // ← 核心赋值
    }
}
```

**关键流程**：

```
CanvasScaler.OnEnable()
  → 注册到 Canvas.preWillRenderCanvases 事件
  → 在每帧Canvas开始渲染之前执行

Canvas.preWillRenderCanvases 触发
  → CanvasScaler.Handle()
  → 根据ScaleMode计算scaleFactor
  → 设置 Canvas.scaleFactor = 计算值
  → Canvas在后续BuildBatch中使用此scaleFactor
```

`scaleFactor`的影响范围：

```csharp
// Canvas.scaleFactor 直接影响所有子UI的缩放
// 设置前
canvas.scaleFactor = 1.0f;  // 原始大小

// 设置后
canvas.scaleFactor = 2.0f;  // UI整体放大2倍
```

> **注意**：直接修改`Canvas.scaleFactor`和通过`CanvasScaler`修改效果相同——CanvasScaler只是封装了计算逻辑，最终赋值给同一个属性。如果你手动设置了`canvas.scaleFactor`，CanvasScaler的自动计算会被覆盖。

### 6.6.3 Inspector面板中的Canvas配置

Canvas组件在Inspector中的完整配置项：

```
Canvas 组件
├── Render Mode:          Screen Space - Overlay (默认)
│                        ├─ Screen Space - Overlay
│                        ├─ Screen Space - Camera
│                        └─ World Space
│
├── [Overlay模式时]
│   ├── Pixel Perfect:   [ ]  像素对齐
│   ├── Sort Order:      0    sortingOrder的值
│   ├── Target Display:  Display 1
│   └── Additional Settings:
│       ├── Event Camera: (none)
│       ├── Sorting Layer: Default
│       └── Order in Layer: 0
│
├── [Camera模式时]
│   ├── Render Camera:   (Camera引用)
│   ├── Plane Distance:  100
│   ├── Pixel Perfect:   [ ]
│   └── Additional Settings: (同上)
│
└── [WorldSpace模式时]
    ├── Event Camera:    (Camera引用，用于事件)
    └── Additional Settings: (同上)
```

---

## 6.7 多Canvas架构

### 6.7.1 为什么要拆分多Canvas

在大型UI系统中，单个Canvas往往不够用。常见的多Canvas拆分策略：

| 拆分原因 | 说明 |
|---------|------|
| **性能隔离** | 变化频繁的UI（HUD、滚动列表）放在独立Canvas，避免触发主Canvas的BuildBatch |
| **渲染模式不同** | Overlay UI + WorldSpace UI需要不同Canvas |
| **排序需求** | 不同层级的UI需要不同的Sorting Layer或Order范围 |
| **Camera分离** | 某些UI需要绑定不同Camera（如UI Camera + 场景Camera） |

### 6.7.2 典型拆分策略

**双层结构**（最常见）：

```
Canvas A (Overlay, Sorting Order = 0)
  ├── 背景层
  └── 内容层

Canvas B (Overlay, Sorting Order = 10)
  ├── 弹窗层
  └── 提示层

Canvas C (Overlay, Sorting Order = 100)
  └── Loading动画（变化频繁，独立Canvas避免触发主Canvas重建）
```

**分离相机结构**：

```
Canvas A (Camera模式, 主Camera)
  └── 场景UI（受场景后处理影响）

Canvas B (Overlay, Sorting Order = 0)
  └── HUD和操作界面（不受后处理影响）
```

### 6.7.3 多Canvas的性能收益

```csharp
// 反面示例：所有UI放在一个超大Canvas中
// 任何一个UI元素变化 → 整个Canvas的BuildBatch重新执行
// 这是UI性能问题的常见来源

// 正面示例：按变化频率拆分Canvas
// HUD（变化频繁） → 独立小Canvas
// 主界面（变化少） → 另一个Canvas
// 弹窗（偶尔出现） → 第三个Canvas
// 一个Canvas的UI变化不会影响其他Canvas的Batch
```

---

## 6.8 Canvas与其它核心组件的关系

理解Canvas在UGUI系统中的位置，需要理清它与另外两个关键组件的关系。

### 6.8.1 Canvas vs Graphic

| 维度 | Canvas | Graphic |
|------|--------|---------|
| 职责 | 合批、提交渲染 | 生成网格、管理材质 |
| 核心方法 | BuildBatch()（native） | OnPopulateMesh()（虚方法） |
| 数据方向 | 从CanvasRenderer读取 | 写入CanvasRenderer |
| 更新驱动 | 每帧自动执行 | 通过Dirty标记触发重建 |

**谁先执行**：Graphic先执行（OnPopulateMesh生成数据 → SetMesh存入CanvasRenderer），然后Canvas再执行BuildBatch从CanvasRenderer读取数据并提交。

### 6.8.2 Canvas vs CanvasRenderer

| 维度 | Canvas | CanvasRenderer |
|------|--------|---------------|
| 数量 | 通常一个Canvas下有多个CanvasRenderer | 每个UI元素一个 |
| 职责 | 统一管理所有CanvasRenderer的数据 | 暂存单个UI元素的Mesh和材质 |
| 与Native层的关系 | BuildBatch直接调用Native渲染接口 | SetMesh/SetMaterial桥接到Native层 |
| 生命周期 | 场景级存在 | 随UI元素创建和销毁 |

---

## 6.9 AdditionalShaderChannels：顶点数据通道控制

### 6.9.1 是什么

`Canvas.additionalShaderChannels` 控制哪些顶点附加数据从 CPU 传递到 GPU Shader。默认只传 `position`、`color`、`uv0`，其余通道静默丢弃。

```csharp
[Flags]
public enum AdditionalCanvasShaderChannels
{
    None      = 0,
    TexCoord1 = 1,   // uv1（TEXCOORD1）
    TexCoord2 = 2,   // uv2（TEXCOORD2）
    TexCoord3 = 4,   // uv3（TEXCOORD3）
    Normal    = 8,   // normal（NORMAL）
    Tangent   = 16,  // tangent（TANGENT）
}
```

### 6.9.2 功能对照表

| 功能 | 需要 | 原因 |
|------|------|------|
| TextMeshPro | TexCoord2 | SDF 参数通过 uv2 传递 |
| PositionAsUV1 | TexCoord1 | 位置复制到 uv1 |
| Shadow/Outline | 无（CPU 侧） | 顶点复制，不依赖额外 UV |
| 自定义 Shader 读 uv1/uv2/uv3 | 对应通道 | 未启用时数据被丢弃，不报错 |
| 自定义 Shader 读 normal/tangent | Normal/Tangent | 法线/光照 |

未开启通道时数据被静默丢弃——TMP 渲染为纯色块是最常见的表现。

### 6.9.3 带宽成本

开启额外通道会按比例增加送往 GPU 的顶点数据量：

| 配置 | 相对基线的顶点数据量 |
|------|---------------|
| None（position + color + uv0） | 基线 |
| +TexCoord1 | 约 +1/3 |
| +TexCoord2 | 约 +2/3 |
| +Normal / +Tangent | 再各增一档（Tangent 是 4 个 float，比 Normal 更贵） |

> 这里只给相对量级：单顶点的实际字节数取决于各通道的分量数与引擎的顶点布局对齐方式，不同 Unity 版本可能不同，需要精确数值时以 Frame Debugger 中该批次的 Vertex Buffer 大小为准。

按需开启，避免不必要的带宽浪费。

### 6.9.4 最佳实践

- TMP 项目 → `TexCoord2`
- 无 TMP → `None`
- 不确定 → 仅 `TexCoord2`，非 TMP UI 不受影响
- 除确定需要外不开启 Normal/Tangent/TexCoord3

---

### 推荐的源码阅读路径

```
Canvas 是引擎内置类型（UIModule），看官方文档；
uGUI 侧：Layout/CanvasScaler.cs（分辨率适配）、UI/Core/GraphicRaycaster.cs（Canvas 相关射线检测）、
CanvasUpdateRegistry.cs（订阅 Canvas.willRenderCanvases）。
```

---

## 本章总结

### Canvas的核心地位

Canvas不是"UI组件的容器"这么简单——它是UGUI中**唯一具备渲染提交能力的组件**。没有Canvas，所有UI组件生成的网格数据都只是CPU内存中的闲置数据，永远不会被GPU看见。

### 三种渲染模式的选择建议

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 主界面、弹窗、HUD | **Overlay** | 不受后处理影响，排序简单，性能最好 |
| 需要与场景后处理统一 | **Camera** | UI参与Camera的渲染链路 |
| 3D空间交互（VR/AR） | **World Space** | UI存在于三维空间中 |

### 排序规则速记

- **Overlay**：Sorting Layer → Order in Layer → Hierarchy顺序
- **Camera/WorldSpace**：Camera Depth → Sorting Layer → Order in Layer → planeDistance

### 批处理理解要点

- **Canvas是Batch的边界**——不同Canvas不能合批
- **BuildBatch是native方法**——C#侧看不到实现
- **BuildBatch在Graphic.Rebuild之后执行**——先有数据，再合批提交
- **Canvas拆分是UI性能优化的核心手段**——减小单次BuildBatch的范围

### 关键源码文件

| 文件 | 关键内容 |
|------|---------|
| `Canvas`（引擎内置） | renderMode / sortingOrder / worldCamera / scaleFactor / BuildBatch() |
| `CanvasScaler.cs` | 三种ScaleMode的计算逻辑，最终修改Canvas.scaleFactor |

---

## 勘误汇总（对照官方文档）

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🟡 | 6.3 | `RenderMode` 枚举"定义在 UGUI 源码中" | 引擎侧 `UnityEngine.RenderMode`（`UnityEngine.UIModule`），不在 uGUI 仓库 |
| 2 | 🟡 中等 | 6.6.1 | `public static event Action<float> willRenderCanvases;` | 委托无参（`public delegate void WillRenderCanvases();`）。第 4 章 4.2.1（`+= PerformUpdate`）与第 21 章 21.4.1（`+= OnWillRenderCanvases`）的无参用法才是正确的 |
| 3 | 🟢 轻微 | 6.4.1 | `public int cached SortingLayerValue;` | 标识符中间多了空格，非法 C#；正确为 `cachedSortingLayerValue`（6.6.1 处写法无误） |
| 4 | 🟢 轻微 | 6.9.3 | 带宽表给出 None=32B / +TexCoord1=48B / +TexCoord2=64B 等绝对字节数 | 未说明计算口径（分量数、对齐方式），且与 position(12)+color(4)+uv0(8)=24B 对不上。已改为相对量级，精确值以 Frame Debugger 的 Vertex Buffer 为准 |
