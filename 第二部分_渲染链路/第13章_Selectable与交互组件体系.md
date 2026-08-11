# 第13章 Selectable 与交互组件体系

> UGUI 中的所有交互组件——Button、Toggle、Slider、Scrollbar、Dropdown、InputField——都继承自同一个基类 `Selectable`。理解 Selectable 的状态机机制、过渡方式和导航系统，是掌握交互组件体系的钥匙。

---

## 概述

UI 交互组件面临一个共同的问题：**在不同交互状态下呈现不同的视觉效果**。鼠标悬停时高亮、按下时变色、禁用时变灰——这些行为在所有交互组件中都是一致的。

Selectable 将这套"状态 → 视觉反馈"的逻辑统一封装，子类只需要关注各自特有的交互逻辑（Button 的点击事件、Toggle 的开关状态、Slider 的值变化），不需要重复实现状态管理。

```
UIBehaviour
  └── Selectable ← 所有交互组件的基类
       ├── Button        ← 点击事件（IPointerClickHandler）
       ├── Toggle        ← 开关状态（+ToggleGroup 互斥）
       ├── Slider        ← 连续值拖拽
       ├── Scrollbar     ← 滚动条（Slider 类似，专供 ScrollRect）
       ├── Dropdown      ← 下拉菜单（复合控件）
       └── InputField    ← 文本输入（复合控件）
```

---

## 13.1 状态机

### 13.1.1 SelectionState 枚举

Selectable 定义了四种状态：

| 状态 | 触发条件 |
|------|---------|
| **Normal** | 无任何交互 |
| **Highlighted** | 鼠标悬停，或通过键盘导航选中但未按下 |
| **Pressed** | 鼠标按下 |
| **Disabled** | `interactable = false`，或父级 CanvasGroup 被禁用 |

### 13.1.2 状态评估逻辑

Selectable 内部通过三个布尔变量追踪交互状态：

```csharp
bool isPointerInside;  // 鼠标是否在区域内
bool isPointerDown;    // 鼠标是否按下
bool hasSelection;     // 是否被 EventSystem 选中
```

状态的判定顺序（在 `UpdateSelectionState()` 中实现）：

```csharp
if (!IsInteractable())               → Disabled
else if (isPointerDown)              → Pressed
else if (hasSelection)               → Highlighted（选中状态复用 Highlighted 的值）
else if (isPointerInside)            → Highlighted
else                                 → Normal
```

注意：**Selected（选中）和 Highlighted（悬停）在视觉上使用相同的状态值**，两者没有视觉区别。

### 13.1.3 DoStateTransition——视觉过渡的执行

当状态发生变化时，`DoStateTransition(SelectionState state, bool instant)` 被调用：

```csharp
protected virtual void DoStateTransition(SelectionState state, bool instant) {
    // 根据状态查表获取颜色/Sprite/动画触发器
    switch (state) {
        case Normal:      颜色=normalColor,        Sprite=null,        触发器="Normal";       break;
        case Highlighted: 颜色=highlightedColor,   Sprite=highlighted, 触发器="Highlighted"; break;
        case Pressed:     颜色=pressedColor,        Sprite=pressed,     触发器="Pressed";     break;
        case Disabled:    颜色=disabledColor,       Sprite=disabled,    触发器="Disabled";    break;
    }

    // 根据 Transition 类型执行视觉变化
    switch (m_Transition) {
        case Transition.ColorTint:  StartColorTween(color, instant);    break;
        case Transition.SpriteSwap: DoSpriteSwap(sprite);              break;
        case Transition.Animation:  TriggerAnimation(triggerName);     break;
    }
}
```

SubClass 可以重写此方法，在保持基类状态管理的基础上叠加自定义效果。

---

## 13.2 三种过渡方式

### ColorTint（颜色渐变）

通过修改 `m_TargetGraphic`（通常是 Image 组件）的颜色来表现状态变化：

```csharp
private void StartColorTween(Color targetColor, bool instant) {
    m_TargetGraphic.CrossFadeColor(
        targetColor,
        instant ? 0 : m_Colors.fadeDuration,
        true,   // ignoreTimeScale——不受 timeScale 影响
        true    // useAlpha
    );
}
```

`CrossFadeColor` 使用协程做平滑过渡，且**不受 timeScale 影响**——即使游戏暂停，UI 颜色过渡仍然正常完成。

### SpriteSwap（精灵切换）

通过替换 `Image.overrideSprite` 来切换不同状态的图片：

```csharp
private void DoSpriteSwap(Sprite newSprite) {
    var image = m_TargetGraphic as Image;
    image.overrideSprite = newSprite;  // 不影响原始 sprite 引用
}
```

`overrideSprite` 在设置时覆盖渲染用的精灵，但保留 `sprite` 字段不变——切换回 null 时自动恢复原始精灵。

### Animation（动画）

通过 Animator 的 Trigger 控制状态切换：

```csharp
private void TriggerAnimation(string triggerName) {
    // 先重置所有触发器，避免状态叠加
    m_Animator.ResetTrigger("Normal");
    m_Animator.ResetTrigger("Highlighted");
    m_Animator.ResetTrigger("Pressed");
    m_Animator.ResetTrigger("Disabled");
    m_Animator.SetTrigger(triggerName);
}
```

重置后再触发是避免多个状态同时生效的关键。

---

## 13.3 导航系统（Navigation）

Selectable 实现了 `IMoveHandler`，通过方向键/手柄摇杆在 UI 元素之间移动焦点。

### 导航模式

| Mode | 行为 |
|------|------|
| **None** | 不可用方向键导航 |
| **Horizontal** | 仅左右自动导航 |
| **Vertical** | 仅上下自动导航 |
| **Automatic** | 双轴自动搜索最近的 Selectable |
| **Explicit** | 手动指定上/下/左/右各连到谁 |

### 自动搜索算法

`FindSelectable(Vector3 dir)` 在 Automatic 模式下被调用，核心逻辑：

```csharp
public virtual Selectable FindSelectable(Vector3 dir) {
    dir = transform.rotation * dir;  // 考虑旋转
    Vector3 localPos = transform.position;
    Selectable bestPick = null;
    float bestScore = float.MinValue;

    for (int i = 0; i < s_List.Count; ++i) {  // 遍历场景中所有 Selectable
        Selectable sel = s_List[i];
        if (sel == this || !sel.IsInteractable() || sel.navigation.mode == Mode.None)
            continue;

        Vector3 toTarget = sel.transform.position - localPos;
        float dot = Vector3.Dot(dir, toTarget.normalized);
        if (dot <= 0) continue;  // 不在目标方向

        // 计分：方向对齐度 / 距离——优先选"正方向且最近"的
        float score = dot / toTarget.sqrMagnitude;
        if (score > bestScore) { bestScore = score; bestPick = sel; }
    }
    return bestPick;
}
```

**行为本质**：从当前控件中心向外膨胀，直到接触到最近的符合条件的 Selectable。计分公式兼顾了方向（dot product）和距离。

---

## 13.4 静态列表与生命周期

### 全局 Selectable 列表

场景中所有激活的 Selectable 被注册到静态列表 `s_List`：

```csharp
private static List<Selectable> s_List = new List<Selectable>();

protected override void OnEnable() {
    s_List.Add(this);
    InternalEvaluateAndTransitionToSelectionState(true);  // 初始化视觉
}

protected override void OnDisable() {
    s_List.Remove(this);
    InstantClearState();  // 重置为 Normal 视觉状态
}
```

`InstantClearState()` 将过渡状态重置为白色（ColorTint）、无 overrideSprite（SpriteSwap）、Normal 动画（Animation）。

### IsInteractable——综合判断

```csharp
public bool IsInteractable() {
    return m_GroupsAllowInteraction && m_Interactable;
}
```

`m_GroupsAllowInteraction` 遍历父级所有 CanvasGroup，只有当所有 CanvasGroup 的 `interactable` 都为 true 时才可交互。

一个在不可交互的 CanvasGroup 下的 Button，即使自身的 `interactable = true`，也被视为 Disabled。

---

## 13.5 子类扩展

### Button

最简单的子类，只添加了点击事件：

```csharp
public class Button : Selectable, IPointerClickHandler, ISubmitHandler {
    public ButtonClickedEvent onClick;

    public void OnPointerClick(PointerEventData eventData) {
        if (eventData.button != InputButton.Left) return;
        Press();
    }

    private void Press() {
        if (!IsActive() || !IsInteractable()) return;
        onClick.Invoke();
    }
}
```

### Toggle

添加了开关状态和 ToggleGroup 互斥机制：

```csharp
public void OnPointerClick(PointerEventData eventData) {
    SetIsOn(!m_IsOn);  // 翻转状态
}

private void SetIsOn(bool value) {
    if (m_IsOn == value) return;
    m_IsOn = value;

    if (m_Group != null && IsActive()) {
        m_Group.NotifyToggleOn(this);  // 通知 ToggleGroup 互斥
    }
    PlayEffect();      // 更新勾选标记的视觉
    m_OnValueChanged.Invoke(m_IsOn);
}
```

`ToggleGroup.allowSwitchOff` 控制是否允许全部取消选中（radio button vs checkbox 的区别）。

### Slider

通过 `IDragHandler` 实现连续值拖拽：

```csharp
public void OnDrag(PointerEventData eventData) {
    // 将屏幕坐标转换为本地坐标
    // 计算相对于滑条方向的位置比例
    // 更新 m_Value，约束到 [minValue, maxValue]
    // 调用 UpdateVisuals() 更新 Fill/Handle 位置
    // 触发 OnValueChanged
}
```

Slider 的四个方向枚举：LeftToRight、RightToLeft、BottomToTop、TopToBottom。

---

## 源码阅读路径

```
Selectable.cs → 完整阅读
  重点关注：
  1. OnEnable / OnDisable（静态列表管理）
  2. DoStateTransition（状态→视觉的核心方法）
  3. FindSelectable（导航算法）
  4. IsInteractable（CanvasGroup 链式检查）

Button.cs → 简单示例，理解子类如何扩展
Toggle.cs → ToggleGroup 互斥机制
Slider.cs → IDragHandler 拖拽值变化
```

| 方法 | 所在文件 | 作用 |
|------|---------|------|
| `DoStateTransition()` | Selectable.cs | 状态→视觉过渡（protected virtual，子类可重写） |
| `FindSelectable(Vector3)` | Selectable.cs | 方向导航搜索算法 |
| `IsInteractable()` | Selectable.cs | 综合判断是否可交互（含 CanvasGroup） |
| `OnPointerClick()` → `Press()` | Button.cs | 点击事件触发 onClick |
| `SetIsOn(bool)` | Toggle.cs | 开关状态切换（含 ToggleGroup 通知） |
| `OnDrag()` → `UpdateDrag()` | Slider.cs | 拖拽更新值 |
