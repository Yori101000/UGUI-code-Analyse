# 第26章 DOTween 原理与 UGUI 集成

> 本章讲 DOTween 的**底层机制**——它为什么比手写协程快、对象池怎么工作、Tween 怎么驱动、怎么与 UGUI 生命周期整合。不是 API 手册（官方文档已有）。

---

## 概述

DOTween 是目前 Unity 使用最广泛的 Tween 库。它不是在 Update 里每帧 Lerp，而是一套完整的**补间动画引擎**，核心包括：全局驱动、对象池、链式调用、UGUI 扩展。

与三种"原生"方案对比：

| 方案 | 每帧驱动方式 | Tween 对象管理 | GC 分配 | 链式调用 |
|------|------------|---------------|:------:|:-------:|
| 手写协程 | 每个动画一个协程 | 无 | 高（协程 + 临时分配） | ❌ |
| Animator | Animator 组件 | 组件管理 | 低 | ❌ |
| DOTween | 全局 TweenManager | **对象池复用** | **极低** | ✅ |

DOTween 的核心设计目标就是：**零分配（zero allocation）**。它通过对象池复用 Tween 对象、通过委托而非反射驱动属性、通过全局统一 Update 调度来做到这一点。

---

## 26.1 DOTween 的总体架构

```
调用方（如 transform.DOMoveX(5, 1)）
  ↓
DOTween 的扩展方法
  → 从对象池取一个 Tween 对象（不 new）
    → 设置：目标对象、属性 getter/setter、起始/结束值、时长、缓动函数
      → 加入 TweenManager 的更新列表
        ↓
TweenManager.Update（每帧在 DOTween 的全局 Update 中调用）
  → 遍历所有活动的 Tween
    → 计算进度 t = elapsed / duration
      → 应用缓动函数 t' = Ease(t)
        → 通过 setter 委托写入属性值
          → t >= 1 → 回调 onComplete → Tween 回收入池（AutoKill）
```

核心组件：

| 组件 | 角色 |
|------|------|
| `Tweener` | 驱动单个属性从 A 到 B 的插值 |
| `Sequence` | 管理多个 Tween 的顺序/并行执行 |
| `TweenManager` | 全局调度器，管理所有活动 Tween 的生命周期 |
| `DOTween` | 全局配置（容量、回收策略、Update 触发时机） |

---

## 26.2 Tween 的对象池机制

### 26.2.1 为什么需要池

每次 StartCoroutine 都会产生 GC Alloc（协程实例 + Enumerator）。一个界面同时播放几十个动画，每播放一轮就分配一轮，GC 压力明显。

DOTween 的做法：**Tween 对象不是 new 出来的，是从池里取的。**

```csharp
// 简化的池逻辑
internal static class TweenPool
{
    private static Stack<Tweener> pool = new Stack<Tweener>(capacity);

    public static Tweener Get()
    {
        return pool.Count > 0 ? pool.Pop() : new Tweener();
    }

    public static void Release(Tweener t)
    {
        t.Reset();      // 清空状态
        pool.Push(t);   // 回池
    }
}
```

当调用 `transform.DOMoveX(5, 1)` 时：

```
DOTween 内部 = TweenPool.Get() + 设置参数 + TweenManager.Add()
动画完成   = onComplete 回调 + TweenPool.Release()
```

全程没有 `new Tween()`，没有 `StartCoroutine`。

### 26.2.2 AutoKill 与 SetRecyclable

```csharp
// 默认行为：动画播完自动销毁（回池）
transform.DOMoveX(5, 1);                   // AutoKill = true

// 需要反复播放的动画，关闭 AutoKill，避免每播一次从池里取一次
transform.DOMoveX(5, 1).SetAutoKill(false); // 播完保留，可以 Rewind 重播

// 明确告诉 DOTween 这个 Tween 可以回收
transform.DOMoveX(5, 1).SetRecyclable(true);
```

**性能关键**：高频反复播放的动画（如按钮 hover 缩放），应使用 `SetAutoKill(false)` + `SetRecyclable(true)`，避免反复出入池。

---

## 26.3 Tween 的驱动方式：委托而非反射

手写脚本的典型做法：

```csharp
void Update()
{
    t += Time.deltaTime;
    target.localScale = Vector3.Lerp(start, end, t / duration);
}
```

DOTween 如果用反射来读写属性，每次 set 都有反射开销。它用的是**预编译的委托**：

```
DOFloat(float3.DOScaleX(2, 1)
  → DOTween 内部注册 getter = () => transform.localScale.x
                      setter = x => transform.localScale = new Vector3(x, transform.localScale.y, transform.localScale.z)
```

在 DOTween 的源代码里，这些 setter/getter 是编译时通过代码生成（DOTween's `Plugins` 系统）预定义的，不是运行时反射。UGUI 相关的扩展（`DOMove`、`DOFade`、`DOAnchorPos` 等）也是同样的机制。

### UGUI 扩展方法清单及影响

| 扩展方法 | 修改的属性 | 触发的 UGUI 行为 |
|---------|-----------|----------------|
| `DOMove` / `DOMoveX` / `DOMoveY` | `Transform.position` | Canvas 包围盒更新 |
| `DOLocalMove` | `Transform.localPosition` | Canvas 包围盒更新 |
| `DOScale` | `Transform.localScale` | Canvas 包围盒更新 |
| `DOAnchorPos` / `DOAnchorPosX/Y` | `RectTransform.anchoredPosition` | Canvas 包围盒更新（无 Layout 时不触发 Rebuild） |
| `DOFade`（对 CanvasGroup） | `CanvasGroup.alpha` | **不触发 Rebuild**（纯 GPU） |
| `DOFade`（对 Graphic） | `Graphic.color.a` | **触发 Graphic Rebuild**（顶点颜色变化） |
| `DOColor` | `Graphic.color` | **触发 Graphic Rebuild** |
| `DOFillAmount` | `Image.fillAmount` | **触发 Graphic Rebuild** |
| `DOText` | `Text.text` | **触发完整文本重建**（最贵） |
| `DOSizeDelta` | `RectTransform.sizeDelta` | **触发 Layout Rebuild**（最贵，递归） |

**性能排序（从低到高）：**

```
CanvasGroup.DOFade（极低）< DOScale/DOMove（低）< DOColor（中）< DOText/DOSizeDelta（高）
```

---

## 26.4 Sequence：复杂动画编排

Sequence 不是多个 Tween 各跑各的，而是一个**容器 Tween**，内部按时间轴管理子 Tween：

```csharp
Sequence seq = DOTween.Sequence();
seq.Append(panel.DOScale(1, 0.3f));     // 第一步：面板弹出
seq.Append(icon.DOFade(1, 0.2f));       // 第二步：图标渐显
seq.Join(title.DOFade(1, 0.2f));        // 和第二步同时：标题渐显
seq.AppendInterval(1f);                 // 等待 1 秒
seq.Append(btn.DOAnchorPosY(0, 0.3f));  // 最后：按钮滑入
```

Sequence 内部维护时间轴——子 Tween 的起始时间由它在 Sequence 中的位置决定，不是真的启动 N 个独立 Tween。

**性能注意**：Sequence 本身也是一个池化的 Tween 对象。嵌套 Sequence（Sequence 里再放 Sequence）能正常工作，但深度嵌套会影响可读性，不会显著影响性能。

---

## 26.5 DOTween 与 UI 生命周期

### 26.5.1 目标失效保护

DOTween 内部在每次更新时会检查 `target` 是否为 null 或 `Equals(null)`（Unity 的伪 null），如果目标已被 Destroy，Tween 自动终止不回抛异常。这是 DOTween 相对手写协程的一个重要优势——手写协程在 target 被销毁后还在试图修改它，会抛 MissingReferenceException。

### 26.5.2 OnComplete 的正确用法

```csharp
// 界面关闭：等待退出动画完成再 Destroy
uiPanel.DOScale(0, 0.3f).OnComplete(() =>
{
    Destroy(uiPanel.gameObject);  // 动画播完再销毁
});
```

### 26.5.3 Kill 的时机

UI 被隐藏或回收时，应该 Kill 掉还在播放的 Tween，避免下一帧又去修改已经隐藏或复用的 UI：

```csharp
public override void OnHide()
{
    transform.DOKill();             // Kill 这个对象上所有活动的 Tween
    // 或：DOTween.Kill(this);     // 效果相同
}
```

忘记 Kill 的后果：UI 已 SetActive(false) 但 Tween 还在驱动它的属性 → Unity 仍然会更新 Transform → 浪费 CPU。

---

## 26.6 DOTween 的性能实测

| 场景 | 手写协程 | DOTween |
|------|---------|---------|
| 100 个 UI 同时播放 0.3s 动画 | ~8ms GC Alloc（协程创建）| **~0ms GC Alloc** |
| 持续 10 分钟反复播放 | GC 频繁，卡顿明显 | 稳定（零分配） |
| 目标销毁后继续运行 | MissingReferenceException | 自动终止 |

DOTween 的性能优势完全来自**对象池 + 委托 + 全局驱动**这三条，不是"它用了什么黑魔法"。

---

## 本章总结

```
DOTween 不是"简单的 Lerp 工具"，而是一个补间动画引擎：

对象池 → 零 GC 分配（关键优势）
委托   → 无反射开销
全局驱动 → 无协程开销
Sequence → 免手动计时编排

UGUI 集成要点：
  CanvasGroup.DOFade 不触发 Rebuild（最优）
  DOAnchorPos/DOScale 只更新包围盒（次优）
  DOColor/DOFillAmount 触发 Graphic Rebuild
  DOSizeDelta 触发 Layout Rebuild（避免）
  OnHide 时 DOKill（避免无效更新）
```
