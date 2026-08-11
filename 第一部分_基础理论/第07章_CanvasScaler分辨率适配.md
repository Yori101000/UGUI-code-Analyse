# 第7章 CanvasScaler 分辨率适配

> 本章对应原书结构中的第2章（基础理论部分）。CanvasScaler 决定了 UI 在不同屏幕尺寸和分辨率下的缩放行为，是理解 UGUI 多分辨率适配的核心。

---

## 概述

CanvasScaler 的核心职责只有一件事：**计算一个 `scaleFactor`，然后赋值给 `Canvas.scaleFactor`**。所有子 UI 元素通过这个 scaleFactor 自动缩放。

三种 Scale Mode：

```
ConstantPixelSize → scaleFactor 固定，像素级精确
ScaleWithScreenSize → 根据屏幕分辨率动态计算
ConstantPhysicalSize → 根据屏幕 DPI 保持物理尺寸
```

---

## 7.1 执行入口

CanvasScaler 在 `OnEnable()` 时将 `Handle()` 注册到 `Canvas.preWillRenderCanvases` 事件，**保证在所有 UI 重建之前完成缩放计算**。

```csharp
// CanvasScaler.cs 简化
protected override void OnEnable() {
    Canvas.preWillRenderCanvases += Handle;
}

protected virtual void Handle() {
    switch (m_ScaleMode) {
        case ScaleMode.ConstantPixelSize:     HandleConstantPixelSize();     break;
        case ScaleMode.ScaleWithScreenSize:   HandleScaleWithScreenSize();   break;
        case ScaleMode.ConstantPhysicalSize:  HandleConstantPhysicalSize();  break;
    }
}
```

---

## 7.2 ConstantPixelSize（恒定像素）

scaleFactor 由用户手动指定，固定不变。Canvas 尺寸始终等于屏幕尺寸，UI 元素保持固定的像素尺寸，不随分辨率变化。

```csharp
protected virtual void HandleConstantPixelSize() {
    SetScaleFactor(m_ScaleFactor);
}
```

**适用场景**：像素级精确控制的 UI（如编辑器工具），或不需要适配不同分辨率的简单项目。

---

## 7.3 ScaleWithScreenSize（随屏幕大小缩放）

**最常用的模式**。根据当前屏幕分辨率与参考分辨率的比值计算 scaleFactor。涉及三个核心参数：

```
m_ReferenceResolution → 参考分辨率（如 1920x1080）
m_ScreenMatchMode     → 屏幕匹配模式：Expand / Shrink / MatchWidthOrHeight
m_MatchWidthOrHeight  → Match 模式的混合系数 [0, 1]
```

### 三种 ScreenMatchMode

**① Expand（适配较窄边）**

取宽高比率的**较小值**，保证内容完全可见，类似"内切矩形"：

```csharp
scaleFactor = Mathf.Min(screenWidth / refWidth, screenHeight / refHeight);
```

示例：参考 1920x1080，屏幕 1280x720 → `min(0.667, 0.667) = 0.667`。

**② Shrink（适配较宽边）**

取宽高比率的**较大值**，保证屏幕不留空白，但可能裁剪内容：

```csharp
scaleFactor = Mathf.Max(screenWidth / refWidth, screenHeight / refHeight);
```

**③ MatchWidthOrHeight（加权混合）**

按 `m_MatchWidthOrHeight` 在宽高之间做混合。`0` 时只看宽度，`1` 时只看高度，`0.5` 时取平衡值。

**核心公式（在对数空间计算）**：

```csharp
// 源码中使用对数空间的理由：
// 假设一个轴变 2 倍(ratio=2)，另一个轴变 0.5 倍(ratio=0.5)
// match=0.5 时应该相互抵消 => scaleFactor=1.0

// 线性平均：(0.5 + 2) / 2 = 1.25 ← 错误（放大了）
// 对数平均：(log2(0.5) + log2(2)) / 2 = (-1 + 1) / 2 = 0 → 2^0 = 1.0 ← 正确

float logWidth  = Mathf.Log(screenWidth  / refWidth,  2);
float logHeight = Mathf.Log(screenHeight / refHeight, 2);
float logAvg    = Mathf.Lerp(logWidth, logHeight, m_MatchWidthOrHeight);
scaleFactor     = Mathf.Pow(2, logAvg);
```

等价的简化公式：

```
scaleFactor = (screenWidth/refWidth)^(1-t) × (screenHeight/refHeight)^t
  其中 t = m_MatchWidthOrHeight
```

### 赋值

```csharp
protected void SetScaleFactor(float factor) {
    if (factor == m_PrevScaleFactor) return;  // 无变化则跳过
    m_Canvas.scaleFactor = factor;
    m_PrevScaleFactor = factor;
}
```

---

## 7.4 ConstantPhysicalSize（恒定物理尺寸）

根据屏幕 DPI 计算 scaleFactor，使 UI 在不同 PPI 设备上保持一致的物理大小。

```csharp
protected virtual void HandleConstantPhysicalSize() {
    float dpi = (Screen.dpi == 0) ? m_FallbackScreenDPI : Screen.dpi;
    float targetDPI = m_PhysicalUnit 对应的值;
    SetScaleFactor(dpi / targetDPI);
    SetReferencePixelsPerUnit(m_ReferencePixelsPerUnit * targetDPI / m_DefaultSpriteDPI);
}
```

物理单位映射：

| Unit | targetDPI |
|------|-----------|
| Inches | 1 |
| Centimeters | 2.54 |
| Millimeters | 25.4 |
| Points | 72 |
| Picas | 6 |

**适用场景**：需要在不同 PPI 设备上保持 UI 物理尺寸一致（如印刷级排版 UI）。

---

## 7.5 scaleFactor 对 UI 的影响

CanvasScaler 修改 `m_Canvas.scaleFactor` 后，其影响范围包括：

```
Canvas.scaleFactor
  → Graphic.GetPixelAdjustedRect()     ← pixelPerfect 时的像素对齐
  → Image.sprite 的像素密度计算
  → Text 的字体大小缩放
  → 所有子 RectTransform 的最终显示尺寸
```

---

## 源码阅读路径

```
CanvasScaler.cs → Handle() → HandleScaleWithScreenSize()
                              → SetScaleFactor() → Canvas.scaleFactor
                              → SetReferencePixelsPerUnit()
```

| 方法 | 说明 |
|------|------|
| `Handle()` | 入口，按 ScaleMode 分派 |
| `HandleScaleWithScreenSize()` | ScaleWithScreenSize 模式的核心计算 |
| `HandleConstantPixelSize()` | 固定像素模式 |
| `HandleConstantPhysicalSize()` | 物理尺寸模式 |
| `SetScaleFactor(float)` | 赋值给 Canvas.scaleFactor（含去重） |
| `SetReferencePixelsPerUnit(float)` | 赋值给 Canvas.referencePixelsPerUnit |
