# 第29章 Profiler 实战：从抓到修

> 本章通过三个真实案例，演示如何使用 Profiler + Frame Debugger + 源码定位，完整走通"发现问题 → 定位根因 → 修复实现"的闭环。

---

## 案例一：背包列表滚动卡顿

### 现象

背包打开时第一次滚动不卡，滚动到中间区域后出现明显的帧率下降。

### 抓取

1. 打开 Profiler（Window → Analysis → Profiler）
2. 切换到 Timeline 模式
3. 录制背包操作
4. 找到卡顿帧

### 分析

在卡顿帧中，`LayoutRebuilder.Rebuild` 占用 **8.2ms**：

```
Profiler Timeline 卡顿帧：
  ├── Canvas.SendWillRenderCanvases: 9.1ms
  │   ├── LayoutRebuilder.Rebuild: 8.2ms  ← 瓶颈
  │   │   └── VerticalLayoutGroup.SetLayoutVertical
  │   │       └── 遍历 80 个子节点
  │   ├── Graphic.Rebuild: 0.7ms
  │   └── Canvas.BuildBatch: 0.2ms
```

### 定位

代码发现 Content 使用 `VerticalLayoutGroup` + `ContentSizeFitter`：

```csharp
// 问题代码：每次添加物品触发全量 Layout Rebuild
void AddItem(ItemData data)
{
    var item = Instantiate(itemPrefab, contentTransform);
    // item 加入 → VerticalLayoutGroup 重新排布所有 80 个已有 item
}
```

ContentSizeFitter + VerticalLayoutGroup 的组合意味着每次新增 Item 时：
1. LayoutGroup 重新遍历所有子节点计算尺寸
2. ContentSizeFitter 调整 Content 高度
3. ScrollRect 重新计算滚动范围

当 item 数量达到 80 时，一次完整的 Layout Rebuild 耗时 8ms。

### 修复

**方案：对象池 + 固定高度 + 移除 ContentSizeFitter**

```csharp
public class OptimizedBagList : MonoBehaviour
{
    [SerializeField] private float itemHeight = 120f;
    private Queue<ItemSlot> pool = new Queue<ItemSlot>();

    public void AddItem(ItemData data)
    {
        var slot = GetFromPool();
        slot.Bind(data);
        // 手动设置位置，不触发 Layout Rebuild
        slot.rectTransform.anchoredPosition = new Vector2(0, -currentIndex * itemHeight);
        currentIndex++;
    }
}
```

### 结果

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| LayoutRebuilder.Rebuild | 8.2ms | 0ms |
| Graphic.Rebuild | 0.7ms | 0.3ms |
| 总帧耗时 | 12ms | 3ms |

---

## 案例二：HUD 每帧重建

### 现象

游戏运行时 Profiler 中每帧都出现 `Graphic.Rebuild`，即使 HUD 数据没有变化。

### 抓取

1. Profiler 切换到 Hierarchy 模式
2. 按 `Graphic.Rebuild` 排序
3. 观察调用栈

```
Profiler Hierarchy:
  Graphic.Rebuild (每帧 15 次)  ← 异常
  ├── Text.SetVerticesDirty
  │   └── (调用栈中有 Animator.Evaluate)
  ├── Image.SetVerticesDirty
      └── (调用栈中有 Animator.Evaluate)
```

### 定位

发现 HUD 根对象上挂了一个 Animator，有一个空动画（无关键帧）在持续播放：

```csharp
// 问题：空动画持续播放，导致每帧调用 OnDidApplyAnimationProperties
// → SetAllDirty() → 所有 Graphic 每帧重建
```

从源码可以确认：

```csharp
// Graphic.cs
protected override void OnDidApplyAnimationProperties()
{
    SetAllDirty();  // 动画属性应用后全标记
}
```

### 修复

```csharp
// 方案 A：停止不需要的动画
void Start()
{
    animator.enabled = false;  // 关闭无用的 Animator
}

// 方案 B：将动画中的 UI 分离到独立 Canvas
// HUD Canvas（静态） + AnimCanvas（动画）
```

### 结果

`Graphic.Rebuild` 从每帧 15 次降为 0，帧耗时降低 4ms。

---

## 案例三：Mask 导致 DrawCall 暴增

### 现象

Frame Debugger 中 DrawCall 数量远超预期。场景中只使用了 10 个 UI 元素，DrawCall 却有 15。

### 抓取

1. Window → Analysis → Frame Debugger → Enable
2. 逐条检查 DrawCall

```
Frame Debugger:
  DrawCall #0: UI/Default, TextureA, Stencil Ref=0  ← Image1
  DrawCall #1: UI/Default, TextureA, Stencil Ref=2  ← Mask 内部
  DrawCall #2: UI/Default, TextureA, Stencil Ref=0  ← Mask 外部
  DrawCall #3: UI/Default, TextureA, Stencil Ref=2  ← Mask 内部
  ...
```

每一个在 Mask 内的 UI 元素都使用了不同的 Stencil Ref，导致每个元素单独一个 DrawCall。

### 定位

查看 `Mask.cs` 和 `StencilMaterial.cs` 的源码：

```csharp
// StencilMaterial.cs 简化
public static Material Add(Material baseMat, int stencilID, ...)
{
    // 为不同 stencilID 创建不同 Material 实例
    // 不同 Material 实例 = 不同 Batch
}
```

当 Mask 嵌套深度不同时，`stencilID` 不同 → Material 实例不同 → 无法合批。

### 修复

```csharp
// 方案 A：同一层级 Mask 下的 UI 使用相同材质
// → Stencil 参数相同 → 共享 Material 实例 → 可合批

// 方案 B：用 RectMask2D 替代 Mask
// RectMask2D 不修改材质，不产生额外 DrawCall
// 仅限制在矩形裁剪——但零 DrawCall 开销
```

### 结果

| 方案 | DrawCall | 说明 |
|------|---------|------|
| 原始（Mask 嵌套） | 15 | 每层 Mask 断批 |
| 同层 Mask 合并 | 8 | 同深度子元素合批 |
| 替换为 RectMask2D | 3 | 零额外 DrawCall |
| 优化后目标 | 3 | 仅 3 个批次 |
