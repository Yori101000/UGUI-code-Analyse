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

Selectable 定义了**五种**状态：

```csharp
// Selectable.cs（main）
protected enum SelectionState
{
    Normal,
    Highlighted,
    Pressed,
    Selected,
    Disabled,
}
```

| 状态 | 触发条件 |
|------|---------|
| **Normal** | 无任何交互 |
| **Highlighted** | 鼠标悬停在区域内 |
| **Pressed** | 鼠标按下 |
| **Selected** | 被 EventSystem 选中（键盘/手柄导航焦点、或代码 `EventSystem.SetSelectedGameObject`） |
| **Disabled** | `interactable = false`，或父级 CanvasGroup 禁用交互 |

> ⚠️ **Selected 是独立状态，不是 Highlighted 的别名。** `ColorBlock` 里有独立的 `selectedColor`、`SpriteState` 里有独立的 `selectedSprite`、Animation 模式有独立的 `Selected` 触发器，三者都可以单独配置。
>
> 容易被误判成"没有区别"是因为**默认值撞车**：`defaultColorBlock` 里 `highlightedColor` 和 `selectedColor` 都是 `(245,245,245,255)`。默认长得一样，不代表是同一个状态——把 `selectedColor` 改成别的颜色，手柄导航焦点和鼠标悬停立刻能区分开。这对做手柄/键盘导航的项目是必需的。

### 13.1.2 状态评估逻辑

Selectable 内部通过三个布尔变量追踪交互状态：

```csharp
bool isPointerInside;  // 鼠标是否在区域内
bool isPointerDown;    // 鼠标是否按下
bool hasSelection;     // 是否被 EventSystem 选中
```

判定写在 `currentSelectionState` **属性**里（不是某个 `UpdateSelectionState()` 方法），顺序是：

```csharp
// Selectable.cs（main，currentSelectionState 的判定顺序）
if (!IsInteractable())      → SelectionState.Disabled
else if (isPointerDown)     → SelectionState.Pressed
else if (hasSelection)      → SelectionState.Selected     ← 独立分支
else if (isPointerInside)   → SelectionState.Highlighted
else                        → SelectionState.Normal
```

**顺序即优先级**：按下 > 选中 > 悬停。所以一个已获得导航焦点的按钮，鼠标移上去仍显示 Selected 而非 Highlighted——因为 `hasSelection` 的判断在前。

### 13.1.3 DoStateTransition——视觉过渡的执行

当状态发生变化时，`DoStateTransition(SelectionState state, bool instant)` 被调用：

```csharp
// Selectable.cs（main，简化：省略了 gameObject.activeInHierarchy 判断等细节）
protected virtual void DoStateTransition(SelectionState state, bool instant) {
    // ① 按状态取出三套资源（五个分支，与 SelectionState 一一对应）
    Color tintColor;  Sprite transitionSprite;  string triggerName;
    switch (state) {
        case SelectionState.Normal:
            tintColor = colors.normalColor;      transitionSprite = null;
            triggerName = animationTriggers.normalTrigger;      break;
        case SelectionState.Highlighted:
            tintColor = colors.highlightedColor; transitionSprite = spriteState.highlightedSprite;
            triggerName = animationTriggers.highlightedTrigger; break;
        case SelectionState.Pressed:
            tintColor = colors.pressedColor;     transitionSprite = spriteState.pressedSprite;
            triggerName = animationTriggers.pressedTrigger;     break;
        case SelectionState.Selected:            // ← 独立分支，有自己的颜色/图片/触发器
            tintColor = colors.selectedColor;    transitionSprite = spriteState.selectedSprite;
            triggerName = animationTriggers.selectedTrigger;    break;
        case SelectionState.Disabled:
            tintColor = colors.disabledColor;    transitionSprite = spriteState.disabledSprite;
            triggerName = animationTriggers.disabledTrigger;    break;
    }

    // ② 按 Transition 类型执行视觉变化
    switch (m_Transition) {
        case Transition.ColorTint:  StartColorTween(tintColor * colors.colorMultiplier, instant); break;
        case Transition.SpriteSwap: DoSpriteSwap(transitionSprite);                               break;
        case Transition.Animation:  TriggerAnimation(triggerName);                                break;
    }
}
```

注意 ColorTint 分支里的 `colors.colorMultiplier`（取值 1~5）：它会把颜色整体放大，用于做超出 1.0 的高亮效果——这也是 `ColorBlock` 里除五种颜色外的两个额外字段之一（另一个是 `fadeDuration`）。

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

`FindSelectable(Vector3 dir)` 在 Automatic 模式下被调用。它的骨架是"遍历全部 Selectable → 过滤 → 打分 → 取最高分"：

```csharp
// 结构示意，非逐行照抄——真实实现的打分细节比这里复杂
public virtual Selectable FindSelectable(Vector3 dir) {
    dir = dir.normalized;
    Selectable bestPick = null;
    float bestScore = float.MinValue;

    for (int i = 0; i < s_SelectableCount; ++i) {   // 遍历静态数组，不是 List
        Selectable sel = s_Selectables[i];
        if (sel == this || sel == null) continue;
        if (!sel.IsInteractable() || sel.navigation.mode == Navigation.Mode.None) continue;

        // 用目标的 RectTransform 边界而非单纯的 transform.position 参与计算，
        // 并综合"方向对齐度"与"距离"给出得分
        float score = /* 方向与距离的综合评分 */;
        if (score > bestScore) { bestScore = score; bestPick = sel; }
    }
    return bestPick;
}
```

**行为本质**：优先选"在目标方向上、且距离最近"的控件。

两个实际会遇到的细节：

- **参与计算的是 RectTransform 的边界，不是中心点**。所以一个很宽的按钮，从它左下方按"上"键，仍可能选中它——因为按边界算距离更近。
- **支持环绕（wrap around）**：开启后，在最边缘继续按方向键会跳到反方向的最远端（列表尾 → 列表头）。这是 `Navigation` 上的可选项，不是默认行为。
- **遍历的是场景中全部激活的 Selectable**，不做空间划分。界面上有几百个交互控件时，每次导航都是一次全量遍历——这也是 `s_Selectables` 用数组而非 `List` 的原因。

---

## 13.4 静态列表与生命周期

### 全局 Selectable 列表

场景中所有激活的 Selectable 被注册到一个**静态数组**（不是 `List`）：

```csharp
// Selectable.cs（main）
protected static Selectable[] s_Selectables = new Selectable[10];
protected static int s_SelectableCount = 0;
```

用定长数组 + 计数、按需扩容，而不是 `List<T>`——目的是让导航搜索（`FindSelectable`）遍历时零分配、零装箱。`OnEnable` 时把自己追加进数组并递增计数，`OnDisable` 时把末尾元素挪到自己的位置再递减计数（swap-remove，O(1)）。

访问它用 `Selectable.allSelectablesArray` / `allSelectableCount`；`Selectable.allSelectables` 这个返回 `List` 的旧 API 已标记过时，因为它每次调用都要分配。

`OnDisable` 还会调用 `InstantClearState()`，把过渡状态立即重置（ColorTint 重置为白色、SpriteSwap 清掉 `overrideSprite`、Animation 触发 Normal）——避免对象池复用时残留上一次的按下/高亮外观。

### IsInteractable——综合判断

```csharp
public virtual bool IsInteractable() {
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
// Button.cs（main）
public class Button : Selectable, IPointerClickHandler, ISubmitHandler {
    public ButtonClickedEvent onClick;

    public virtual void OnPointerClick(PointerEventData eventData) {
        if (eventData.button != PointerEventData.InputButton.Left) return;
        Press();
    }

    private void Press() {
        if (!IsActive() || !IsInteractable()) return;

        UISystemProfilerApi.AddMarker("Button.onClick", this);   // Profiler 采样点
        m_OnClick.Invoke();
    }

    public virtual void OnSubmit(BaseEventData eventData) {
        Press();                                    // 键盘/手柄 Submit 也走 Press
        if (!IsActive() || !IsInteractable()) return;
        DoStateTransition(SelectionState.Pressed, false);
        StartCoroutine(OnFinishSubmit());           // 短暂显示按下态再恢复
    }
}
```

两点值得注意：

- `Press()` 里的 `UISystemProfilerApi.AddMarker("Button.onClick", this)` —— **这就是你在 Profiler 里看到 `Button.onClick` 条目的来源**。onClick 回调里干了重活，会直接体现在这个采样点上。
- `OnSubmit` 走的是同一个 `Press()`，但额外手动切到 Pressed 状态并起一个协程延时恢复——因为键盘提交没有"按下-抬起"的过程，不这么做就看不到按下反馈。

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

---

## 勘误汇总

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 严重 | 13.1.1 / 13.1.2 | 「Selectable 定义了**四种**状态」，且「**Selected（选中）和 Highlighted（悬停）在视觉上使用相同的状态值**，两者没有视觉区别」 | `SelectionState` 有**五个**成员：`Normal / Highlighted / Pressed / Selected / Disabled`。`Selected` 是完全独立的状态——`ColorBlock.selectedColor`、`SpriteState.selectedSprite`、`AnimationTriggers.selectedTrigger` 三套资源都独立可配。之所以容易误判，是因为 `defaultColorBlock` 里 `selectedColor` 与 `highlightedColor` 的**默认值恰好都是 (245,245,245,255)**——默认撞车不等于同一状态。做手柄/键盘导航的项目必须区分这两者 |
| 2 | 🔴 严重 | 13.1.2 | 状态判定在 `UpdateSelectionState()` 方法中，顺序为 Disabled → Pressed → `hasSelection` **→ Highlighted** → Normal | main 中是 `currentSelectionState` **属性**（无 `UpdateSelectionState()` 方法）；`hasSelection` 命中的是 **`SelectionState.Selected`** 而非 Highlighted。正确顺序：Disabled → Pressed → **Selected** → Highlighted → Normal |
| 3 | 🟡 中等 | 13.1.3 | `DoStateTransition` 的 switch 只有 Normal / Highlighted / Pressed / Disabled 四个分支 | 有五个分支，缺 `Selected`。另外 ColorTint 分支实际传的是 `tintColor * colors.colorMultiplier`，原文省略了 `colorMultiplier` |
| 4 | 🟡 中等 | 13.4 | 静态列表为 `private static List<Selectable> s_List`，`OnEnable` 用 `s_List.Add`、`OnDisable` 用 `s_List.Remove` | main 用**静态数组** `protected static Selectable[] s_Selectables = new Selectable[10]` + `s_SelectableCount` 计数，移除用 swap-remove（O(1)）。用数组而非 List 是为了让导航遍历零分配；返回 `List` 的 `allSelectables` 旧 API 已标记过时 |
| 5 | 🟢 轻微 | 13.3 | `FindSelectable` 给出了具体打分公式 `dot / toTarget.sqrMagnitude`，并遍历 `s_List` | 真实实现基于 RectTransform 边界而非 `transform.position` 打分，另支持环绕（wrap around），细节比原文复杂。已改为结构示意并标注"非逐行照抄"，同时补充三条实际影响 |
| 6 | 🟢 轻微 | 13.4 | `public bool IsInteractable()` | main 为 `public virtual bool IsInteractable()`（子类可重写） |
| 7 | 🟢 轻微 | 13.5 | `Press()` 只调用 `onClick.Invoke()`；未提 `OnSubmit` | `Press()` 内含 `UISystemProfilerApi.AddMarker("Button.onClick", this)`——这是 Profiler 中 `Button.onClick` 条目的来源。另 `OnSubmit` 会手动切 Pressed 态并起协程恢复，原文完全未涉及 |

