# 第11章 EventSystem 事件系统

> 本文是对原文的结构化重写，按实际执行流程组织内容，消除原文小节间的重复。对照 UGUI 源码修正了 5 处错误。

---

## 概述：从一次点击看完整链路

在你点击一个 Button 的瞬间，EventSystem 内部发生了这些事情：

```
① 输入采集
  鼠标/触摸 → InputModule.Process() → 构建 PointerEventData（位置、按键状态等）

② 射线检测
  EventSystem.RaycastAll() → RaycasterManager 获取所有 Raycaster
    → GraphicRaycaster.Raycast() → GraphicRegistry 取 Graphic 列表
      → RectangleContainsScreenPoint() 矩形判断 → graphic.Raycast() 附加过滤
        → 排序 → 生成 RaycastResult

③ 事件分发
  ExecuteEvents.ExecuteHierarchy(target, eventData, handler)
    → 按 PointerDown → PointerUp → PointerClick 顺序调用接口

④ UI 响应
  Button.OnPointerClick() → Press() → m_OnClick.Invoke()
```

整个链路每一帧都在 Update 中驱动。下面按这四个阶段逐层展开。

---

## 11.1 EventSystem 总体架构

### 11.1.1 组件结构

场景中创建 UI 时 Unity 自动生成 EventSystem 对象：

```
EventSystem GameObject
  ├── EventSystem 组件          ← 核心调度器
  └── StandaloneInputModule    ← 输入采集（旧输入系统）
```

EventSystem 本身不读取鼠标键盘，它只做三件事：**管理 InputModule → 触发 Raycast → 分发事件**。

### 11.1.2 每帧执行流程

```csharp
// EventSystem.cs
protected virtual void Update()
{
    if (current != this) return;
    TickModules();

    // 1. 选择当前激活的 InputModule
    for (int i = 0; i < m_SystemInputModules.Count; i++)
    {
        var module = m_SystemInputModules[i];
        if (module.IsModuleSupported() && module.ShouldActivateModule())
        {
            if (m_CurrentInputModule != module)
                ChangeEventModule(module);
            break;
        }
    }

    // 2. 执行 InputModule 的 Process()
    if (!changedModule && m_CurrentInputModule != null)
        m_CurrentInputModule.Process();
}
```

`m_CurrentInputModule.Process()` 这一行是真正的入口——输入采集、Raycast、事件分发全部从这里开始。

### 11.1.3 核心组件职责一览

| 组件 | 角色 | 一句话 |
|------|------|--------|
| **EventSystem** | 调度中心 | 选 InputModule、调 RaycastAll、管理选中对象 |
| **InputModule** | 输入采集 | 读设备输入 → 构建 PointerEventData |
| **GraphicRaycaster** | UI 命中 | 遍历 Graphic 做矩形包含判断 |
| **PhysicsRaycaster** | 3D 命中 | 发射 `Physics.RaycastAll` |
| **GraphicRegistry** | 缓存 | Canvas → Graphic 列表的全局映射，避免每次 Raycast 遍历全场景 |
| **RaycasterManager** | 注册表 | 维护所有 Raycaster 的全局 List |
| **ExecuteEvents** | 事件执行 | 静态方法，调用目标上的接口回调 |

---

## 11.2 输入采集：InputModule 与 PointerEventData

### 11.2.1 InputModule 继承结构

```
BaseInputModule
  └── PointerInputModule（处理指针类输入）
       ├── StandaloneInputModule      ← 旧输入系统
       ├── TouchInputModule
       └── InputSystemUIInputModule   ← 新输入系统
```

不同 InputModule 的 `Process()` 实现不同，但最终都输出同一种东西：**PointerEventData**。

### 11.2.2 StandaloneInputModule.Process()

```csharp
public override void Process()
{
    // 第一部分：更新当前选中对象（如 InputField 的输入状态）
    bool usedEvent = SendUpdateEventToSelectedObject();

    // 第二部分：触摸优先，其次鼠标
    if (!ProcessTouchEvents() && input.mousePresent)
        ProcessMouseEvent();

    // 第三部分：键盘/手柄导航事件
    if (eventSystem.sendNavigationEvents)
    {
        if (!usedEvent) usedEvent |= SendMoveEventToSelectedObject();
        if (!usedEvent) SendSubmitEventToSelectedObject();
    }
}
```

`ProcessMouseEvent()` 内部会调用 `GetMousePointerEventData()`，这一步触发 `eventSystem.RaycastAll()`——Raycast 就发生在这里。

### 11.2.3 PointerEventData：跨帧输入状态容器

一次点击从按下到抬起可能跨越多帧，中间状态全部保存在 PointerEventData 中：

| 字段 | 含义 |
|------|------|
| `position` | 当前鼠标/触摸屏幕坐标 |
| `delta` | 与上一帧的位置差 |
| `button` | 哪个键（Left / Right / Middle） |
| `pointerEnter` | 当前悬停的 GameObject |
| `pointerPress` | 当前按下的 GameObject（Click 判定用） |
| `lastPress` | 上一帧的 Press 对象 |
| `pointerCurrentRaycast` | 当前帧的 Raycast 命中结果 |
| `eligibleForClick` | 是否仍有资格触发 Click（拖拽开始后变为 false） |
| `dragging` | 是否正在拖拽 |
| `clickCount` | 连续点击次数（双击判定用） |
| `clickTime` | 上次点击的 `Time.unscaledTime` |
| `scrollDelta` | 滚轮增量 |

每次鼠标移动、按下、抬起，都会刷新 PointerEventData 并重新执行 RaycastAll。

---

## 11.3 射线检测体系

### 11.3.1 EventSystem.RaycastAll：统一调度入口

```csharp
public void RaycastAll(PointerEventData eventData, List<RaycastResult> raycastResults)
{
    raycastResults.Clear();
    var modules = RaycasterManager.GetRaycasters();   // 从全局注册表获取所有 Raycaster

    for (int i = 0; i < modules.Count; ++i)
    {
        var module = modules[i];
        if (module == null || !module.IsActive()) continue;
        module.Raycast(eventData, raycastResults);    // 每个 Raycaster 各自实现检测
    }

    raycastResults.Sort(s_RaycastComparer);            // 合并结果，统一排序
}
```

EventSystem 不关心 Raycaster 的具体类型——它只管调度。所有 Raycaster 的结果会被合并后统一按 Sorting Layer → Sorting Order → Depth → Distance 排序。

### 11.3.2 RaycasterManager：全局 Raycaster 注册表

```csharp
internal static class RaycasterManager
{
    private static readonly List<BaseRaycaster> s_Raycasters = new List<BaseRaycaster>();
    
    public static List<BaseRaycaster> GetRaycasters() => s_Raycasters;
    public static void AddRaycaster(BaseRaycaster r) { /* 去重后加入 */ }
    public static void RemoveRaycasters(BaseRaycaster r) { /* 移除 */ }
}
```

注册/注销在 `BaseRaycaster.OnEnable` / `OnDisable` 中自动完成。开发者通常不需要手动管理。

### 11.3.3 三种 Raycaster 的实现机制

统一接口 `BaseRaycaster.Raycast()`，三种实现完全不同：

| | GraphicRaycaster | PhysicsRaycaster | Physics2DRaycaster |
|------|------|------|------|
| 检测对象 | Graphic（RectTransform） | Collider（3D） | Collider2D |
| 检测方式 | 屏幕坐标 + 矩形包含判断 | 从屏幕点发射 3D 射线 | 从屏幕点发射 2D 射线 |
| 核心 API | `RectangleContainsScreenPoint` | `Physics.RaycastAll` | `Physics2D.GetRayIntersectionAll` |
| 需要 Camera | Overlay 模式传 null | 必须（`camera.ScreenPointToRay`） | 必须 |
| 排序依据 | Graphic.depth | 射线距离 | 射线距离 |

**继承结构**：

```
BaseRaycaster
├── GraphicRaycaster
└── PhysicsRaycaster
     └── Physics2DRaycaster（继承自 PhysicsRaycaster）
```

**PhysicsRaycaster 的大致实现**（与 GraphicRaycaster 对比）：

```csharp
// PhysicsRaycaster.Raycast()
Ray ray = eventCamera.ScreenPointToRay(eventData.position);  // 屏幕 → 3D 射线
RaycastHit[] hits = Physics.RaycastAll(ray, maxDistance, layerMask);
foreach (var hit in hits)
    resultAppendList.Add(new RaycastResult { gameObject = hit.collider.gameObject, /*...*/ });
```

只有 GraphicRaycaster 是"屏幕矩形判断"；PhysicsRaycaster 和 Physics2DRaycaster 都在发射真正的物理射线。

### 11.3.4 GraphicRaycaster.Raycast() 完整流程

```
GraphicRegistry.GetGraphicsForCanvas(canvas)   ← 从缓存获取当前 Canvas 的 Graphic 列表（O(1)）
  │
  ├─ 过滤 1：depth == -1        → 跳过（未参与渲染）
  ├─ 过滤 2：raycastTarget == false → 跳过（不接收射线）
  ├─ 过滤 3：canvasRenderer.cull → 跳过（已被 RectMask2D / Mask 裁剪）
  │
  ├─ 矩形检测：RectTransformUtility.RectangleContainsScreenPoint()
  │    → 鼠标屏幕坐标是否在 RectTransform 矩形区域内？
  │
  ├─ 附加过滤：graphic.Raycast(sp, camera)
  │    → Graphic 虚方法，默认 return true，Image 可重写做 Alpha Hit Test
  │
  ├─ 按 depth（CanvasRenderer.absoluteDepth）降序排序
  │
  └─ 生成 RaycastResult 列表并返回
```

### 11.3.5 两个 "Raycast" 的区别（重要）

名字都叫 Raycast，但层次不同：

| | GraphicRaycaster.Raycast() | Graphic.Raycast() |
|------|------|------|
| 所属类 | GraphicRaycaster（继承自 BaseRaycaster） | Graphic（UI 组件的基类） |
| 角色 | Raycaster 层：遍历所有 Graphic + 矩形检测 + 排序 | Graphic 层：单个元素的附加过滤 |
| 调用方 | EventSystem.RaycastAll → RaycasterManager | GraphicRaycaster.Raycast() 内部 |
| 默认行为 | 完整命中检测流程 | `return true`（矩形命中就算） |
| 重写者 | 一般不重写 | Image 重写做 Alpha Hit Test |

调用关系是 **GraphicRaycaster.Raycast() 内部调用 Graphic.Raycast()**。

### 11.3.6 GraphicRegistry：Canvas → Graphic 映射

如果每帧 Raycast 都遍历整个场景查找 Graphic，大场景下 CPU 开销不可接受。GraphicRegistry 提前缓存了映射：

```csharp
public class GraphicRegistry
{
    private readonly Dictionary<Canvas, IndexedSet<Graphic>> m_Graphics;
    
    public static void RegisterGraphicForCanvas(Canvas c, Graphic graphic) { /*...*/ }
    public static void UnregisterGraphicForCanvas(Canvas c, Graphic graphic) { /*...*/ }
    public static IList<Graphic> GetGraphicsForCanvas(Canvas canvas) { /*...*/ }
}
```

**注册**：`Graphic.OnEnable()` → `GraphicRegistry.RegisterGraphicForCanvas(canvas, this)`

**注销**：`Graphic.OnDisable()` → `GraphicRegistry.UnregisterGraphicForCanvas(canvas, this)`

**查询**：`GraphicRegistry.GetGraphicsForCanvas(canvas)`——Dictionary 查询，O(1)。

`IndexedSet<Graphic>` 是 UGUI 自定义容器，内部同时维护 `List<T>`（保持遍历顺序）和 `Dictionary<T, int>`（O(1) 去重）。

### 11.3.7 RaycastTarget 属性详解

`raycastTarget` 是 `Graphic` 类上的 `[SerializeField]`，默认 `true`。**所有 Graphic 子类都有**：Image、Text、RawImage、MaskableGraphic 等。TMP_Text 不继承 Graphic，但有独立的同名属性。

GraphicRaycaster 中的过滤：

```csharp
if (!graphic.raycastTarget)
    continue;  // 直接跳过
```

**三种操作方式：**

1. **Inspector**：选中组件，底部 `Raycast Target` 复选框取消勾选
2. **代码**：`GetComponent<Image>().raycastTarget = false;`
3. **批量清理**：
```csharp
void DisableRaycastOnChildren(Transform parent)
{
    foreach (var graphic in parent.GetComponentsInChildren<Graphic>())
        graphic.raycastTarget = false;
}
```

**哪些该关：**

| 关掉 | 保留 |
|------|------|
| 纯装饰背景 Image | 可点击的 Button |
| 标签/说明文字 Text | 需要拖拽的 Slider 滑块 |
| 仅用于布局的占位 Graphic | InputField 自身 |
| ScrollView 内容区的缩略图 | 需要 Hover 检测的交互区域 |

**Text 是重灾区**：聊天列表、任务列表、背包物品名——上百个 Text 全开着 raycastTarget 时，鼠标每动一像素就遍历这上百个做矩形检测，而且 Text 覆盖在 Button 上还会阻挡点击事件向上冒泡。

> 注意：关闭 `raycastTarget` 只影响事件检测，不影响渲染显示。

---

## 11.4 事件分发：ExecuteEvents 与点击生命周期

### 11.4.1 ExecuteEvents：静态事件执行器

所有事件最终都通过 `ExecuteEvents` 这个静态类分发：

```csharp
// 仅在 target 上查找
ExecuteEvents.Execute(target, eventData, ExecuteEvents.pointerClickHandler);

// 沿层级向上查找（事件冒泡）
ExecuteEvents.ExecuteHierarchy(target, eventData, ExecuteEvents.pointerClickHandler);
```

内部维护的事件接口列表：

```
pointerEnterHandler     pointerExitHandler
pointerDownHandler      pointerUpHandler      pointerClickHandler
beginDragHandler        dragHandler           endDragHandler
initializePotentialDrag  dropHandler           scrollHandler
updateSelectedHandler   selectHandler          deselectHandler
moveHandler             submitHandler          cancelHandler
```

**UGUI 的事件系统本质上就是接口回调系统**——GameObject 实现了哪个接口，就能收到对应事件。

### 11.4.2 ExecuteHierarchy 事件冒泡

点击 Text 子节点时，Button 父节点仍然能收到事件：

```
点击 Text 子节点 → Text 没有 IPointerClickHandler → 向上找父节点
  → 父节点 Button 实现了 IPointerClickHandler → Button.OnPointerClick() 被调用
```

源码内逻辑：

```csharp
// ExecuteEvents.ExecuteHierarchy
GameObject current = target;
while (current != null)
{
    var result = Execute(current, eventData, handler);
    if (result != null) return result;   // 找到实现者就停
    current = current.transform.parent?.gameObject;  // 否则继续向上
}
```

### 11.4.3 一次完整点击的生命周期

```
鼠标按下
  │
  ├─ ① ProcessMouseEvent() → RaycastAll() → 找到命中 Graphic
  ├─ ② 比较与上一帧 Hover 对象是否不同
  │     不同 → PointerExit（旧对象） + PointerEnter（新对象）
  │
  ├─ ③ PointerDown
  │     ExecuteHierarchy() 查找 IPointerDownHandler → 执行 OnPointerDown()
  │     记录 pointerPress = 命中对象，eligibleForClick = true
  │
  └─ ── 可能经过多帧 ──
  │
鼠标抬起
  │
  ├─ ④ PointerUp
  │     Execute(pointerPress) → IPointerUpHandler.OnPointerUp()
  │
  └─ ⑤ Click 判定
        pointerPress == 当前抬起对象？ && eligibleForClick == true？
          是 → IPointerClickHandler.OnPointerClick()
          否 → 不触发 Click（拖拽了或者手指滑走了）
```

**Click 不是 MouseDown 触发的，而是 MouseUp 后判定的。** 三个条件缺一不可：按下 + 抬起 + 按下和抬起是同一对象。

### 11.4.4 Button.OnClick 的触发链路

```
Button.OnPointerClick(PointerEventData eventData)
{
    if (eventData.button != PointerEventData.InputButton.Left) return;
    Press();
}
  → Press() → m_OnClick.Invoke()  // m_OnClick 即 UnityEvent
```

Inspector 中拖进去的 `button.onClick.AddListener()` 回调，最终就是 `UnityEvent.Invoke()`。

### 11.4.5 拖拽如何取消 Click

鼠标按下后，如果移动超过 `eventSystem.pixelDragThreshold` 像素（默认 5px），系统会认为进入拖拽状态：

```
PointerDown → 记录按下位置
  → 鼠标移动 → 判断位移 > pixelDragThreshold？
    是 → dragging = true, eligibleForClick = false, BeginDrag → Drag...
    否 → 仍保持 Click 资格
```

这就是 ScrollRect 中 Button 偶尔点不动的根本原因——手指/鼠标在按下和抬起之间有一点点微动，UI 判定为拖拽了。

### 11.4.6 双击检测

```csharp
// 源码逻辑
if ((Time.unscaledTime - clickTime) < 0.3f)
    clickCount++;       // 300ms 内连续点击，次数累加
else
    clickCount = 1;     // 超过 300ms，重新计数
clickTime = Time.unscaledTime;
```

开发中使用：`if (eventData.clickCount == 2) { /* 双击逻辑 */ }`

### 11.4.7 Button 的高亮/按压状态从哪来

Button 继承自 `Selectable`。`Selectable.DoStateTransition()` 根据当前状态主动控制外观：

| 状态 | 触发条件 |
|------|---------|
| Normal | 无任何交互 |
| Highlighted | 鼠标悬停（PointerEnter） |
| Pressed | 鼠标按下（PointerDown） |
| Selected | 被选中（键盘导航/代码设置） |
| Disabled | interactable = false |

外观切换方式有三种：Color Tint（修改颜色）、Sprite Swap（切换精灵）、Animation（播放 Animator）。这不是 Animator 自动完成的——是 Selectable 代码驱动的。

---

## 11.5 帧同步与性能优化

### 11.5.1 EventSystem 的帧驱动本质

EventSystem 继承 `UIBehaviour → MonoBehaviour`，核心逻辑在 `Update()` 中。每帧执行一次：输入检测 → Raycast → 事件派发。

**输入频率 = 当前帧率。** 30FPS = 每秒 30 次输入更新。低帧率下 Button 点击感觉变慢就是这个原因。

### 11.5.2 timeScale 对 UI 输入的影响

> ⚠ **勘误**：原文称 `timeScale = 0` 时 UI 输入依然有效，理由是"EventSystem 默认属于 unscaled update"。**这是错的。**

`EventSystem.Update()` 是标准 MonoBehaviour 的 Update，**受 timeScale 控制**。`timeScale = 0` 时 EventSystem 停止更新，所有 UI 输入完全失效。需要暂停游戏又保持 UI 可操作的，要额外处理（把 UI 放到不受 timeScale 影响的机制中）。

### 11.5.3 PointerEventData 的跨帧状态

点击不是单帧完成的——`pointerPress`、`eligibleForClick`、`dragging`、`clickTime`、`clickCount` 这些状态跨帧持续存在。PointerEventData 本质上是一个跨帧的状态机载体。

### 11.5.4 UI 输入与 FixedUpdate 无关

UI 输入始终在 `Update` 中，与 `FixedUpdate` 无关。修改 `Time.fixedDeltaTime` 不影响 UI 点击频率。

### 11.5.5 性能优化要点

UI 输入的性能开销 = **Raycast 遍历成本 + Canvas 重建成本**。

**关键优化项：**

1. **关闭无意义元素的 raycastTarget**：收益最大的单项操作。装饰性 Text 和 Image 是重点目标。
2. **减少单个 Canvas 的 Graphic 数量**：Raycast 遍历的是整个 Canvas 的 Graphic 列表。
3. **拆分动态 Canvas**：频繁变化的 UI 放独立 Canvas，避免拖累静态 UI 的重建。
4. **避免深层 Layout 嵌套 + 动态 Text 更新**：它们触发 Layout Rebuild，间接导致 Canvas Rebuild。

### 11.5.6 UI 输入与网络

UGUI 输入是纯本地的。帧同步游戏中（MOBA/RTS），UI 点击通常不走同步通道——只通过 `Button.onClick → 发送 RPC → 服务器逻辑` 手动同步。

---

## 11.6 新输入系统兼容

### 11.6.1 架构变化

```
旧：Input Manager → StandaloneInputModule → PointerEventData → EventSystem
新：Input Action / Device → InputSystemUIInputModule → PointerEventData → EventSystem
```

核心变化只有一处：**InputModule 的替换**。EventSystem 和 Raycast 体系完全不变。

### 11.6.2 InputSystemUIInputModule

```csharp
public class InputSystemUIInputModule : BaseInputModule
{
    public override void Process()
    {
        // 读取 InputAction → 构建 PointerEventData → Raycast → 事件分发
    }
}
```

它将 `InputActionAsset` 中的 Action 绑定到 UI 事件：

| Action | 对应数据 |
|--------|---------|
| Point | `pointerData.position` |
| LeftClick | `pointerData.button = Left`，触发 Down/Up/Click |
| RightClick / MiddleClick | 对应按钮的 Down/Up/Click |
| ScrollWheel | `pointerData.scrollDelta` |
| Navigate | 键盘/手柄方向 → MoveEvent → 驱动导航选择 |

### 11.6.3 分层设计

```
Device Layer（鼠标/键盘/手柄/触摸）
  → Action Layer（Point / Click / Scroll / Navigate）
    → UI Layer（PointerEventData → ExecuteEvents）
```

这种三层抽象让 UI 代码与具体设备解耦——同一个 `LeftClick` Action 可以绑定鼠标左键、触摸、手柄 A 键等。换平台不需要改 UI 逻辑。

---

## 本章总结

EventSystem 是 UGUI 的"输入中间层"——将多设备输入统一为 PointerEventData，通过 Raycaster 体系完成命中检测，再用 ExecuteEvents 分发给实现了特定接口的 UI 组件。

至此，UGUI 三大核心体系完整建立：

| 体系 | 核心组件 | 一句话 |
|------|---------|--------|
| **空间** | RectTransform + Layout | 定义 UI 的位置、尺寸和排列 |
| **渲染** | Graphic + Canvas + CanvasRenderer | 生成顶点 → 合并 Batch → 提交 DrawCall |
| **交互** | EventSystem + InputModule + Raycaster + ExecuteEvents | 采集输入 → 找到目标 → 分发事件 → 触发回调 |

三者形成闭环：交互触发回调 → 回调修改 Layout/RectTransform → Graphic 重建顶点 → Canvas 重建 Batch → 渲染到屏幕 → 下一帧输入再次读取新状态。

---

## 勘误汇总

| # | 严重程度 | 原文章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 严重 | 11.8.4 | `timeScale = 0` 时 UI 输入依然有效，"UGUI 输入系统默认属于 unscaled update" | EventSystem.Update() 是标准 MonoBehaviour Update，受 timeScale 控制 |
| 2 | 🟡 中等 | 11.6.10 | "Graphic.Raycast 内部还会检测 Mask、RectMask2D、CanvasGroup" | Mask/RectMask2D 裁剪判断在 GraphicRaycaster.Raycast() 主流程中，不在 Graphic.Raycast() 内部 |
| 3 | 🟢 轻微 | 11.3.11 / 11.2.9 | 只给出 `IsPointerOverGameObject()` 无参用法 | 无参版本仅检测 PointerId=-1（鼠标左键），移动端/多键场景需传 pointerId |
| 4 | 🟢 轻微 | 11.7.1 / 11.7.13 | 旧系统"轮询" vs 新系统"事件驱动"的二分对立 | 旧系统同样只在 Process() 中按需读取；差异在抽象层级，非运行模式二分 |
| 5 | 🟢 轻微 | 11.2.3 | 继承结构写为 PhysicsRaycaster 和 Physics2DRaycaster 平级 | Physics2DRaycaster 继承自 PhysicsRaycaster，非直接继承 BaseRaycaster |
