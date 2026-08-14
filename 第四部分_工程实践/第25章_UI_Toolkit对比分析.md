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

### 25.3.2 渲染性能

UGUI 的优势在于 DrawCall 合批——Canvas.BuildBatch 对多材质批处理有成熟的优化。UI Toolkit 的渲染器在复杂场景下可能产生更多 DrawCall。

## 25.4 如何选择

### 选 UGUI 的场景

- **游戏 HUD、血条、技能栏**：需要频繁更新、自定义 Shader、复杂交互
- **需要深度控制渲染**：自定义 OnPopulateMesh、Mesh 修改器
- **已有大量 UGUI 项目**：迁移成本高，保持一致性更重要
- **目标平台性能有限**：UGUI 在低端设备上有更成熟的优化方案

### 选 UI Toolkit 的场景

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

注意事项：
- UGUI 在 UI Toolkit 之上渲染（Sorting Order 控制）
- 两者间不能合批
- 事件系统互不干扰（UGUI 用 EventSystem，UI Toolkit 用 UIDocument）

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
| 2 | 🟢 轻微 | 全文 | 其余内容 | UI Toolkit 属独立系统（内置包 `com.unity.ui` / UIElements），不在 uGUI 仓库；对比结论以官方文档为准。本章未做逐条源码核查 |

