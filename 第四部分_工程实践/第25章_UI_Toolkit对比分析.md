# 第25章 UI Toolkit 对比分析

> UI Toolkit（原名 UIElements）是 Unity 主推的新 UI 系统，与 UGUI 在架构设计上有根本差异。理解两者的异同，有助于在项目中做正确的技术选型。

## 25.1 架构对比

### 25.1.1 UGUI：GameObject + Component

UGUI 的每个 UI 元素是一个 GameObject：
- 继承关系对应 GameObject 层级树
- 排版通过 LayoutGroup/ContentSizeFitter 等组件驱动
- 样式通过 Inspector 参数逐元素设置
- 事件通过 C# 接口（IPointerClickHandler 等）分发

### 25.1.2 UI Toolkit：Visual Tree + USS + UXML

UI Toolkit 基于 Web 技术模型：
- VisualElement 树（轻量级，非 GameObject）
- USS 样式表（类似 CSS）统一控制外观
- UXML 模板（类似 HTML）声明 UI 结构
- 事件冒泡机制（类似 DOM 事件流）

```
UGUI 的 UI 树：                    UI Toolkit 的 Visual Tree：
Canvas                              UIDocument
└── GameObject (Image)              └── VisualElement
    └── GameObject (Text)               ├── Label
    └── GameObject (Button)             ├── Button
        └── Text (子对象)               └── VisualElement (容器)
```

## 25.2 关键差异

| 维度 | UGUI | UI Toolkit |
|------|------|-----------|
| 构建方式 | 编辑器拖拽 + Inspector | UXML 声明 + USS 样式 |
| 元素类型 | GameObject + Component | VisualElement（轻量级） |
| 布局系统 | LayoutGroup（OOP 组件） | Flexbox（CSS 标准） |
| 样式系统 | 逐元素 Inspector 设置 | USS 样式表（集中控制） |
| 事件系统 | C# 接口 + ExecuteEvents | 事件冒泡 + 回调 |
| 渲染方式 | Canvas.BuildBatch() | 内部自建渲染器 |
| 合批 | 自动按 Material/Texture 合并 | 自动 + 自定义 Shader |
| 自定义 Shader | UI/Default 继承 | 原生自定义 Shader |
| 脚本交互 | GetComponent + 直接属性 | VisualElement.Query + Data Binding |
| 编辑器支持 | SceneView 实时编辑 | UI Builder 可视化编辑 |
| 运行时 vs 编辑器 | 主要用于运行时 | 编辑器和运行时双用 |
| 性能 | 大量 GameObject 时开销大 | 轻量级元素，适合大规模 UI |

### 25.2.1 布局系统差异

```csharp
// UGUI：HorizontalLayoutGroup 代码控制
public class MyLayout : LayoutGroup {
    public override void SetLayoutHorizontal() {
        // 遍历子节点，调用 SetChildAlongAxis
    }
}

// UI Toolkit：Flexbox 声明式
// USS 文件：
// .container { display: flex; flex-direction: row; }
// .item { flex-grow: 1; }
```

UI Toolkit 的 Flexbox 布局更接近 Web 开发习惯，且支持响应式设计（`@media` 查询）。

### 25.2.2 事件系统差异

```
UGUI 事件流：                    UI Toolkit 事件流：
EventSystem.Update()             UIDocument 调度
  → InputModule.Process()          → TrickleDown（从根到目标）
    → Raycaster.Raycast()          → Target（目标元素）
      → ExecuteEvents.Execute()    → BubbleUp（从目标到根）
```

UI Toolkit 的事件冒泡机制比 UGUI 更灵活——父容器可以捕获子元素的事件，适用于通用事件处理。

## 25.3 性能对比

### 25.3.1 元素实例化

| 操作 | UGUI | UI Toolkit | 差距来源 |
|------|------|-----------|------|
| 批量创建元素 | 较慢 | **明显更快** | UGUI 每个元素是 GameObject（Transform 层级、组件初始化、注册到各种 Registry）；VisualElement 是纯托管对象 |
| 批量修改外观 | 较慢 | **更快** | UGUI 逐个 `SetVerticesDirty` 触发顶点重建；UI Toolkit 改 USS 属性走自己的脏标记体系 |
| 批量销毁元素 | 较慢 | **明显更快** | `GameObject.Destroy` 的开销远大于从 Visual Tree 摘掉一个节点 |

> ⚠️ 这里刻意不给具体毫秒数。此类数字高度依赖 Unity 版本、目标设备、Editor 还是 Build、元素结构复杂度，网上流传的对比数据往往缺少这些前提，直接引用容易得出错误结论。**要做选型决策，请在你的目标设备上对你的真实界面做一次实测。** 上表只表达方向性差距。

### 25.3.2 渲染路径：两套完全独立的系统

这是全章最需要理解的一点——**UI Toolkit 根本不经过 UGUI 的渲染链路**：

```
UGUI：
  Graphic.OnPopulateMesh → VertexHelper → CanvasRenderer.SetMesh
    → Canvas.BuildBatch()（引擎 native）→ DrawCall

UI Toolkit：
  VisualElement 的 generateVisualContent → 自己的 MeshGenerationContext
    → UIElements 内部渲染器自行合并 → DrawCall
```

`Canvas`、`CanvasRenderer`、`CanvasUpdateRegistry`、`Canvas.BuildBatch` —— 本书前 24 章讲的这一整套，在 UI Toolkit 里**一个都不用**。它有自己的脏标记、自己的布局求解（Yoga/Flexbox）、自己的网格生成和批处理。

由此推出三条实际影响：

| 影响 | 说明 |
|------|------|
| **两者永远不能合批** | 不是"条件不满足"，是根本不在同一个批处理系统里。混用时 DrawCall 是两边各自的总和 |
| **UGUI 的合批知识不迁移** | 第 9 章那套"同材质 + 同纹理 + 同 Stencil"的规则对 UI Toolkit 不适用，它有自己的批处理边界 |
| **性能诊断工具不同** | Frame Debugger 仍能看到 DrawCall，但 UGUI 的 `Canvas.BuildBatch` / `Graphic.Rebuild` 采样点在 UI Toolkit 侧不存在，要看 UIElements 自己的 Profiler 标记 |

至于哪边 DrawCall 更少：**取决于界面结构，没有普适结论**。UGUI 的合批在"大量元素共享同一图集"时效率很高；UI Toolkit 在"元素多但结构规整"时也有自己的优化。选型不要押在这一条上。

### 25.3.3 自定义渲染能力的差距

这是 UGUI 目前仍然明显占优的地方，也是很多项目最终留在 UGUI 的原因：

| 能力 | UGUI | UI Toolkit |
|------|------|-----------|
| 逐元素换材质 | `graphic.material = myMat`，任意自定义 Shader | 运行时受限，没有等价的"给某个 VisualElement 挂任意材质"的通用做法 |
| 自定义网格生成 | 继承 `Graphic` 重写 `OnPopulateMesh`（第 5 章 5.9） | `generateVisualContent` 回调 + `MeshGenerationContext`，能力和生态都不如前者成熟 |
| 顶点后处理 | `IMeshModifier` 链，Shadow/Outline/渐变（第 17 章） | 无对应机制 |
| 现成特效生态 | 大量第三方 UI 特效插件基于 `BaseMeshEffect` / 自定义 Shader | 少 |

所以第 17、18、19 三章讲的东西——顶点特效、自定义 UI Shader、描边发光溶解——**在 UI Toolkit 里基本无法照搬**。

> 以上为系统能力层面的对比，随 Unity 版本演进会变化，以你使用版本的官方文档为准。

## 25.4 如何选择

### 25.4.1 选 UGUI 的场景

- **游戏 HUD、血条、技能栏**：需要频繁更新、自定义 Shader、复杂交互
- **需要深度控制渲染**：自定义 OnPopulateMesh、Mesh 修改器
- **已有大量 UGUI 项目**：迁移成本高，保持一致性更重要
- **目标平台性能有限**：UGUI 在低端设备上有更成熟的优化方案

### 25.4.2 选 UI Toolkit 的场景

- **数据驱动 UI**：背包、商城、设置面板——结构固定、数据变化
- **编辑器扩展**：UI Toolkit 本身是 Unity 编辑器 UI 的未来
- **跨平台运行时 UI**：复杂布局、多分辨率适应
- **团队有 Web 开发经验**：USS + UXML + C# 的开发模式与前端一致

## 25.5 共存策略

两者可以在同一项目中共存：

```
场景中同时运行 UGUI HUD + UI Toolkit 菜单：

UGUI Canvas（Overlay）
  └── HUD（实时更新、自定义 Shader 特效）

UIDocument（Panel Settings）
  └── 主菜单 / 背包 / 设置（数据驱动、布局复杂）
```

共存时真正会踩的是**两条边界**：排序边界和输入边界。

### 25.5.1 排序边界：只能整块压，不能交错

UGUI 的层级由 `Canvas.sortingOrder` 决定，UI Toolkit 的层级由 `PanelSettings` 的 Sort Order 决定——**两套独立的排序系统**。它们之间只能比较"整个 Canvas"和"整个 Panel"谁在上，**无法让某个 UGUI 元素插进 UI Toolkit 的两个元素中间**。

```
可以做到：
  Panel（主菜单，Sort Order = 0）
  Canvas（HUD，sortingOrder = 10）      → HUD 整体压在主菜单之上 ✅

做不到：
  想让一个 UGUI 特效插在 UI Toolkit 的背景和按钮之间 ❌
```

**设计含义**：混用时必须按"整屏功能块"划分归属，不能按元素混排。一个界面里既要 UGUI 特效又要 UI Toolkit 布局，通常意味着这个界面应该整体选一边。

### 25.5.2 输入边界：谁吃掉了这次点击

两套系统都要处理鼠标/触摸，**必须让它们协商谁消费这次输入**，否则会出现"点了 UI Toolkit 的按钮，底下 UGUI 的按钮也响应了"这类问题。

Unity 的做法是把 UI Toolkit 的运行时面板接入 EventSystem：`UIDocument` 在检测到场景中有 EventSystem 时，会配合 `PanelRaycaster` / `PanelEventHandler` 把面板作为一个 Raycaster 参与命中检测——这样第 12 章讲的那套 `RaycastAll` → 排序 → 分发流程就能同时看到两边。

实践上要注意：

- **确认场景里有 EventSystem**，UI Toolkit 运行时面板依赖它做输入协商
- **UGUI 侧照旧关无用的 `raycastTarget`**（第 12 章 12.3.7），减少参与竞争的元素
- **不要两边都自己读 `Input`**：绕过 EventSystem 直接轮询会让协商失效
- 出现"点击穿透"时，先用 `EventSystem.current.IsPointerOverGameObject()` 和 UI Toolkit 侧的命中结果分别确认是谁接住了事件

> 输入协商的具体组件与行为随 Unity 版本变化较大，以你使用版本的官方文档为准。这里给的是排查思路，不是版本无关的 API 承诺。

---

## 本章总结

```
UI Toolkit 不是"新版 UGUI"，是另一套系统

架构：VisualElement 树 + USS + UXML（Web 模型）
      vs GameObject + Component + Inspector（Unity 模型）

渲染：两条完全独立的链路，一个都不共用
      UGUI：OnPopulateMesh → CanvasRenderer → Canvas.BuildBatch
      UIT： generateVisualContent → UIElements 自己的渲染器
      → 永远不能合批；UGUI 的合批知识不迁移

能力：UGUI 在自定义渲染上明显占优
      逐元素换材质 / OnPopulateMesh / IMeshModifier 链 / 特效生态
      → 第 17、18、19 章的内容在 UI Toolkit 里基本无法照搬

共存：只能整块压，不能交错（排序）
      必须让 EventSystem 协商输入归属（输入）
      → 混用要按"整屏功能块"划分，不要按元素混排
```

**选型的一句话**：需要频繁更新 + 自定义 Shader + 顶点特效的，留在 UGUI；结构规整、数据驱动、布局复杂的，UI Toolkit 更省事。**不要为了"用新技术"迁移已有的 UGUI 界面**——本章列的能力差距会在迁移到一半时集中爆发。

---

## 源码阅读路径

```
UI Toolkit 不是 uGUI：源码在引擎内置包 com.unity.ui（UnityCsReference/UIElements）。
对照阅读：VisualElement vs RectTransform+Graphic、PanelSettings/UIDocument vs Canvas、
EventDispatcher vs EventSystem、Flexbox vs LayoutGroup。
```

---

## 勘误汇总

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🟡 中等 | 25.3.1 | 给出「创建 1000 个 Image：UGUI ~8-15ms / UI Toolkit ~2-5ms」等精确到毫秒的对比数字，但未说明 Unity 版本、设备、Editor/Build、元素结构等任何测量前提 | 选型章最容易被直接当作决策依据，无前提的数字比不给更危险。已改为方向性对比，并注明需在目标设备实测 |
| 2 | 🟡 中等 | 25.3.2 原「渲染性能」 | 只用两句话带过（「UGUI 的优势在于 DrawCall 合批……UI Toolkit 在复杂场景下可能产生更多 DrawCall」），未说明两者是**完全独立的渲染系统** | 已重写：UI Toolkit 不经过 `Canvas` / `CanvasRenderer` / `BuildBatch`，有自己的网格生成与批处理。原文那句"谁 DrawCall 更多"的暗示也不成立——取决于界面结构，无普适结论 |
| 3 | 🟡 中等 | 25.5 原「共存策略」 | 三条注意事项过于笼统（「UGUI 在 UI Toolkit 之上渲染」「事件系统互不干扰」） | 「互不干扰」是错的——两者都要处理输入，必须经 EventSystem 协商归属，否则点击穿透。排序也不是简单的谁上谁下，而是**只能整块压、不能交错**。已拆为 25.5.1 排序边界 / 25.5.2 输入边界 |
| 4 | 🟢 轻微 | 全文 | 缺「本章总结」段；自定义渲染能力差距（材质 / OnPopulateMesh / IMeshModifier）完全未提 | 已补总结段与 25.3.3 能力对比表——这是第 17/18/19 章内容能否迁移的关键，也是很多项目最终留在 UGUI 的原因 |
| 5 | 🟢 轻微 | 全文 | 其余内容 | UI Toolkit 属独立系统（内置包 `com.unity.ui` / UIElements），不在 uGUI 仓库，**无法按本书的 main 基线核查**；相关结论以所用 Unity 版本的官方文档为准，正文已就版本敏感处加注 |

