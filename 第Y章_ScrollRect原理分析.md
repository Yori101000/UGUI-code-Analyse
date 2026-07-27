# 第Y章 ScrollRect 原理分析

> ScrollRect 是 UGUI 中最复杂的组件之一，涉及拖拽事件、惯性物理、弹性回弹、滚动条联动、布局系统等多个子系统。本章分析其核心源码实现。

---

## 概述

ScrollRect 继承自 `UIBehaviour`，实现了 9 个接口：

```csharp
public class ScrollRect : UIBehaviour,
    IInitializePotentialDragHandler,  // 拖拽初始化→重置速度
    IBeginDragHandler,                // 开始拖拽→记录起始位置
    IDragHandler,                     // 拖拽中→计算偏移
    IEndDragHandler,                  // 结束拖拽→标记拖拽结束
    IScrollHandler,                   // 滚轮事件
    ICanvasElement,                   // 参与 Canvas 重建管线
    ILayoutElement,                   // 自身作为布局元素
    ILayoutGroup,                     // 影响子级布局
    ILayoutController                 // 布局控制
```

---

## Y.1 核心结构

```
ScrollRect
├── m_Content         ← 可滚动的内容容器（RectTransform）
├── m_ViewRect        ← 可视区域（通常带 Mask 裁剪）
├── m_HorizontalScrollbar  ← 水平滚动条
├── m_VerticalScrollbar    ← 垂直滚动条
├── m_MovementType    ← 运动模式：Unrestricted / Elastic / Clamped
├── m_Inertia         ← 是否启用惯性
├── m_DecelerationRate← 惯性衰减率（默认 0.135）
└── m_Elasticity      ← 弹性系数
```

---

## Y.2 拖拽事件链路

### OnInitializePotentialDrag → 重置速度

```csharp
public virtual void OnInitializePotentialDrag(PointerEventData eventData) {
    m_Velocity = Vector2.zero;  // 清除上次惯性的残留
}
```

### OnBeginDrag → 记录初始状态

```csharp
public virtual void OnBeginDrag(PointerEventData eventData) {
    UpdateBounds();  // 更新 Content 和 Viewport 的边界

    // 将鼠标位置转换到 Viewport 本地坐标
    RectTransformUtility.ScreenPointToLocalPointInRectangle(
        viewRect, eventData.position, eventData.pressEventCamera,
        out m_PointerStartLocalCursor);

    // 记录 Content 起始位置（用于计算拖拽增量）
    m_ContentStartPosition = m_Content.anchoredPosition;
    m_Dragging = true;
}
```

### OnDrag → 计算偏移并移动 Content

```csharp
public virtual void OnDrag(PointerEventData eventData) {
    // 转换当前鼠标位置到 Viewport 本地坐标
    Vector2 localCursor;
    RectTransformUtility.ScreenPointToLocalPointInRectangle(
        viewRect, eventData.position, eventData.pressEventCamera, out localCursor);

    UpdateBounds();
    Vector2 pointerDelta = localCursor - m_PointerStartLocalCursor;
    Vector2 position = m_ContentStartPosition + pointerDelta;

    // 在 Elastic 模式下对超出量做 RubberDelta 拉伸
    if (m_MovementType == MovementType.Elastic) {
        // RubberDelta 将超出量映射到渐近曲线，模拟橡皮筋
    }

    SetContentAnchoredPosition(position);
}
```

`SetContentAnchoredPosition` 只修改启用的轴：

```csharp
protected virtual void SetContentAnchoredPosition(Vector2 position) {
    if (m_Horizontal)
        m_Content.anchoredPosition = new Vector2(position.x, m_Content.anchoredPosition.y);
    if (m_Vertical)
        m_Content.anchoredPosition = new Vector2(m_Content.anchoredPosition.x, position.y);
}
```

### OnEndDrag → 只是标记

```csharp
public virtual void OnEndDrag(PointerEventData eventData) {
    m_Dragging = false;  // 惯性滑动在 LateUpdate 中处理
}
```

---

## Y.3 LateUpdate：物理模拟中心

惯性衰减和弹性回弹全部在 `LateUpdate` 中处理，不在 `OnDrag` 中：

### 惯性衰减

```csharp
// 指数衰减
m_Velocity[x] *= Mathf.Pow(m_DecelerationRate, Time.unscaledDeltaTime);
if (Mathf.Abs(m_Velocity[x]) < 1f) m_Velocity[x] = 0;

// 应用速度到位置
position[x] += m_Velocity[x] * Time.unscaledDeltaTime;
```

关键点：
- `m_DecelerationRate` 默认 0.135。0 = 瞬间停止，1 = 永不停止。
- 使用 `Time.unscaledDeltaTime`，**不受 timeScale 影响**。

### 弹性回弹（Elastic 模式）

```csharp
if (m_MovementType == MovementType.Elastic && offset[axis] != 0) {
    float refVel = m_Velocity[axis];
    position[axis] = Mathf.SmoothDamp(
        position[axis],                    // 当前位置
        position[axis] + offset[axis],      // 目标位置（边界内）
        ref refVel,                         // 瞬时速度（ref）
        m_Elasticity,                       // 弹性系数
        Mathf.Infinity,
        Time.unscaledDeltaTime
    );
    m_Velocity[axis] = refVel;
}
```

`Mathf.SmoothDamp` 模拟弹簧-阻尼系统，**永不超调**（never overshoots），值越大回弹越慢。

### Clausped 模式

直接修正位置到边界内，没有弹性动画。

---

## Y.4 MovementType 三种模式对比

| 模式 | 拖拽中 | 抬起后 | 适用场景 |
|------|--------|--------|---------|
| **Unrestricted** | 无边界限制 | 惯性继续外滑 | 无限轮播 |
| **Elastic** | RubberDelta 拉伸 | SmoothDamp 弹性回弹 | iOS 风格列表 |
| **Clamped** | 卡在边界不动 | 直接停止 | 严格边界控制 |

---

## Y.5 与 Scrollbar 联动

双向绑定：

```
用户拖拽 Content
  → LateUpdate 更新位置 + UpdateScrollbars
    → 设置 Scrollbar.value

用户拖拽 Scrollbar
  → Scrollbar.onValueChanged
    → SetHorizontalNormalizedPosition(value)
      → 根据 value(0~1) 计算 Content 位置
```

`UpdateScrollbars` 核心计算：

```csharp
// value = 当前偏移量 / (Content总尺寸 - Viewport尺寸)
// size  = Viewport尺寸 / Content总尺寸（滑块长度占比）
m_HorizontalScrollbar.SetValueWithoutNotify(value);
m_HorizontalScrollbar.size = size;
```

`size` 表示可视区域占总内容的比例（如 Viewport 200 / Content 1000 = 0.2），不应硬编码。

---

## Y.6 布局系统参与

ScrollRect 实现了 `ILayoutGroup` 和 `ILayoutController`，参与 Canvas 的 Layout 重建：

```csharp
public void SetLayoutHorizontal() {
    // 处理 m_ViewRect 的 AutoHideAndExpand 逻辑
    // 更新 m_Content 的 sizeDelta
}

public void Rebuild(CanvasUpdate executing) {
    // 在 Prelayout 阶段更新缓存数据
    // 在 PostLayout 阶段更新边界
}
```

这使得 ScrollRect 在布局发生变化（如 Content 尺寸改变）时能自动调整滚动范围。

---

## 源码阅读路径

```
ScrollRect.cs → 完整阅读
  重点方法：
  1. OnBeginDrag / OnDrag / OnEndDrag → 拖拽事件链路
  2. LateUpdate → 惯性 + 弹性 + 边界修正 + 滚动条同步
  3. SetContentAnchoredPosition → 实际修改 Content 位置
  4. UpdateScrollbars → Scrollbar 联动
  5. CalculateOffset → 边界超出量计算
  6. GetBounds / AdjustBounds → 边界计算
```

| 方法 | 作用 |
|------|------|
| `OnInitializePotentialDrag` | 重置速度 |
| `OnBeginDrag` | 记录起始位置、更新边界 |
| `OnDrag` | 计算偏移、RubberDelta 拉伸 |
| `OnEndDrag` | 设置 m_Dragging = false |
| `LateUpdate` | 惯性衰减、弹性回弹、边界修正、更新滚动条 |
| `SetContentAnchoredPosition` | 设置 Content 的 anchoredPosition |
| `UpdateScrollbars` | 同步 Scrollbar 的 value 和 size |
| `RubberDelta` | Elastic 模式下的拖拽拉伸计算 |
