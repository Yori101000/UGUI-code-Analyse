# 第11章 EventSystem 事件系统（修正版）

> 本文是对原文的精炼总结，并对照 UGUI 源码修正了原文中的 5 处错误。

---

## 概述

UGUI 不仅是一套绘制系统，更是一套完整的交互系统。按钮点击、拖拽、滚动、输入框等交互行为，背后都依赖 **EventSystem** 完成输入采集、射线检测、事件分发和回调执行。

整个事件系统的链路为：

```
输入设备（鼠标/触摸/键盘/手柄）
  → InputModule（输入采集，生成 PointerEventData）
    → EventSystem（调度 Raycast + 事件分发）
      → Raycaster（命中检测，找到目标对象）
        → ExecuteEvents（接口回调执行）
          → UI 组件响应（Button.onClick 等）
```

核心组件职责：

| 组件 | 职责 |
|------|------|
| **EventSystem** | 全局调度中心：管理 InputModule、执行 Raycast、分发事件 |
| **InputModule** | 输入采集：将设备输入转为标准化的 PointerEventData |
| **GraphicRaycaster** | UI 命中检测：判断屏幕坐标命中了哪个 Graphic |
| **GraphicRegistry** | 维护 Canvas → Graphic 列表的全局缓存，避免 Raycast 遍历全场景 |
| **RaycasterManager** | 管理所有 Raycaster 的全局注册表 |
| **ExecuteEvents** | 静态事件执行器，通过接口调用目标对象的回调 |

---

## 11.1 输入与事件分发

### 11.1.1 EventSystem 的核心职责

EventSystem 本身不读取鼠标键盘——它只负责**调度**。真正的输入采集由 InputModule 完成。

EventSystem.Update() 每帧执行：

```csharp
protected virtual void Update()
{
    if (current != this) return;
    TickModules();
    
    // 选择当前激活的 InputModule
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
    
    // 执行当前 InputModule 的 Process
    if (!changedModule && m_CurrentInputModule != null)
        m_CurrentInputModule.Process();
}
```

流程：更新所有 InputModule → 选一个激活的 → 调用它的 `Process()`。

### 11.1.2 InputModule 继承结构

```
BaseInputModule
  └── PointerInputModule
       ├── StandaloneInputModule    （旧输入系统）
       ├── TouchInputModule
       └── InputSystemUIInputModule（新输入系统）
```

所有 InputModule 都重写 `Process()`——不同输入系统的本质区别，就是 `Process` 的实现不同。

### 11.1.3 StandaloneInputModule 的 Process 流程

```csharp
public override void Process()
{
    bool usedEvent = SendUpdateEventToSelectedObject();  // 更新选中对象（如 InputField）

    if (!ProcessTouchEvents() && input.mousePresent)     // 触摸优先，否则处理鼠标
        ProcessMouseEvent();

    if (eventSystem.sendNavigationEvents)                // 键盘/手柄导航
    {
        if (!usedEvent)
            usedEvent |= SendMoveEventToSelectedObject();
        if (!usedEvent)
            SendSubmitEventToSelectedObject();
    }
}
```

三部分：更新选中对象 → 处理鼠标/触摸 → 处理导航事件。

### 11.1.4 PointerEventData

所有点击、拖拽、移动事件最终都转换为 **PointerEventData**——一次指针输入的完整上下文：

| 字段 | 含义 |
|------|------|
| `position` | 当前鼠标/触摸位置 |
| `delta` | 移动增量 |
| `pointerEnter` | 当前悬停对象 |
| `pointerPress` | 当前按下对象 |
| `lastPress` | 上一帧的按下对象 |
| `pointerCurrentRaycast` | 当前射线命中结果 |
| `eligibleForClick` | 是否有资格触发 Click |
| `clickCount` / `clickTime` | 双击判定用 |
| `dragging` | 是否正在拖拽 |

### 11.1.5 ExecuteEvents 事件执行器

所有事件派发都通过 `ExecuteEvents` 这个静态类完成：

```csharp
ExecuteEvents.Execute(target, eventData, ExecuteEvents.pointerClickHandler);
// → 调用 target 上的 IPointerClickHandler.OnPointerClick()
```

它内部维护了大量事件函数：`pointerEnterHandler`、`pointerDownHandler`、`pointerUpHandler`、`pointerClickHandler`、`beginDragHandler`、`dragHandler`、`endDragHandler` 等。

**UGUI 的事件系统本质上就是接口回调系统**——GameObject 实现了对应接口就能收到事件。

### 11.1.6 事件冒泡：ExecuteHierarchy

点击 Text 子节点时，Button 父节点仍然能收到事件，因为 UGUI 使用 `ExecuteEvents.ExecuteHierarchy()`：

```
当前节点 → 没有对应接口？ → 父节点 → 继续向上 → 直到找到实现了接口的对象
```

---

## 11.2 Raycast 机制

### 11.2.1 UGUI Raycast 不是物理射线

UGUI 使用的是 **Graphic Raycast**（屏幕空间矩形检测），不是 `Physics.Raycast`。本质是：**屏幕坐标是否在 RectTransform 的矩形区域内部**。

### 11.2.2 EventSystem.RaycastAll 流程

```csharp
public void RaycastAll(PointerEventData eventData, List<RaycastResult> raycastResults)
{
    raycastResults.Clear();
    var modules = RaycasterManager.GetRaycasters();   // 获取所有 Raycaster

    for (int i = 0; i < modules.Count; ++i)
    {
        var module = modules[i];
        if (module == null || !module.IsActive()) continue;
        module.Raycast(eventData, raycastResults);    // 每个 Raycaster 各自检测
    }

    raycastResults.Sort(s_RaycastComparer);            // 统一排序
}
```

EventSystem 不自己做检测——它遍历所有 Raycaster，每个 Raycaster 各自检测，最后统一排序。

### 11.2.3 三种 Raycaster 的实现机制

虽然都实现统一的 `Raycast()` 抽象方法，但三种 Raycaster 的底层检测方式完全不同：

| | GraphicRaycaster | PhysicsRaycaster | Physics2DRaycaster |
|------|------|------|------|
| 检测对象 | Graphic（RectTransform） | Collider（3D） | Collider2D |
| 检测方式 | 屏幕坐标 + 矩形包含判断 | 从屏幕点发射 3D 射线，`Physics.RaycastAll` | 从屏幕点发射 2D 射线，`Physics2D.GetRayIntersectionAll` |
| 需要 Camera | 仅 Camera/World 模式需要 | 必须（用 `camera.ScreenPointToRay`） | 必须 |
| 核心方法 | `RectangleContainsScreenPoint` | `Physics.RaycastAll` | `Physics2D.GetRayIntersectionAll` |
| 排序依据 | Graphic.depth | 射线距离（distance） | 射线距离 |

**PhysicsRaycaster 的大致逻辑：**

```csharp
// 从屏幕坐标发射真正的 3D 射线
Ray ray = eventCamera.ScreenPointToRay(eventData.position);
var hits = Physics.RaycastAll(ray, maxDistance, layerMask);

foreach (var hit in hits)
{
    resultAppendList.Add(new RaycastResult {
        gameObject = hit.collider.gameObject,
        module = this,
        distance = hit.distance,
        // ...
    });
}
```

只有 GraphicRaycaster 是"屏幕矩形判断"——PhysicsRaycaster 和 Physics2DRaycaster 都在发射真正的物理射线。EventSystem 通过 `RaycasterManager` 统一调度，三者的结果是合并到一起再统一排序的。

**继承结构：**

```
BaseRaycaster
├── GraphicRaycaster       （UI 命中检测）
└── PhysicsRaycaster       （3D 碰撞体）
     └── Physics2DRaycaster（2D 碰撞体，继承自 PhysicsRaycaster）
```

### 11.2.4 GraphicRaycaster 检测流程——与 Graphic.Raycast() 的关系

这里容易混淆，因为有两个同名方法：

- **`GraphicRaycaster.Raycast()`**：Raycaster 层面的检测入口，遍历 Graphic 列表 → 矩形判断 → 排序
- **`Graphic.Raycast()`**：单个 Graphic 身上的虚方法，用于附加过滤（Alpha Hit Test 等）

前者调用后者，每个 Graphic 经过两步筛选才能命中：

```
步骤 1：矩形检测（GraphicRaycaster 负责）
  RectTransformUtility.RectangleContainsScreenPoint(rect, pos, camera)
  → 屏幕坐标是否在 RectTransform 矩形区域内？

步骤 2：Graphic 自身额外过滤（Graphic 虚方法负责）
  graphic.Raycast(sp, camera)
  → 矩形命中了，但 Graphic 自己说"不算"就不算
```

`Graphic.Raycast()` 的默认实现直接 return true，只有特定子类重写：

```csharp
// Graphic 基类默认实现
public virtual bool Raycast(Vector2 sp, Camera eventCamera)
{
    return true;  // 矩形命中就算
}

// Image 重写（开启 AlphaHitTestMinimumThreshold 时）
public override bool Raycast(Vector2 sp, Camera eventCamera)
{
    // 检测点击位置像素的 Alpha 是否大于阈值
    // 透明像素返回 false，不响应点击
}
```

**完整调用层级关系：**

```
EventSystem.RaycastAll()
  └─ RaycasterManager.GetRaycasters()          ← 全局统一调度
       ├─ GraphicRaycaster.Raycast()            ← BaseRaycaster 接口实现
       │    ├─ RectangleContainsScreenPoint()   ← 屏幕矩形检测
       │    └─ graphic.Raycast()                ← Graphic 虚方法（附加过滤）
       ├─ PhysicsRaycaster.Raycast()            ← BaseRaycaster 接口实现
       │    └─ Physics.RaycastAll()             ← 发射真正的 3D 射线
       └─ Physics2DRaycaster.Raycast()          ← BaseRaycaster 接口实现
            └─ Physics2D.GetRayIntersectionAll()← 发射真正的 2D 射线
```

### 11.2.5 GraphicRaycaster 过滤流程

具体的过滤与命中检测步骤：

```
GraphicRegistry.GetGraphicsForCanvas(canvas)   ← 获取当前 Canvas 所有 Graphic
  → 过滤：depth == -1（未参与渲染）→ 跳过
  → 过滤：raycastTarget == false → 跳过
  → 过滤：canvasRenderer.cull（已被裁剪）→ 跳过
  → RectTransformUtility.RectangleContainsScreenPoint()  ← 屏幕坐标是否在矩形内
  → graphic.Raycast(sp, eventCamera)          ← Graphic 自身额外过滤（如 Alpha Hit Test）
  → 按 depth 降序排序（视觉上层的优先）
  → 生成 RaycastResult
```

### 11.2.5 Graphic.Raycast 的作用

`Graphic.Raycast()` 是 Graphic 类的虚方法，默认返回 `true`。Image 重写它用于 **Alpha Hit Test**（只有当点击位置的像素 Alpha 大于阈值时才命中，透明区域不响应）。

> ⚠ **勘误 #1**：原文称"Graphic.Raycast 内部还会检测 Mask、RectMask2D、CanvasGroup"。实际上 Mask/RectMask2D 的裁剪判断发生在 `GraphicRaycaster.Raycast()` 主流程中（`canvasRenderer.cull` 字段即为 RectMask2D 设置），**不在 Graphic.Raycast() 内部**。CanvasGroup 的 `blocksRaycasts` 属性也不在 Graphic.Raycast 中检查。

### 11.2.6 RaycastTarget 优化

### 11.2.7 RaycastTarget 属性详解

`raycastTarget` 是 `Graphic` 类上的一个 `[SerializeField]` 属性，默认值为 `true`：

```csharp
// Graphic.cs
[SerializeField]
private bool m_RaycastTarget = true;

public bool raycastTarget
{
    get => m_RaycastTarget;
    set
    {
        m_RaycastTarget = value;
        SetVerticesDirty();  // 修改后会触发重建
    }
}
```

**所有 Graphic 子类都有这个属性**：Image、Text（旧版）、RawImage、MaskableGraphic 等。TMP_Text 虽然没有继承 Graphic，但它有独立的 `raycastTarget`。

**GraphicRaycaster 中的过滤代码**（前面流程中已列出）：

```csharp
if (!graphic.raycastTarget)
    continue;  // 直接跳过，不参与命中检测
```

**三种操作方式：**

1. **Inspector 面板**：选中 Image/Text/RawImage 组件，在 Inspector 最底部有一个 `Raycast Target` 复选框，取消勾选即可。这是最常用的方式。

2. **代码手动设置**：
```csharp
GetComponent<Image>().raycastTarget = false;
```

3. **运行时批量清理**：复杂 UI 中，一个 Prefab 里可能有几十个装饰性 Text。可以通过脚本在 `Awake()` 或编辑器中统一关闭：
```csharp
// 递归关闭某个节点下所有 Graphic 的 raycastTarget
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
| 标签/说明文字 Text | 需要拖拽的 Slider |
| 仅用于布局的占位 Graphic | 需要悬停检测的 Hover 区域 |
| ScrollView 内容区的缩略图 | InputField 本身 |

**为什么 Text 是重灾区**：聊天气泡、任务列表、背包物品名——这些 Text 如果 Canvas 下有上百个且全部开着 `raycastTarget`，鼠标每动一像素就要遍历这上百个做矩形检测。而且 Text 通常覆盖在 Button 上，开着还会阻挡 Button 的点击事件穿透到父级。

> 注意：关闭 `raycastTarget` 只影响事件检测，不影响渲染——UI 照样显示，只是"不参与点击"。

### 11.2.7 排序规则

RaycastResult 排序综合考虑：Sorting Layer → Sorting Order → Graphic Depth → 与 Camera 的距离。视觉上最前面的对象优先响应事件。**Raycast 排序与渲染顺序一致。**

### 11.2.8 Blocking Objects

GraphicRaycaster 支持 Blocking Objects 模式：先执行 `Physics.Raycast`，如果前方有 Collider，UI 不再响应。用于 World Space UI 被场景物体遮挡的场景。

### 11.2.9 UI 阻挡 3D 点击

UI 和 Physics 同时命中时，用 `EventSystem.current.IsPointerOverGameObject()` 判断。注意默认无参版本只检测 `PointerId = -1`（鼠标左键），移动端或右键检测需要传入 `pointerId`。

> ⚠ **勘误 #2**：原文多次使用 `IsPointerOverGameObject()` 无参版本，但未提及它在触摸屏和多按钮场景下的局限性。

---

## 11.3 UI 点击流程

### 11.3.1 完整流程

一次 Button 点击从 PointerDown 开始，但 **Click 在 PointerUp 时才判定**：

```
鼠标按下 → ProcessMouseEvent() → RaycastAll → 找到命中 Graphic
  → PointerEnter（若与上一帧不同）
  → PointerDown → ExecuteHierarchy 查找 IPointerDownHandler → 记录 pointerPress
  --- 可能经过多帧 ---
  → 鼠标抬起 → PointerUp → 执行 IPointerUpHandler
  → Click 判定：pointerPress == pointerUp 的对象？ 是 → 触发 PointerClick
  → Button.OnPointerClick() → Press() → m_OnClick.Invoke()
```

### 11.3.2 为什么"按下再抬起才叫点击"

Click 判定条件三点必须同时满足：MouseDown + MouseUp + 按下和抬起是同一对象。拖拽过程中（移动超过像素阈值后 `dragging = true` 且 `eligibleForClick = false`）不会触发 Click——这就是 ScrollRect 中 Button 偶尔点不动的根本原因。

### 11.3.3 接口调用顺序

正常点击：**PointerEnter → PointerDown → InitializePotentialDrag → PointerUp → PointerClick**

拖拽时：**PointerDown → BeginDrag → Drag... → EndDrag**（Click 不触发）

### 11.3.4 Button 状态切换

Button 继承自 `Selectable`，不是依赖 Animator 自动完成的。`Selectable.DoStateTransition()` 根据 Normal / Highlighted / Pressed / Disabled / Selected 状态主动切换颜色、Sprite 或 Trigger Animation。

### 11.3.5 双击检测

`PointerEventData.clickCount` 和 `clickTime`（使用 `Time.unscaledTime`）实现双击判定：两次点击间隔 < 0.3 秒则 `clickCount++`。开发中判断 `eventData.clickCount == 2` 即可实现双击逻辑。

---

## 11.4 GraphicRegistry 机制

### 11.4.1 为什么需要

如果每帧 Raycast 都要遍历整个场景找 Graphic，大场景下 CPU 开销不可接受。GraphicRegistry 提前缓存了 **Canvas → Graphic 集合** 的映射。

### 11.4.2 核心数据结构

```csharp
private readonly Dictionary<Canvas, IndexedSet<Graphic>> m_Graphics;
```

每个 Canvas 对应一个 `IndexedSet<Graphic>`——这是 UGUI 自定义容器，内部同时维护 `List<T>`（保持遍历顺序）和 `Dictionary<T, int>`（O(1) 去重）。

### 11.4.3 注册与注销

**注册**（`Graphic.OnEnable`）：

```csharp
protected override void OnEnable()
{
    base.OnEnable();
    CacheCanvas();
    GraphicRegistry.RegisterGraphicForCanvas(canvas, this);  // 注册
    SetAllDirty();
}
```

**注销**（`Graphic.OnDisable`）：

```csharp
protected override void OnDisable()
{
    GraphicRegistry.UnregisterGraphicForCanvas(canvas, this);  // 注销
    CanvasUpdateRegistry.UnRegisterCanvasElementForRebuild(this);
    canvasRenderer.Clear();
    base.OnDisable();
}
```

整个过程自动完成，开发者无需手动管理。动态创建/销毁的 UI 也会自动注册/注销。

### 11.4.4 与 Raycast 的关系

`GraphicRaycaster.Raycast()` 通过 `GraphicRegistry.GetGraphicsForCanvas(canvas)` 获取 Graphic 列表——`Dictionary` 查询，**时间复杂度 O(1)**。Registry 只是数据源，真正的过滤逻辑仍在 `GraphicRaycaster.Raycast()` 内部。

---

## 11.5 RaycasterManager

### 11.5.1 核心职责

RaycasterManager 是**所有 Raycaster 的全局注册表**，维护一个 `List<BaseRaycaster>`。EventSystem 通过它获取当前场景所有 Raycaster，逐个调用 `Raycast()`。

### 11.5.2 注册与注销

```
BaseRaycaster.OnEnable  →  RaycasterManager.AddRaycaster(this)
BaseRaycaster.OnDisable →  RaycasterManager.RemoveRaycasters(this)
```

### 11.5.3 与 GraphicRegistry 的层级关系

```
EventSystem
  → RaycasterManager       ← 管理所有 Raycaster（顶层）
    → GraphicRaycaster     ← 单个 Raycaster 实例
      → GraphicRegistry    ← 该 Raycaster 对应的 Canvas 下所有 Graphic（底层）
```

| | GraphicRegistry | RaycasterManager |
|------|---------|------|
| 管理对象 | Canvas → Graphic 集合 | 所有 BaseRaycaster |
| 服务对象 | GraphicRaycaster | EventSystem |
| 数据结构 | `Dictionary<Canvas, IndexedSet<Graphic>>` | `List<BaseRaycaster>` |

---

## 11.6 GraphicRaycaster 源码深析

### 11.6.1 主流程

```csharp
public override void Raycast(PointerEventData eventData, List<RaycastResult> resultAppendList)
{
    var canvasGraphics = GraphicRegistry.GetGraphicsForCanvas(canvas);
    
    for (int i = 0; i < canvasGraphics.Count; ++i)
    {
        Graphic graphic = canvasGraphics[i];
        
        // 过滤 1：未参与渲染
        if (graphic.depth == -1) continue;
        // 过滤 2：不接收射线
        if (!graphic.raycastTarget) continue;
        // 过滤 3：已被裁剪（RectMask2D / CanvasGroup cull）
        if (graphic.canvasRenderer.cull) continue;
        
        // 矩形命中检测
        if (!RectTransformUtility.RectangleContainsScreenPoint(
            graphic.rectTransform, eventData.position, eventCamera))
            continue;
        
        // Graphic 自身过滤（Alpha Hit Test 等）
        if (!graphic.Raycast(eventData.position, eventCamera))
            continue;
        
        // 通过所有过滤，加入结果集
        s_SortedGraphics.Add(graphic);
    }
    
    // 按 depth 降序排序
    s_SortedGraphics.Sort((g1, g2) => g2.depth.CompareTo(g1.depth));
    
    // 生成 RaycastResult
    for (int i = 0; i < s_SortedGraphics.Count; i++)
    {
        var result = new RaycastResult { /* ... */ };
        resultAppendList.Add(result);
    }
}
```

`depth` 来自 `CanvasRenderer.absoluteDepth`，与渲染顺序一致。

### 11.6.2 性能考虑

每次 Raycast 都遍历当前 Canvas 下所有 Graphic。因此优化重点：关闭无意义元素的 `raycastTarget`（尤其是纯装饰性 Text）、减少单个 Canvas 内 Graphic 数量。

---

## 11.7 新输入系统支持

### 11.7.1 架构变化

```
旧系统：Input Manager → StandaloneInputModule → PointerEventData → EventSystem
新系统：Input Action / Device → InputSystemUIInputModule → PointerEventData → EventSystem
```

核心变化：`StandaloneInputModule` 被 `InputSystemUIInputModule` 替代。但 **EventSystem 和 Raycast 体系不变**——变的是输入来源，不是输入处理机制。

> ⚠ **勘误 #3**：原文将旧系统描述为"轮询"、新系统描述为"事件驱动"，做二分对立。实际上旧系统同样只在 `Process()` 中按需读取输入状态，不是每帧遍历所有设备。新系统确实使用 Action 抽象，但连续值（如指针位置）内部仍有状态读取。两者真正的差异在于**抽象层级和可配置性**，而非"轮询 vs 事件"的二分。

### 11.7.2 InputSystemUIInputModule 的 Process

```
读取 InputAction（Point/LeftClick/ScrollWheel 等）
  → 构建 PointerEventData
  → EventSystem.RaycastAll()
  → 更新 Pointer 状态
  → 触发 UI 事件
```

所有输入被抽象为 InputAction，UGUI 不再关心具体是什么设备——同一个 `LeftClick` Action 可以绑定鼠标左键、触摸、手柄 A 键等。

### 11.7.3 分层设计

```
Device Layer（鼠标/键盘/手柄）
  → Action Layer（Point/Click/Scroll/Navigate）
    → UI Layer（PointerEventData → ExecuteEvents）
```

这种分层让 UI 代码与具体设备解耦，跨平台时不需要改动 UI 逻辑。

---

## 11.8 UI 输入与帧同步

### 11.8.1 帧驱动本质

EventSystem 继承自 `UIBehaviour → MonoBehaviour`，其核心逻辑在 `Update()` 中。每帧执行一次：输入检测 → Raycast → 事件派发。输入频率 = 当前帧率——30FPS 时每秒只能处理 30 次输入更新。

### 11.8.2 timeScale 影响（已修正）

> ⚠ **勘误 #4（严重）**：原文称"即使 `Time.timeScale = 0`，UGUI 输入依然有效。原因在于 EventSystem.Update 并不会停止。UGUI 输入系统默认属于 unscaled update。"

**这是错误的。** EventSystem.Update() 是标准 MonoBehaviour 的 Update，**受 timeScale 控制**。`timeScale = 0` 时 EventSystem 不再更新，UI 输入会完全停止响应。如果确实需要在暂停状态下保持 UI 可用，需要额外处理（如将 EventSystem 逻辑移到不受 timeScale 影响的生命周期，或使用独立的不暂停 Canvas）。

### 11.8.3 PointerEventData 的跨帧状态

点击不是单帧完成的，中间状态保存在 PointerEventData：`pointerPress`、`eligibleForClick`、`dragging`、`clickTime`、`clickCount` 等字段跨帧持续存在。PointerEventData 本质上是跨帧输入状态容器。

### 11.8.4 与 FixedUpdate 无关

UI 输入始终在 Update 中处理，与 FixedUpdate 无关。修改 `Time.fixedDeltaTime` 不影响 UI 点击频率。

### 11.8.5 UI 输入与网络

UGUI 输入是纯本地的，不会自动网络同步。帧同步游戏中，UI 点击通常不走同步通道——只通过 `Button.onClick → RPC → 服务器逻辑` 手动同步。

---

## 本章总结

EventSystem 是 UGUI 的"输入中间层"——将输入采集、空间检测与事件分发整合为统一流程。

至此，UGUI 三大核心体系已经完整建立：

| 体系 | 核心组件 | 职责 |
|------|---------|------|
| **空间系统** | RectTransform + Layout | 定义 UI 的位置、尺寸、排列规则 |
| **几何系统** | Graphic + Canvas + CanvasRenderer | 生成顶点数据、构建 Batch、提交渲染 |
| **交互系统** | EventSystem + InputModule + Raycaster + ExecuteEvents | 采集输入、命中检测、事件分发、回调执行 |

三者相互协作：Layout 驱动 RectTransform → Graphic 基于 RectTransform 生成顶点 → Canvas 合并 Batch → RenderPipeline 调度 → GPU 输出像素 → EventSystem 处理输入 → 回调修改 Layout/RectTransform → 触发新一轮 Rebuild → 闭环。

---

## 勘误汇总

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 严重 | 11.8.4 | `timeScale = 0` 时 UI 输入依然有效，"UGUI 输入系统默认属于 unscaled update" | EventSystem.Update() 是标准 MonoBehaviour Update，受 timeScale 控制。timeScale=0 时 UI 输入停止响应 |
| 2 | 🟡 中等 | 11.6.10 | "Graphic.Raycast 内部还会检测 Mask、RectMask2D、CanvasGroup" | Mask/RectMask2D 裁剪判断在 GraphicRaycaster.Raycast() 主流程中，不在 Graphic.Raycast() 内部 |
| 3 | 🟢 轻微 | 11.3.11 / 11.2.9 | 只给出 `IsPointerOverGameObject()` 无参用法 | 无参版本仅检测 PointerId=-1（鼠标左键），移动端/多键场景需传 pointerId 参数 |
| 4 | 🟢 轻微 | 11.7.1 / 11.7.13 | 旧系统"轮询" vs 新系统"事件驱动"的二分对立 | 旧系统同样只在 Process() 中按需读取；两者差异在抽象层级，非运行模式二分 |
| 5 | 🟢 轻微 | 11.2.3 | 继承结构写为 `BaseRaycaster → PhysicsRaycaster` 和 `BaseRaycaster → Physics2DRaycaster` 平级 | `Physics2DRaycaster` 继承自 `PhysicsRaycaster`，非直接继承 `BaseRaycaster` |
