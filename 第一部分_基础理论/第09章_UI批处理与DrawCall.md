# 第9章 UI 批处理与 DrawCall

> 批处理（Batching）是 UGUI 渲染体系中最重要的性能优化机制——它将多个 UI 元素的 Mesh 合并为同一个 DrawCall 提交，大幅减少 GPU 状态切换的开销。

---

## 9.1 为什么需要批处理

### 9.1.1 GPU 状态切换的成本

GPU 的核心工作模式是"设置状态 → 提交命令 → GPU 执行"。每当 GPU 处理一个新的 DrawCall 时，需要先切换到该 DrawCall 所需的完整渲染状态：

```
提交 DrawCall N：
  设置 Shader → 上传 Shader 参数（Uniform）
  绑定 Material → 设置 Blend / ZWrite / Stencil 等状态
  绑定 Texture → 设置纹理采样器
  绑定 Vertex Buffer → 设置顶点格式
  提交绘制命令 → GPU 开始执行

提交 DrawCall N+1（如果状态不同）：
  重新设置以上所有状态 → GPU 状态切换
```

每次状态切换，CPU 需要将新状态打包成 GPU 能识别的命令，通过图形 API（DirectX / OpenGL / Vulkan / Metal）送入 GPU 驱动。这个过程具有固定的 CPU 开销和驱动层开销，并且会打断 GPU 的指令流水线。

对于 CPU 侧而言，提交 100 个只有一个 Quad 的 DrawCall，比提交 1 个包含 100 个 Quad 的合并 DrawCall 开销大得多。**状态切换的固定成本是 DrawCall 性能问题的根源。**

### 9.1.2 UGUI 的渲染特点

UGUI 的 UI 布局通常具有以下特点：

- **大量小 Mesh**：一个 Button 只有 1 个 Quad（2 个三角形），一个 Text 可能由几十个 Quad 组成
- **层级密集**：弹窗、列表、HUD 等界面有大量 UI 元素叠加
- **共享资源广泛**：多数 Image 使用默认材质（`UI/Default`），多数 Text 使用默认字体材质

如果每个 UI 元素都单独提交一个 DrawCall，一个稍微复杂的界面就可能产生数百甚至上千个 DrawCall——而 GPU 大多数时间会浪费在状态切换上，而不是实际的像素渲染。

**批处理的核心思想**：将共享相同渲染状态的 UI 元素合并为同一个 DrawCall，让 GPU 一次性完成绘制。

---

## 9.2 批处理的整体流程

UGUI 不会将每个 UI 元素单独提交给 GPU。所有 UI 元素先由 Canvas 统一收集，经过批处理后再提交。完整的流程如下：

```
每帧渲染循环
  │
  ├─ CanvasUpdateRegistry.PerformUpdate()
  │    遍历 dirty Graphic，执行 Rebuild
  │    → Graphic 生成 Mesh 数据
  │    → CanvasRenderer.SetMesh() 暂存
  │
  └─ Canvas.BuildBatch()           ← 关键：批处理在此发生
       遍历当前 Canvas 下所有 CanvasRenderer
       按材质/纹理分组
       同组 Mesh 合并为一个大合并 Mesh
       生成 DrawCall → 提交到 GPU Command Buffer
```

### 9.2.1 三个角色的分工

| 角色 | 职责 | 与批处理的关系 |
|------|------|--------------|
| **Graphic** | 生成顶点数据（Image/Text/RawImage） | 提供原始的 Mesh 数据 |
| **CanvasRenderer** | 数据容器，存储 Mesh 和 Material | 是 BuildBatch 遍历的对象 |
| **Canvas** | 负责收集、合并、提交 | BuildBatch 执行批处理逻辑 |

### 9.2.2 BuildBatch 的本质

`Canvas.BuildBatch()` 是一个 Unity 引擎的 native 方法，C# 侧看不到其实现。通过 Frame Debugger 可以观察到它的执行效果：

1. 遍历当前 Canvas 下的所有 CanvasRenderer
2. 从每个 CanvasRenderer 中读取已存储的 Mesh、Material、Texture
3. 按照**材质实例 + 纹理 + Shader + Stencil 状态**等条件对 CanvasRenderer 分组
4. 将同一组的 Mesh 合并为一个大的 Vertex Buffer + Index Buffer
5. 为每个合并后的 Buffer 生成一个 DrawCall，提交到 GPU

**关键认知**：DrawCall 的真正提交发生在 `Canvas.BuildBatch()` 内部，而不是 CanvasRenderer。CanvasRenderer 只是数据暂存层。

### 9.2.3 UGUI 提交与非 UI 渲染的对比

| | 非 UI 的 MeshRenderer | UGUI 的 CanvasRenderer |
|---|---|---|
| Mesh 来源 | MeshFilter 组件 | Graphic.OnPopulateMesh 生成 |
| 提交方式 | 每个 MeshRenderer 一个 DrawCall（无合并） | 多个 UI 合并为一个 DrawCall |
| 材质来源 | Material 属性 | CanvasRenderer 暂存 |
| 合并逻辑 | 无（引擎级静态合并除外） | Canvas.BuildBatch 负责合并 |

这就是为什么 UGUI 下即使有上百个 Image，DrawCall 数量也可能只有十几个——只要它们共享材质和纹理，就会被合并到少量 Big Mesh 中。

---

## 9.3 合批条件：精确匹配

### 9.3.1 必要条件

两个 UI 元素要合并到同一个 DrawCall，必须**精确匹配**以下所有条件：

```
合并条件（缺一不可）：
  ├── 相同 Material 实例（同一个 C# 对象，值相等不够）
  ├── 相同 Texture（同一张纹理）
  ├── 相同 Shader + 相同 Shader Keywords
  ├── 相同 Stencil 状态（Ref / Comp / Pass / Fail）
  ├── 相同 RectMask2D 裁剪矩形（或者都未设置）
  ├── 属于同一个 Canvas
  └── Additional Shader Channels 一致
```

**"精确匹配"的含义**：不是"看起来一样就行"，必须是**同一个内存对象**。

> 以上条件依据 Unity 官方文档与 Frame Debugger 观察总结——合批规则在引擎 Native 层实现，C# 侧无源码。

### 9.3.2 Material 实例匹配

两个 Image 使用相同的材质球（同一个 Material 实例拖到 Inspector 上）才能合批。如果分别复制出两个材质球，即使参数完全一样，也**算作不同材质实例**，会打断 Batch。

```csharp
// ❌ 即使 Shader 和参数完全一样，两个 new Material 也是不同的实例
ImageA.material = new Material(Shader.Find("UI/Default"));
ImageB.material = new Material(Shader.Find("UI/Default"));

// ✅ 共享同一个 Material 实例
public Material sharedMat;
ImageA.material = sharedMat;
ImageB.material = sharedMat;
```

这也是为什么不要轻易对 `Graphic.material` 赋值——每次赋值都可能创建新的 Material Instance，直接打断与其它元素的合批。**UI 默认使用的是 UGUI 内置的 shared material，不做特殊处理时所有同级元素自然共享同一材质实例，这是性能最好的状态。**

### 9.3.3 Texture 匹配

Texture 必须完全一致。同一张图集里的不同 Sprite 之所以能合批，就是因为它们指向的是同一张 Atlas Texture。

```csharp
// ImageA 和 ImageB 使用同一张图集中的不同 Sprite
ImageA.sprite = atlas.GetSprite("icon_1");  // sprite.texture → Atlas Texture
ImageB.sprite = atlas.GetSprite("icon_2");  // sprite.texture → 同一张 Atlas Texture
// ✅ 可以合批 —— texture 是同一个

// 如果两个 Sprite 来自不同的 Texture
ImageA.sprite = spriteA;  // textureA（2048x2048）
ImageB.sprite = spriteB;  // textureB（另一个 2048x2048）
// ❌ 无法合批 —— texture 不同
```

将多个小纹理打包到一张图集中，是 UI 合批最基础的优化手段。详见第14章。

### 9.3.4 Shader 与 Keywords 匹配

即使使用同一个 Shader 文件，如果通过 Material.EnableKeyword 开启了不同的 Keyword，也会被视为不同的 Shader Variant，从而打断合批。

```csharp
// Shader 相同但 Keywords 不同 → 打断合批
MatA.EnableKeyword("UI_GRAY");
MatB.DisableKeyword("UI_GRAY");
```

### 9.3.5 Additional Shader Channels

Canvas 的顶点布局由 `Canvas.additionalShaderChannels` 统一声明（Position / Normal / Tangent / Color / TexCoord0~3），同一 Canvas 下的所有元素共用这一套布局。main 的 `VertexHelper.FillMesh` 固定按 9 通道写入（见第 2 章），引擎 Batch 时按 Canvas 的通道配置读取数据。

默认配置（Position + Color + TexCoord0）已满足大多数 UI 元素，不会开启额外通道。一旦某个元素需要 Normal / Tangent / UV2 等额外数据（如 TMP 的 SDF 参数、自定义 Shader），就需要相应开启 `additionalShaderChannels`，否则数据不会传递到 Shader。通道配置影响"数据是否送达 Shader"，不直接决定能否合批——合批仍以材质、纹理、Shader 状态为准（行为依据：Unity 官方文档 `Canvas.additionalShaderChannels`；main 中不存在 `Graphic.GetModifiedMesh()` 方法）。

---

## 9.4 Batch 断裂原因

### 9.4.1 材质变化

材质变化是最常见的 Batch 断裂原因。在 Frame Debugger 中可以看到 `Break Batch Reason: Different Material`。

常见触发场景：
- 对某个 Image 单独设置了一个材质球
- 使用 Outline/Shadow 组件（它们会创建新的材质实例）
- Mask 组件改变了 Stencil 状态，生成了新的材质实例
- 对 `Graphic.material` 赋值，导致材质实例化

### 9.4.2 纹理切换

纹理切换打断 Batch，在 Frame Debugger 中显示为 `Break Batch Reason: Different Texture`。

- 不同 Image 引用了不同 Texture 的 Sprite（未打图集）
- RawImage 直接使用了不同的 Texture
- Sprite 来自不同图集

**解决方案**：将常用小图打入同一张图集。详见第14章。

### 9.4.3 Mask / Stencil 状态变化

当 Canvas 中存在 Mask 组件时，Mask 内部的 UI 元素会被分配不同的 Stencil 操作状态。UGUI 通过 `StencilMaterial.Add()` 为每个 Stencil 状态创建对应的材质副本。这些材质副本具有不同的 Stencil Ref / Stencil Comp 参数，因此与外部未受 Mask 影响的元素无法合批。

```
没有 Mask 的 UI（Stencil Ref=0, Comp=Always）   ← Batch A
Mask 外侧的 UI（Stencil Ref=0, Comp=Always）    ← 与 Batch A 相同
Mask 内侧的 UI（Stencil Ref=1, Comp=Equal）     ← Batch B（Different Stencil）
Mask 内侧嵌套 Mask 的 UI（Stencil Ref=2, Comp=Equal）← Batch C
Mask 外侧的 UI（Stencil Ref=0, Comp=Always）    ← 回到 Batch A（如果材质相同）
```

**每个 Mask 至少打断一次 Batch**——嵌套 Mask 则打断更多。Mask 是 UI 系统中 DrawCall 数量的主要推手之一。详见第15章。

### 9.4.4 RectMask2D 裁剪状态变化

RectMask2D 的裁剪方式与 Mask 完全不同：它**既不用 Stencil，也不修改顶点**。`RectMask2D` 通过 `IClippable.SetClipRect()` 通知子 Graphic，后者调用 `canvasRenderer.EnableRectClipping(rect)`；GPU 侧由 UI Shader 的 `UNITY_UI_CLIP_RECT` 关键字 + `_ClipRect` 在片元阶段裁剪（见第 15 章 15.2、第 18 章）。完全落在裁剪区外的元素则走 `Cull()` → `canvasRenderer.cull = true`，直接不提交。

不创建材质实例，但**裁剪矩形和关键字开关本身是渲染状态**：即便两个 UI 元素使用相同的材质和纹理，如果分别位于两个不同的 RectMask2D 下（裁剪矩形不同），也无法合并到同一个 DrawCall。同一个 RectMask2D 下的元素之间则可以正常合批——这是它与 Mask 的关键差异。

### 9.4.5 跨 Canvas

不同 Canvas 之间的 UI 元素永远无法合并为同一个 DrawCall，因为每个 Canvas 独立执行自己的 `BuildBatch()`。这也是为什么**过度拆分 Canvas 会降低合批效率**——拆分出的每个 Canvas 都有独立的 Batch 范围，原本可以合并的 UI 元素因为分属不同 Canvas 而无法合批。

### 9.4.6 Batch 断裂汇总

| 断裂原因 | Frame Debugger 显示 | 常见来源 |
|----------|-------------------|---------|
| 材质不同 | `Different Material` | 独立材质球、动态材质赋值 |
| 纹理不同 | `Different Texture` | 未打图集、跨图集引用 |
| Shader 不同 | `Different Shader` | 使用不同 UI Shader |
| Stencil 不同 | `Different Stencil State` | Mask、嵌套 Mask |
| 裁剪区域不同 | `Different RectMask2D` | 多个独立 RectMask2D |
| 跨 Canvas | `Different Canvas` | 子 Canvas 拆分 |

---

## 9.5 DrawCall 的本质

### 9.5.1 DrawCall 不只是一个数字

DrawCall 常常被简化为"帧率杀手"，但它的本质是：

> **DrawCall 是 CPU 向 GPU 提交的一次完整渲染命令，包含所有渲染状态的完整描述。**

一次 DrawCall 包含：

```
DrawCall = {
    Vertex Buffer  → 顶点数据（位置、UV、颜色）
    Index Buffer   → 三角形索引
    Shader         → Vertex Shader + Fragment Shader
    Material       → Blend / ZWrite / Cull / Stencil 等 RenderState
    Texture        → 采样纹理
    Uniform 参数   → 颜色、变换矩阵等 Shader 参数
}
```

GPU 收到 DrawCall 后，才能开始执行顶点处理和像素渲染。**DrawCall 本身不是 GPU 瓶颈，CPU 端提交 DrawCall 的开销才是瓶颈。** 尤其是在移动设备（iOS/Android）上，图形 API 驱动的 CPU 开销远高于 PC，DrawCall 数量对性能的影响更加显著。

### 9.5.2 DrawCall 与 Batch 的关系

```
Batch（批处理） → 合并 Mesh → 一个合并后的 DrawCall

Batch #1: [Image_1 + Image_2 + Image_3] → DrawCall #1（合并 Mesh）
Batch #2: [Image_4 + Image_5]           → DrawCall #2（合并 Mesh）
Batch #3: [Text_1 + Text_2]             → DrawCall #3（合并 Mesh）
```

最终提交给 GPU 的不是每个 UI 元素各自的 DrawCall，而是每个 Batch 对应的合并 DrawCall。因此 **"DrawCall 数量 = Batch 数量"**，而不是 UI 元素数量。这也解释了为什么 UI 数量多不一定卡，而 DrawCall 高一定卡。

### 9.5.3 为什么说"UI 数量多不卡，DrawCall 高才卡"

假设一个界面有 200 个 Image：

- **场景 A（未合批）**：200 个不同材质/纹理的 Image → 200 个 DrawCall → CPU 提交开销 200 次 → 大概率卡顿
- **场景 B（已合批）**：200 个共享同一材质、同一图集的 Image → 合并为 1~3 个 Batch → 1~3 个 DrawCall → CPU 提交开销极低 → 流畅运行

同样的 200 个 UI 元素，性能表现天差地别。**问题从来不在"UI 元素的数量"，而在"DrawCall 的数量"。**

UGUI 内部处理大 Mesh 的效率较高（合并后的 Mesh 即使包含数千个顶点，也只是一个 DrawCall）。因此，真正影响 UI 渲染性能的，是合批条件是否被满足。

---

## 9.6 批处理与图集的关联

> 详细内容请参考第14章"UI 资源与图集系统"，此处仅做简要关联。

### 9.6.1 图集对合批的意义

合批条件要求在同一个 DrawCall 中的所有 UI 元素使用**相同纹理**。如果每个 UI 元素引用各自的小纹理，就无法合批。图集的作用就是把多个小纹理打包到一张大纹理中：

```
未打图集 → Image A 引用 texture_a，Image B 引用 texture_b
         → 纹理不同 → Batch 断裂 → 2 个 DrawCall

已打图集 → Image A 引用 atlas_tex 的某个区域，
           Image B 引用 atlas_tex 的另一个区域
         → 纹理相同（都是 atlas_tex）→ 可以合批 → 1 个 DrawCall
```

### 9.6.2 图集使用的注意事项

- **图集分组策略**：同一界面频繁出现的 Sprite 应放在同一图集
- **图集大小上限**：过大图集浪费显存，过小图集导致跨图集引用打断合批
- **运行时替换**：SpriteAtlas 打包后，`sprite.texture` 自动替换为 Atlas Texture，代码不需要关心 Texture 切换
- **动态图集**：Unity 的动态图集（Dynamic Atlas）可以在运行时将未打图集的小纹理自动合并，但有内存和性能代价

---

## 9.7 批处理与性能

> 详细内容请参考第20章"UI 性能分析"，此处仅做简要关联。

### 9.7.1 Batch 中断是 UI 性能问题的第一根源

UI 性能问题可以归纳为五个维度（详见第20章），其中"Batch 中断"排在第一位。一次 Batch 中断就意味着一个新的 DrawCall。

### 9.7.2 Frame Debugger 诊断

使用 Frame Debugger 是定位 Batch 中断的最直观手段：

1. 打开 Window → Analysis → Frame Debugger
2. 点击 Enable 开始录制
3. 在事件列表中找到 Canvas 相关的 DrawCall
4. 选中条目，查看右侧的 "Break Batch Reason"

常见的显示信息：
- `Different Material` — 材质不同
- `Different Texture` — 纹理不同
- `Different Stencil State` — Stencil 状态不同
- `Different RectMask2D` — 裁剪区域不同
- `Different Shader` — Shader 不同

### 9.7.3 降低 DrawCall 的实践原则

- **合理使用图集**：将同界面 Sprite 打包到同一图集
- **共享材质实例**：避免对 Graphic.material 赋值，优先使用默认 shared material
- **减少 Mask 使用层级**：每个 Mask 至少打断一次 Batch
- **合并相同 RectMask2D**：而非为每个容器创建独立的 RectMask2D
- **控制 Canvas 数量**：不要将简单 UI 拆分到多个 Canvas，除非需要动静分离（一个元素频繁变化不至于触发整个 Canvas 重建）
- **优先使用简单 Image Type**：Sliced / Tiled / Filled 不影响合批，但某些自定义 Shader 可能打断

---

## 9.8 常见误区

### 9.8.1 "DrawCall 越低越好"

DrawCall 降低到极致（追求 0~1 个 DrawCall）不一定是最优解。不同的材质、不同的 Shader 效果本身就是 UI 设计的需求——一个带特效的 UI 可能天然就需要多个 DrawCall。**关键不是"追求最低 DrawCall"，而是"消除无意义的 Batch 中断"**。

### 9.8.2 "多个 Canvas 会让 DrawCall 翻倍"

跨 Canvas 确实无法合批，但多个 Canvas 并不一定导致 DrawCall 翻倍。如果不同 Canvas 中的 UI 已经属于不同材质/不同纹理，它们本来也无法合批，拆分与否没有区别。**拆分 Canvas 的主要代价不是 DrawCall 翻倍，而是每个 Canvas 独立执行 BuildBatch 的 CPU 开销。**

### 9.8.3 "图集一定能减少 DrawCall"

图集只能合并"使用该图集中 Sprite 的 Image"的 DrawCall。如果两个 Sprite 虽然在同一个图集，但两个 Image 使用了不同的材质（比如一个加了特效材质），它们仍然无法合批。**图集解决的是 Texture 层面的匹配问题，但材质层面的匹配同样重要。**

### 9.8.4 "合并所有 UI 到一个 Canvas 最好"

过度集中的 Canvas 也有问题——当 Canvas 中任意一个元素发生变化时，整个 Canvas 可能需要重新执行 BuildBatch。如果一个 Canvas 中混合了大量静态背景元素和频繁更新的动态元素，每次动态元素变化都会触发全量重建。**合理的做法是"动静分离"**：静态 UI 放在一个 Canvas，动态 UI 放在另一个 Canvas。

---

### 推荐的源码阅读路径

```
合批规则在引擎 Native 层（Canvas.BuildBatch），C# 侧无源码；
UI 侧对照：Graphic.materialForRendering / GetModifiedMaterial、StencilMaterial.cs、Mask.cs、RectMask2D.cs。
验证工具：Frame Debugger 逐 DrawCall 查看断批原因。
```

---

## 9.9 本章总结

### 核心要点

1. **批处理是 UGUI 自动执行的性能优化机制**：将共享相同渲染状态的 UI 元素合并为同一个 DrawCall，减少 GPU 状态切换

2. **合批条件是精确匹配**：相同 Material 实例 + 相同 Texture + 相同 Shader(+Keywords) + 相同 Stencil 状态 + 相同 RectMask2D + 相同 Canvas

3. **Batch 断裂原因**：材质变化、纹理切换、Mask/Stencil 变化、RectMask2D 裁剪状态变化、跨 Canvas——这些都能在 Frame Debugger 中直接看到

4. **DrawCall 的本质**：一次完整的 GPU 渲染状态切换。CPU 端提交 DrawCall 的开销是性能瓶颈，而非 GPU 执行本身

5. **UI 数量多不是问题，DrawCall 高才是问题**：200 个共享材质纹理的 Image 可以合并到极少 DrawCall，流畅运行

### 与其它章节的关联

| 关联章节 | 关联内容 |
|---------|---------|
| 第1章 整体架构 | Graphic → CanvasRenderer → Canvas 的渲染链路，是批处理的数据基础 |
| 第14章 图集系统 | 图集解决了 Texture 匹配问题，是合批的最基础优化手段 |
| 第15章 Mask 与裁剪 | Mask 通过 Stencil 打断 Batch，是 DrawCall 数量升高的主要推手之一 |
| 第20章 性能分析 | Batch 中断是 UI 性能问题的第一维度，Frame Debugger 是核心诊断工具 |

### 一句话概括

> **批处理是 UGUI 自动执行的 DrawCall 合并机制，其核心在于降低 CPU 向 GPU 提交渲染状态切换的开销——合批成功的关键是材质、纹理、Shader 三者的精确匹配。**

---

## 勘误汇总（对照 uGUI main 与官方文档）

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 | 9.3.5 | UGUI 通过 `Graphic.GetModifiedMesh()` 计算需要哪些顶点通道 | main 中不存在该方法；顶点布局由 `Canvas.additionalShaderChannels` 统一声明，`VertexHelper.FillMesh` 固定按 9 通道写入 |
| 2 | 🔴 | 9.4.4 | 「RectMask2D 通过**修改顶点位置**实现裁剪」 | 不改顶点：`SetClipRect()` → `canvasRenderer.EnableRectClipping()`，GPU 侧由 `UNITY_UI_CLIP_RECT` + `_ClipRect` 在片元阶段裁剪；完全在区外的元素走 `Cull()` 置 `canvasRenderer.cull`。断批结论本身正确，仅机制描述有误。全书统一口径见第 15 章 |
