# 第1章 UGUI 整体架构

> 本章对应原书结构中的第1章（基础理论部分）。UGUI 是 Unity 官方推出的 UI 系统，全称 Unity UI，于 Unity 4.6 引入。理解 UGUI 的整体架构，是后续章节深入分析各个子系统的前提。

---

## 1.1 为什么要先理解架构

在深入具体组件源码之前，先建立对 UGUI 整体架构的认知，有两个好处：

1. **遇到问题知道去哪找答案**——渲染问题是 Graphic 的职责还是 Canvas 的职责？
2. **避免"只见树木不见森林"**——不先建立架构认知，就会迷失在细节中。

本章回答三个问题：
- UGUI 和 IMGUI 有什么本质区别？
- UGUI 的系统架构是怎样的？
- UGUI 的渲染本质是什么？

---

## 1.2 IMGUI vs UGUI

Unity 在 UGUI 之前，唯一的 UI 方案是 IMGUI（Immediate Mode GUI），即 `OnGUI()` 函数中的 `GUI.Button`、`GUI.Label` 等调用。

### 1.2.1 IMGUI：即时模式

```csharp
// IMGUI：每帧执行绘制代码
void OnGUI() {
    if (GUI.Button(new Rect(10, 10, 100, 50), "Click Me")) {
        Debug.Log("Button clicked");
    }
}
```

IMGUI 的特征：
- **每帧重绘**：`OnGUI()` 每帧被调用，所有 UI 元素每帧重新绘制
- **无状态**：UI 元素不保存状态，每帧从头绘制
- **缺乏组件化**：没有 GameObject + Component 的概念，UI 元素不可复用
- **定位为编辑器工具**：IMGUI 在 Unity 编辑器中大量使用（Inspector、SceneView 等），但在运行时 UI 中已基本被 UGUI 取代

### 1.2.2 UGUI：保留模式（Retained Mode）

```csharp
// UGUI：声明结构，系统维护
// 将 Button 拖到场景中 → 系统只在状态变化时重建
```

UGUI 的特征：
- **保留模式**：UI 结构在场景中持久存在，不是每帧重绘
- **GameObject + Component**：每个 UI 元素是一个 GameObject，上面挂载多个 Component
- **可视化编辑**：通过 SceneView 和 Inspector 拖拽编辑
- **自动批处理**：系统自动合并 DrawCall

### 1.2.3 核心差异

| 维度 | IMGUI | UGUI |
|------|-------|------|
| 渲染模式 | 即时（每帧重绘） | 保留（状态变化时更新） |
| 组织方式 | 代码控制 | GameObject + Component |
| 编辑方式 | 运行时代码 | 编辑器可视化 |
| 性能 | 每帧 CPU 密集 | Dirty 机制 + 延迟重建 |
| 适用场景 | 编辑器工具、调试面板 | 运行时 UI |

---

## 1.3 UGUI 的三层架构

UGUI 系统的整体架构可以分为三个逻辑层：

```
┌──────────────────────────────────────┐
│           组件层（Component Layer）     │
│  Graphic / Image / Text / Button     │
│  LayoutGroup / ScrollRect / Mask     │
│  负责：生成UI数据、响应事件、定义布局    │
├──────────────────────────────────────┤
│           系统层（System Layer）       │
│  CanvasUpdateRegistry / Canvas       │
│  EventSystem / LayoutRebuilder       │
│  负责：调度更新、合批提交、事件分发      │
├──────────────────────────────────────┤
│           渲染层（Render Layer）       │
│  CanvasRenderer / Mesh / Material    │
│  GPU Shader / Command Buffer         │
│  负责：将数据送入GPU、绘制到屏幕        │
└──────────────────────────────────────┘
```

### 组件层

组件层是开发者直接接触最多的层面。每个 UI 元素由若干 Component 组成：

- **Graphic 系**（继承自 Graphic）：Image、RawImage、Text——负责生成顶点数据
- **交互系**（继承自 Selectable）：Button、Toggle、Slider、Dropdown——负责交互状态
- **布局系**（继承自 LayoutGroup）：LayoutGroup 及其子类——负责子元素排列

### 系统层

系统层负责调度和协调：

- **CanvasUpdateRegistry**：统一调度 Layout 重建和 Graphic 重建
- **Canvas**：渲染入口，触发 BuildBatch 合批
- **EventSystem**：事件调度器，管理输入模块和射线检测

### 渲染层

渲染层负责将 UI 数据提交给 GPU：

- **CanvasRenderer**：每个 UI 元素对应一个，存储该元素的 Mesh 和 Material
- **BuildBatch**（引擎 native 方法）：遍历 CanvasRenderer，合并同材质同纹理的 Mesh，提交 DrawCall

---

## 1.4 UGUI 的渲染本质

这是整个 UGUI 体系中最重要的一句话：

> **所有 UI 最终都转换为 Mesh 数据提交给 GPU。**

在 UGUI 眼中，不存在"按钮"、"图片"、"文字"这些概念——只有**顶点**、**三角形**、**UV** 和**颜色**。

一个 Image 在 GPU 眼里就是这样的：

```
4 个顶点（四个角）→ 2 个三角形（6 个索引）
→ 一个 Quad（四边形网格）
→ 贴上纹理 → 一个 DrawCall
```

### 从 UI 元素到屏幕像素的完整链路

```
Image/Text 组件
  ↓ OnPopulateMesh(VertexHelper vh)
Mesh（顶点 + 三角形 + UV + 颜色）
  ↓ canvasRenderer.SetMesh(mesh)
CanvasRenderer（存储 Mesh）
  ↓ Canvas.BuildBatch()（引擎 native 方法）
合并 + 提交 DrawCall
  ↓
GPU Vertex Shader → 光栅化 → Fragment Shader → FrameBuffer
```

### 源码视角：Graphic.cs 中的 Rebuild

```csharp
// Graphic.cs（UGUI 核心文件，路径：UnityEngine.UI/UI/Core/Graphic.cs）
public class Graphic : UIBehaviour, ICanvasElement {
    // —— Dirty 标记 ——
    public virtual void SetVerticesDirty() {
        m_VertsDirty = true;
        // 将自己注册到 CanvasUpdateRegistry 的 Graphic 重建队列
        CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
    }

    // —— 重建入口 ——
    public virtual void Rebuild(CanvasUpdate update) {
        if (update == CanvasUpdate.PreRender) {
            UpdateGeometry();   // 阶段一：更新顶点
            UpdateMaterial();   // 阶段二：更新材质
        }
    }

    // —— 顶点重建 ——
    private void UpdateGeometry() {
        // 1. 调用子类的 OnPopulateMesh 生成顶点
        DoMeshGeneration();
        // 2. 将 Mesh 提交给 CanvasRenderer
        canvasRenderer.SetMesh(m_WorkerMesh);
    }

    // —— 子类重写此方法生成自己的顶点 ——
    protected virtual void OnPopulateMesh(VertexHelper vh) {
        // Image 类重写 → 生成 4 个顶点 + 2 个三角形
        // Text 类重写 → 每个字符生成一个 Quad
    }

    // —— 材质更新 ——
    private void UpdateMaterial() {
        canvasRenderer.SetMaterial(material, mainTexture);
    }
}
```

### 三种 Canvas 渲染模式

所有 UI 必须放在 Canvas 下。Canvas 提供三种渲染模式（由 `Canvas.renderMode` 控制）：

```csharp
// Canvas.cs（路径：UnityEngine.UI/UI/Core/Canvas.cs）
public class Canvas : Behaviour {
    public RenderMode renderMode;   // ScreenSpaceOverlay / ScreenSpaceCamera / WorldSpace
    public float scaleFactor;      // 由 CanvasScaler 动态修改
}
```

| RenderMode | 说明 | 典型场景 |
|-----------|------|---------|
| ScreenSpaceOverlay | 覆盖在场景之上，不需要 Camera | 最常用，HUD、主界面 |
| ScreenSpaceCamera | 由指定 Camera 渲染，支持后处理 | 需要 Post-Processing 的 UI |
| WorldSpace | 在 3D 空间中的 UI | 3D 世界中的标签、交互面板 |

---

## 1.5 源码结构概览

UGUI 的 C# 源码以 Unity Package 形式发布，核心代码位于 `UnityEngine.UI/UI/Core/` 目录下。

### 核心目录与类继承

```
UnityEngine.UI/UI/Core/
├── Canvas.cs / CanvasRenderer.cs   ← 渲染入口和 Mesh 容器
├── CanvasUpdateRegistry.cs         ← UI 更新调度器
├── Graphic.cs                      ← 所有可视 UI 的抽象基类
│    └── MaskableGraphic.cs         ← 支持 Mask 裁剪
│         ├── Image.cs / RawImage.cs / Text.cs
├── VertexHelper.cs                 ← 顶点构建工具
├── Layout/                         ← 布局系统
├── Mask.cs / RectMask2D.cs         ← 裁剪系统
├── EventSystem.cs / ExecuteEvents.cs ← 事件系统
├── Selectable.cs                   ← 交互组件基类
│    ├── Button / Toggle / Slider / Scrollbar / Dropdown
└── StencilMaterial.cs              ← Stencil 材质管理
```

### 本章涉及的核心文件

| 文件 | 类 | 职责 |
|------|----|------|
| `Canvas.cs` | Canvas | 渲染入口，触发 BuildBatch |
| `CanvasRenderer.cs` | CanvasRenderer | 保存 UI 元素的 Mesh 和 Material |
| `Graphic.cs` | Graphic | 抽象基类，定义 Rebuild 和 Dirty 机制 |
| `CanvasUpdateRegistry.cs` | CanvasUpdateRegistry | 统一调度所有 UI 更新 |

---

## 1.6 UGUI 与其它 UI 系统的设计差异

理解 UGUI 的设计哲学，有助于在实际项目中做出更好的技术决策。

### 1.6.1 与 FlexBox（CSS）的差异

| 维度 | UGUI（LayoutGroup） | CSS FlexBox |
|------|-------------------|-------------|
| 布局计算 | 两阶段（水平→垂直） | 一阶段（主轴→交叉轴） |
| 子元素反馈 | ILayoutElement 反馈尺寸需求 | flex-grow/flex-shrink 分配空间 |
| 编辑方式 | 编辑器拖拽 + Inspector 参数 | 书写 CSS 代码 |

### 1.6.2 与 FMOD/NoesisGUI 的差异

第三方 UI 方案的特点是 **CPU 不参与 UI 网格更新**——顶点生成和更新在 GPU 端完成。而 UGUI 的 UI 元素改变时会触发 CPU 端的 `OnPopulateMesh` 重新生成顶点数据。这是 UGUI 的性能热点：**频繁修改 UI 内容（如每帧更新 Text）会导致 CPU 侧持续重建 Mesh**。

### 1.6.3 UGUI 的设计倾向总结

```
保留模式 → 按需重建，不是每帧重绘
GameObject+Component → 组件化，可组合，可复用
延迟重建 → Dirty 标记 + 统一调度
两阶段布局 → 先算尺寸再分配空间
CPU 生成 Mesh → 灵活但 CPU 开销大
合批提交 → 自动合并，受材质/纹理/Stencil 约束
```

---

## 1.7 本章总结

1. **IMGUI vs UGUI**：IMGUI 是即时模式（每帧重绘），UGUI 是保留模式（状态变化时更新）。IMGUI 适合编辑器工具，UGUI 适合运行时 UI。

2. **三层架构**：组件层（Graphic/Image/Button）→ 系统层（CanvasUpdateRegistry/Canvas/EventSystem）→ 渲染层（CanvasRenderer/GPU）

3. **渲染本质**：所有 UI 最终都转换为 Mesh 数据提交给 GPU。Image、Text 的本质都是 Mesh + Material + Texture。

4. **Dirty 标记机制**：UI 变化不立即重建，而是通过 `SetVerticesDirty()` 标记，由 `CanvasUpdateRegistry` 在下一帧统一调度。这是 UGUI 性能的基石。

5. **源码入口**：`Graphic.cs` 中的 `SetVerticesDirty()` / `Rebuild()` / `UpdateGeometry()` / `OnPopulateMesh()` 是理解 UGUI 渲染流程的起点。

### 推荐的源码阅读路径

```
打开 Graphic.cs → 依次阅读：
  1. SetVerticesDirty()    ← 理解 Dirty 标记机制
  2. SetMaterialDirty()    ← 同上
  3. Rebuild()             ← 理解重建流程的两个阶段
  4. OnPopulateMesh()      ← 虚方法，子类实现生成顶点
  5. UpdateGeometry()      ← 理解 Mesh 如何提交给 CanvasRenderer
```
