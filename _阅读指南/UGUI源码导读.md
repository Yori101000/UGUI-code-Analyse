# UGUI 源码导读

> 本文档不是章节总结，而是**源码阅读指南**——告诉你每个知识点对应 UGUI 源码中的哪个文件、哪个类、哪个方法，以及怎么看懂它们。建议配合 UGUI 源码（Unity-Technologies/uGUI，**以 main 分支为准**）一起阅读。

> ⚠️ **结构说明（2026-08）**：本书为四部分 25 章结构（见 `README.md` 目录表）。本文档按新编号 01~25 编写；TMP 深度分析在第 16 章、URP/HDRP 在第 10 章、ScrollRect 在第 11 章、Profiler 实战在第 24 章。

> ⚠️ **引擎内置类型**：`Canvas`、`CanvasRenderer`、`RectTransformUtility`、`UIVertex`、`TextGenerator`、`RenderMode` 等属 `UnityEngine.UIModule`（`RectTransform` 属 `UnityEngine.CoreModule`），**不在 uGUI 仓库中**。它们以官方文档 + 运行时反射为准。`TextMeshPro` 是独立包 `com.unity.textmeshpro`（仓库 Unity-Technologies/TextMeshPro），也不在 uGUI 仓库中。

---

## 如何获取 UGUI 源码

```
方式一：GitHub（推荐）
https://github.com/Unity-Technologies/uGUI → 分支选 main

方式二：Unity 包管理器
Window → Package Manager → Unity UI → 在文件管理器中显示（本地包缓存）
```

main 分支核心目录结构：

```
com.unity.ugui/
└── Runtime/UGUI/
    ├── UI/Core/                  ← UI 组件核心
    │   ├── Graphic.cs / MaskableGraphic.cs / Image.cs / RawImage.cs / Text.cs
    │   ├── Selectable.cs / Button.cs / Toggle.cs / Slider.cs / Scrollbar.cs / Dropdown.cs / InputField.cs
    │   ├── Mask.cs / RectMask2D.cs / StencilMaterial.cs / MaskUtilities.cs
    │   ├── CanvasUpdateRegistry.cs / GraphicRegistry.cs / GraphicRaycaster.cs
    │   ├── ScrollRect.cs / FontUpdateTracker.cs
    │   ├── Utility/              ← VertexHelper.cs（顶点构建工具）
    │   ├── Layout/               ← LayoutGroup.cs / LayoutRebuilder.cs / CanvasScaler.cs 等
    │   ├── Culling/              ← ClipperRegistry.cs / IClippable.cs / IClipper.cs
    │   └── VertexModifiers/      ← IMeshModifier.cs / BaseMeshEffect.cs / Shadow.cs / Outline.cs / PositionAsUV1.cs
    └── EventSystem/              ← 事件系统，与 UI/Core 同级
        ├── EventSystem.cs / ExecuteEvents.cs / UIBehaviour.cs
        ├── EventData/PointerEventData.cs
        ├── InputModules/         ← BaseInputModule.cs / StandaloneInputModule.cs
        └── Raycasters/           ← BaseRaycaster.cs / PhysicsRaycaster.cs / Physics2DRaycaster.cs
```

注意：`UI-Default.shader` 等 UI Shader 是引擎内置资源（`Data/Resources/BuiltinShaders/UI-Default.shader`），不在 C# 仓库中。

---

# 第一部分 基础理论（01~09）

## 第01章 UGUI 整体架构

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Graphic.cs` | 可视化 UI 的抽象基类：`SetVerticesDirty` / `Rebuild` / `OnPopulateMesh` / `UpdateGeometry` |
| `UI/Core/CanvasUpdateRegistry.cs` | 重建调度入口（`PerformUpdate`） |
| `Canvas` / `CanvasRenderer` / `RectTransform`（引擎内置） | 渲染入口、数据容器、空间定义 |

**核心概念 → 源码映射**

- "保留模式"：对比 `OnGUI()`（IMGUI，每帧执行）与 UGUI 组件在场景中持久存在、Dirty 后延迟重建。
- "一帧数据流"：`Graphic.OnPopulateMesh` → `CanvasRenderer.SetMesh` → 引擎 `Canvas.BuildBatch` → GPU。

**阅读路径**：先看 `Graphic.cs` 的 `SetVerticesDirty()` 与 `Rebuild()`，理解"标记 → 调度 → 重建"的骨架；细节交给第 04/05 章。

## 第02章 UI 的本质：从图形到网格

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Utility/VertexHelper.cs` | 顶点构建工具：`NativeArray<UIVertex>` + `NativeArray<ushort>`、`Clear()`、`AddVert/AddTriangle`、`FillMesh`（9 通道单流上传） |
| `UIVertex`（引擎内置，UIModule） | 顶点结构体：position / normal / tangent / color / uv0~uv3 / prevPosition |
| `UI/Core/Image.cs`、`UI/Core/Text.cs` | `OnPopulateMesh` 的两种典型实现 |

**核心概念 → 源码映射**

- "UI 在 GPU 眼中是 Mesh"：`Graphic.OnPopulateMesh(VertexHelper)` 就是把矩形/字符转成顶点流。
- "RectTransform 是空间起点"：`Graphic.GetPixelAdjustedRect()` 读取 `rectTransform.rect` 生成矩形顶点。

**阅读路径**：`VertexHelper.cs`（内部存储 → `Clear` → `AddVert/AddTriangle` → `FillMesh`）→ `Image.cs OnPopulateMesh` → `Text.cs OnPopulateMesh`。

## 第03章 RectTransform 核心机制

**源码文件**：`RectTransform`（引擎内置，CoreModule）、`RectTransformUtility`（引擎内置，UIModule）。**uGUI 仓库内无 C# 实现**，uGUI 只是调用方（如 `GraphicRaycaster`、`Scrollbar.cs` 调用 `RectTransformUtility` 的坐标转换）。

**核心概念 → API 映射**

- 锚点/轴心/offsetMin/offsetMax/sizeDelta/anchoredPosition：`RectTransform` 属性（官方文档 + 反射验证）。
- 坐标转换：`RectTransformUtility.ScreenPointToLocalPointInRectangle` / `ScreenPointToWorldPointInRectangle` / `WorldToScreenPoint` / `ScreenPointToRay` / `PixelAdjustPoint` / `PixelAdjustRect`。

**阅读路径**：官方文档 `RectTransform` → `RectTransformUtility`；再到 uGUI 仓库搜索 `RectTransformUtility.` 看真实调用场景。

## 第04章 UI 更新与重建系统

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/CanvasUpdateRegistry.cs` | 单例 + `Canvas.willRenderCanvases`；`IndexedSet` 双队列；`PerformUpdate()` 分阶段执行 |
| `UI/Core/Layout/LayoutRebuilder.cs` | 布局重建执行器（`ICanvasElement`） |
| `UI/Core/Culling/ClipperRegistry.cs` | 裁剪更新（`ClipperRegistry.instance.Cull()`） |

**核心概念 → 源码映射**

- 阶段划分：Layout Rebuild 遍历 `Prelayout→Layout→PostLayout`；`ClipperRegistry.Cull()` 居中；Graphic Rebuild 遍历 `PreRender→LatePreRender`。
- Dirty 合并：`Graphic.Rebuild(CanvasUpdate.PreRender)` 内检查 `m_VertsDirty` / `m_MaterialDirty`。

**阅读路径**：`CanvasUpdateRegistry.cs` 的 `PerformUpdate()` → `Graphic.cs` 的 `Rebuild()` → `LayoutRebuilder.cs`。

## 第05章 Graphic 系统

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Graphic.cs` | `SetVerticesDirty/SetMaterialDirty/SetLayoutDirty`、`Rebuild`、`DoMeshGeneration`、`UpdateGeometry/UpdateMaterial`、`materialForRendering`、`GetModifiedMaterial` |
| `UI/Core/MaskableGraphic.cs` | Stencil 裁剪支持（`Cull`、`GetModifiedMaterial`） |
| `UI/Core/Image.cs` / `RawImage.cs` / `Text.cs` | 三种子类实现 |
| `UI/Core/VertexModifiers/IMeshModifier.cs` | `ModifyMesh(VertexHelper)` 顶点后处理接口 |

**核心概念 → 源码映射**

- 三种 Dirty：顶点/材质/布局分别触发 `UpdateGeometry` / `UpdateMaterial` / 布局重算。
- `DoMeshGeneration()`：`OnPopulateMesh(s_VertexHelper)` → `IMeshModifier` 链 → `FillMesh(workerMesh)` → `canvasRenderer.SetMesh(workerMesh)`。

**阅读路径**：`Graphic.cs`（Dirty → Rebuild → DoMeshGeneration）→ `MaskableGraphic.cs` → `Image.cs OnPopulateMesh`。

## 第06章 Canvas 系统

**源码文件**：`Canvas`（引擎内置，UIModule）——C# 侧只有 API 壳，`BuildBatch`/`SendWillRenderCanvases` 为 native。uGUI 侧：`UI/Core/Layout/CanvasScaler.cs`、`UI/Core/GraphicRaycaster.cs`。

**核心概念 → 源码映射**

- `RenderMode`（Overlay/Camera/WorldSpace）：引擎 `UnityEngine.RenderMode` 枚举。
- `Canvas.willRenderCanvases`：`CanvasUpdateRegistry` 构造函数中订阅，是每帧重建的触发器。

**阅读路径**：官方文档 `Canvas` → `CanvasUpdateRegistry.cs` 的构造函数 → `CanvasScaler.cs`。

## 第07章 CanvasScaler 分辨率适配

**源码文件**：`UI/Core/Layout/CanvasScaler.cs`。

**核心概念 → 源码映射**

- 三种 `ScaleMode`：`ConstantPixelSize` / `ScaleWithScreenSize` / `ConstantPhysicalSize`；`HandleScaleWithScreenSize()` 按参考分辨率与 `matchWidthOrHeight` 计算 `scaleFactor`。
- `HandleWorldCanvas()`：World Space 模式直接返回，不缩放。

**阅读路径**：`CanvasScaler.cs` → `HandleScaleWithScreenSize()` → 修改 `canvas.scaleFactor` 的路径。

## 第08章 CanvasRenderer 机制

**源码文件**：`CanvasRenderer`（引擎内置，UIModule）——`SetMesh/SetMaterial/SetTexture/SetColor/EnableRectClipping` 都是引擎 API。uGUI 侧调用点：`Graphic.cs`（`SetMesh(workerMesh)`、`UpdateMaterial()`）、`UI/Core/RectMask2D.cs`（`EnableRectClipping`）。

**核心概念 → 源码映射**

- "数据容器"：`Graphic` 生成数据 → `canvasRenderer.SetMesh/SetMaterial` 存入 → `Canvas.BuildBatch` 读取。
- 裁剪：`RectMask2D` 通过 `canvasRenderer.EnableRectClipping` 设置矩形裁剪，与 `Mask` 的 Stencil 方案不同。

**阅读路径**：`Graphic.cs` 搜索 `canvasRenderer.` → `RectMask2D.cs` 搜索 `EnableRectClipping`。

## 第09章 UI 批处理与 DrawCall

**源码文件**：合批规则在引擎 Native 层（`Canvas.BuildBatch`），C# 侧无源码。UI 侧相关：`Graphic.materialForRendering`（`GetModifiedMaterial` 链）、`UI/Core/StencilMaterial.cs`、`UI/Core/Mask.cs`、`UI/Core/RectMask2D.cs`。

**核心概念 → 源码映射**

- 断批原因：材质实例不同 / 纹理不同 / Stencil 状态不同 / RectMask2D 裁剪区域不同 / 跨 Canvas。
- Stencil 注入：`Mask` → `StencilMaterial.Add()` 生成带 stencil 状态的材质副本 → 打断批次。

**验证方式**：Frame Debugger（Window → Analysis）逐 DrawCall 查看断批原因。

---

# 第二部分 渲染链路（10~19）

## 第10章 UI 与渲染管线

**源码文件**：uGUI 侧无新源码；关键资源是引擎内置 `UI-Default.shader`（`Data/Resources/BuiltinShaders/`）。URP/HDRP 属于 **Unity-Technologies/Graphics** 仓库（`com.unity.render-pipelines.universal` / `.high-definition` 包），不在 uGUI 中。

**核心概念 → 源码映射**

- `Canvas.renderMode` 决定 UI 走哪条渲染路径；SRP 下 UI 仍由 Canvas 系统批处理提交。
- UI Shader 关键点：`Cull Off`、`Blend One OneMinusSrcAlpha`、`ZTest [unity_GUIZTestMode]`、`Stencil` 块。

**阅读路径**：BuiltinShaders 的 `UI-Default.shader` → Graphics 仓库中 URP 的 Canvas 支持代码。

## 第11章 Layout 布局系统

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Layout/LayoutGroup.cs` | 布局容器基类：`SetLayoutHorizontal/Vertical`、`SetChildAlongAxis` |
| `UI/Core/Layout/LayoutRebuilder.cs` | 布局重建执行器（`MarkLayoutForRebuild`、`Rebuild`） |
| `UI/Core/Layout/ContentSizeFitter.cs` | 按子元素首选尺寸驱动自身 |
| `UI/Core/Layout/GridLayoutGroup.cs`、`HorizontalOrVerticalLayoutGroup.cs` | 两种常用排列 |
| `UI/Core/ScrollRect.cs` | 滚动容器（`LateUpdate` 更新位置、`MovementType` 等） |

**核心概念 → 源码映射**：`ILayoutElement`（min/preferred/flexible）→ `LayoutRebuilder` → `SetChildAlongAxis` 写回 RectTransform。

**阅读路径**：`LayoutRebuilder.cs` 的 `Rebuild` → `LayoutGroup.cs` 的 `SetLayoutHorizontal/Vertical` → `ScrollRect.cs`。

## 第12章 EventSystem 事件系统

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `EventSystem/EventSystem.cs` | 事件循环入口（`Update`、`RaycastAll`、`current`） |
| `EventSystem/ExecuteEvents.cs` | 事件分发委托表（`GetEventHandler`、`Execute`） |
| `EventSystem/EventData/PointerEventData.cs` | 指针事件数据 |
| `EventSystem/InputModules/BaseInputModule.cs`、`StandaloneInputModule.cs` | 输入轮询（`Process()`） |
| `UI/Core/GraphicRaycaster.cs`、`EventSystem/Raycasters/BaseRaycaster.cs` 等 | 命中检测 |

**核心概念 → 源码映射**：`EventSystem.Update → BaseInputModule.Process → GraphicRaycaster.Raycast → ExecuteEvents.Execute`。

**阅读路径**：`EventSystem.cs` → `StandaloneInputModule.cs` 的 `Process()` → `GraphicRaycaster.cs` → `ExecuteEvents.cs`。

## 第13章 Selectable 与交互组件体系

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Selectable.cs` | 交互基类：`SelectionState` 状态机、`DoStateTransition`、`Navigation` |
| `UI/Core/Button.cs` / `Toggle.cs` / `ToggleGroup.cs` | 点击 / 开关组件 |
| `UI/Core/Slider.cs` / `Scrollbar.cs` / `Dropdown.cs` / `InputField.cs` | 数值与文本输入组件 |
| `UI/Core/Navigation.cs` / `ColorBlock.cs` / `SpriteState.cs` | 过渡与导航数据 |

**核心概念 → 源码映射**：`Selectable` 监听 EventSystem 的指针/导航事件 → 更新 `SelectionState` → 按 Transition（Color/Sprite/Animation）驱动视觉。

**阅读路径**：`Selectable.cs` 的 `OnPointerDown/Up/Enter/Exit` → `DoStateTransition` → 以 `Button.cs` 对照。

## 第14章 UI 资源与图集系统

**源码文件**：`Sprite`、`Texture`、`SpriteAtlas` 均为引擎类型（`UnityEngine`），**不在 uGUI 仓库**；动态图集（Dynamic Atlas）是引擎内部实现。uGUI 侧只有消费方：`Image.cs`（`sprite` 属性、`activeSprite`）、`RawImage.cs`。

**核心概念 → 源码映射**

- `Image.sprite` → `Sprite.texture` 决定合批纹理；Tight Mesh（`Sprite` 的 `meshType`）影响顶点数。
- 图集打包后多个 Sprite 共享同一张 Texture，是合批的基础（第 9 章）。

**阅读路径**：官方文档 `SpriteAtlas` → `Image.cs` 的 sprite 处理 → 用 Frame Debugger 验证合批。

## 第15章 Mask 与裁剪机制

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Mask.cs` | Stencil 裁剪：`GetModifiedMaterial` 注入 stencil 状态 |
| `UI/Core/RectMask2D.cs` | 矩形裁剪：`EnableRectClipping`（CanvasRenderer 侧） |
| `UI/Core/StencilMaterial.cs` | 按 stencil 参数缓存/创建材质副本 |
| `UI/Core/MaskUtilities.cs` | Stencil 深度分配（最多 8 层） |
| `UI/Core/Culling/ClipperRegistry.cs`、`IClippable.cs` / `IClipper.cs` | 裁剪矩形收集与 `Cull()` |

**核心概念 → 源码映射**：`Mask`（Stencil，断批）vs `RectMask2D`（矩形裁剪，不断批但影响批处理边界）。

**阅读路径**：`Mask.cs` → `StencilMaterial.cs` → `RectMask2D.cs` → `ClipperRegistry.cs`。

## 第16章 文本渲染系统

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/Text.cs` | 传统 Text：`OnPopulateMesh` 调用 `cachedTextGenerator.PopulateWithErrors(...)`，按 Quad 写回 `VertexHelper` |
| `UI/Core/FontUpdateTracker.cs` | 字体纹理重建跟踪（`TrackText/UntrackText`） |
| `Font` / `TextGenerator`（引擎内置） | 字符栅格化与排版引擎（FreeType/native） |
| TMP（独立包） | `com.unity.textmeshpro/Scripts/Runtime/`：`TMP_Text.cs`、`TextMeshProUGUI.cs`、`TMP_FontAsset.cs`、`TMP_SpriteAsset.cs`、`TMP_TextInfo.cs` 等 |

**核心概念 → 源码映射**

- 动态字体：`Font.RequestCharactersInTexture` → 引擎栅格化 → `Font.textureRebuilt` 事件 → `FontUpdateTracker` 通知 Text 重建。
- TMP：SDF 图集 + 字符缓存，`uv2` 传 SDF 参数（需开启 Canvas 的 `additionalShaderChannels` 的 TexCoord1）。

**阅读路径**：`Text.cs OnPopulateMesh` → `FontUpdateTracker.cs` → TMP 包 `TMP_Text.cs`。

## 第17章 UI Mesh 扩展机制

**源码文件**

| 文件 | 关键内容 |
|------|---------|
| `UI/Core/VertexModifiers/IMeshModifier.cs` | `ModifyMesh(VertexHelper)` 接口 |
| `UI/Core/VertexModifiers/BaseMeshEffect.cs` | 基类：自动在 `OnEnable/OnDisable/OnDidApplyAnimationProperties` 中 `SetVerticesDirty` |
| `UI/Core/VertexModifiers/Shadow.cs` / `Outline.cs` | 顶点复制：`ListPool<UIVertex>.Get()` → `GetUIVertexStream` → 修改 → `AddUIVertexTriangleStream` → `Release` |
| `UI/Core/Utility/VertexHelper.cs` | `GetUIVertexStream` / `AddUIVertexTriangleStream` |

**阅读路径**：`IMeshModifier.cs` → `BaseMeshEffect.cs` → `Shadow.cs`（最简实现）→ `Outline.cs`（×5 顶点）。

## 第18章 UI Shader 机制

**源码文件**：`UI-Default.shader`（引擎内置：`Data/Resources/BuiltinShaders/`，不在 C# 仓库）；uGUI 侧 stencil 来源：`Mask.cs` + `StencilMaterial.cs`。

**核心概念 → 源码映射**

- `UI/Default` 关键块：`Blend One OneMinusSrcAlpha`、`Cull Off`、`ZTest [unity_GUIZTestMode]`、`Stencil` 块由 `unity_UI*` 属性驱动。
- `CanvasRenderer` 传入的顶点通道：Position/Color/TexCoord0（+ additionalShaderChannels 的额外通道）。

**阅读路径**：打开安装目录 `UI-Default.shader` 逐段对照第 18 章表格。

## 第19章 UI 特效实现

**源码文件**：特效以 Shader 为主，无新增 uGUI C# 源码；可参考 `UI/Core/VertexModifiers/Shadow.cs`、`Outline.cs`（CPU 顶点特效），以及引擎内置 `UI-Default.shader`（Blend/Stencil 基础）。GrabPass、模糊、溶解等为 Shader 语言/渲染管线特性。

**核心概念 → 源码映射**：CPU 顶点特效（`BaseMeshEffect` 链）vs GPU 像素特效（自定义 Shader 替换 material）；后者不增加顶点、不影响合批（只要不实例化材质）。

---

# 第三部分 性能与工具（20~21）

## 第20章 UI 性能分析

**源码热点映射**（无独立新源码文件，热点都在前文章节）：

| 性能维度 | 对应源码 |
|---------|---------|
| 重建调度 | `CanvasUpdateRegistry.PerformUpdate()`（第 04 章） |
| 顶点生成 | `Graphic.DoMeshGeneration()`、`Text.OnPopulateMesh`、`VertexHelper.FillMesh`（第 02/05/16 章） |
| 顶点膨胀 | `IMeshModifier` 链（Shadow ×2、Outline ×5）（第 17 章） |
| 断批 | `Mask`/`StencilMaterial`、`RectMask2D`、材质实例化（第 09/15 章） |
| 提交 | 引擎 `Canvas.BuildBatch`（第 06 章） |

**验证方式**：Profiler 的 `Canvas.SendWillRenderCanvases`、`Graphic.Rebuild`、`Canvas.BuildBatch` 等采样点。

## 第21章 调试与分析工具

**工具 → 观察点映射**（工具本身无源码）：

| 工具 | 观察什么 | 对应源码概念 |
|------|---------|-------------|
| Frame Debugger | 每个 DrawCall 的批次与断批原因 | 第 09 章合批条件 |
| Profiler UI Module | Rebuild 次数、Batch 数、顶点数 | `CanvasUpdateRegistry` / `Graphic.Rebuild` |
| Profiler Timeline | `Canvas.SendWillRenderCanvases` 频率 | 每帧 Dirty 的 UI 元素 |
| RenderDoc | GPU 侧顶点/片段数据 | `Canvas.BuildBatch` 之后 |

---

# 第四部分 工程实践（22~25）

## 第22章 UI 架构设计

工程模式（UIManager、UI 栈、UIBase 生命周期）**无 UGUI 源码对应**；对接的引擎/uGUI API：`Canvas`（多 Canvas 拆分）、`CanvasGroup`（alpha/raycast 控制）、`Graphic`（`SetVerticesDirty` 重建控制）、`RectTransform`。

## 第23章 DOTween 原理与 UGUI 集成

DOTween 是第三方库（Demigiant/DOTween），**不在 uGUI 仓库**。与 UGUI 的对接点：tween 修改 `RectTransform`/`Graphic.color` → 属性 setter 调 `SetVerticesDirty` → 经 `CanvasUpdateRegistry` 重建。

## 第24章 综合案例与实战

案例涉及的源码（均为前文文件）：`Graphic.cs`（重建）、`CanvasUpdateRegistry.cs`（调度）、`ScrollRect.cs` + `LayoutRebuilder.cs`（虚拟列表）、`IMeshModifier`/`VertexHelper`（特效）、`Canvas`（动静分离）。Profiler 实战对应函数见第 20/21 章映射表。

## 第25章 UI Toolkit 对比分析

UI Toolkit 是独立系统（内置包 `com.unity.ui`，UIElements），**不是 uGUI**。对比阅读要点：`VisualElement`（vs `RectTransform`+`Graphic`）、`PanelSettings`/`UIDocument`（vs `Canvas`）、事件用 `EventDispatcher`（vs `EventSystem`）、布局用 Flexbox（vs `LayoutGroup`）。

---

# 附录：源码阅读速查表

| 想解决的问题 | 打开文件 | 关键方法 |
|-------------|---------|---------|
| UI 什么时候重建 | `CanvasUpdateRegistry.cs` | `PerformUpdate()` |
| 顶点怎么生成 | `Graphic.cs` / `Image.cs` / `Text.cs` | `OnPopulateMesh(VertexHelper)` |
| 顶点存在哪 | `Utility/VertexHelper.cs` | `AddVert/AddTriangle/FillMesh/Clear` |
| 材质怎么进渲染 | `Graphic.cs` | `materialForRendering` / `GetModifiedMaterial` |
| 怎么给顶点加特效 | `VertexModifiers/BaseMeshEffect.cs` | `ModifyMesh(VertexHelper)` |
| 阴影/描边实现 | `VertexModifiers/Shadow.cs`、`Outline.cs` | `GetUIVertexStream` + `AddUIVertexTriangleStream` |
| 布局怎么算 | `Layout/LayoutRebuilder.cs` | `MarkLayoutForRebuild` / `Rebuild` |
| 事件怎么分发 | `EventSystem/ExecuteEvents.cs` | `Execute()` / `GetEventHandler()` |
| 命中检测 | `UI/Core/GraphicRaycaster.cs` | `Raycast()` |
| Stencil 裁剪 | `Mask.cs` / `StencilMaterial.cs` | `GetModifiedMaterial` / `Add` |
| 矩形裁剪 | `RectMask2D.cs` | `EnableRectClipping` |
| 文本顶点 | `Text.cs` | `PopulateWithErrors` / `AddUIVertexQuad` |
| 字体重建通知 | `FontUpdateTracker.cs` | `TrackText` / `UntrackText` |
| 分辨率适配 | `Layout/CanvasScaler.cs` | `HandleScaleWithScreenSize()` |
| 滚动容器 | `ScrollRect.cs` | `LateUpdate()` / `UpdateBounds` |
