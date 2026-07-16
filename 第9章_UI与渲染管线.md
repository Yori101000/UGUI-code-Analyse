# 第9章 UI 与渲染管线（修正版）

> 本文是对原文的精炼总结，并对照 UGUI 源码修正了原文中的 9 处错误。

---

## 核心渲染流程（原文此处有错误，已修正）

理解 UGUI 渲染，首先要搞清楚数据是怎么一步步走到 GPU 的。原文给出的流程有误，修正后的正确流程如下：

### 阶段一：顶点重建

当 UI 发生变化（比如 Image 换了张图、Text 改了内容），对应的 Graphic 会被标记为 dirty。下一帧，`CanvasUpdateRegistry.PerformUpdate()` 遍历所有 dirty 元素，调用 `Graphic.Rebuild()`：

```
Graphic.Rebuild()
├─ UpdateGeometry()
│    └─ OnPopulateMesh(VertexHelper vh)   // 子类重写此方法，填充顶点数据
│    └─ canvasRenderer.SetMesh(workMesh)   // 将顶点存入 CanvasRenderer
└─ UpdateMaterial()
     └─ canvasRenderer.SetMaterial(mat, tex) // 将材质/纹理也存入 CanvasRenderer
```

这里的关键是：**CanvasRenderer 此时是"数据容器"**，通过 `SetMesh` / `SetMaterial` / `SetTexture` 把渲染所需数据暂存起来，并不是在这里提交 DrawCall。

### 阶段二：批处理与提交

此后，Unity 引擎的渲染循环会调用 `Canvas.BuildBatch()`（这是一个 native 方法，C# 侧看不到实现）：

```
Canvas.BuildBatch()
├─ 遍历当前 Canvas 下所有 CanvasRenderer
├─ 从每个 CanvasRenderer 中读取已存储的 Mesh / Material / Texture
├─ 将材质和纹理相同的 Mesh 合并为一个大的合并 Mesh
└─ 将合并后的 Mesh + DrawCall 提交到 GPU Command Buffer
```

**DrawCall 的真正提交发生在 Canvas.BuildBatch() 内部，不是 CanvasRenderer。**

### 阶段三：GPU 执行

GPU Command Buffer 中的 DrawCall 被 RenderPipeline 调度，送入 GPU 执行：

```
Vertex Shader → 裁剪 → 光栅化 → Fragment Shader → Alpha Blend → FrameBuffer
```

### 修正后的完整流程

```
Graphic.OnPopulateMesh()      →  生成顶点数据
CanvasRenderer.SetMesh()      →  暂存数据（数据容器）
Canvas.BuildBatch()           →  读取数据、合并Mesh、提交DrawCall
RenderPipeline                →  调度执行
GPU                           →  像素输出
```

### 三个核心组件的关系

| 组件 | 角色 | 关键方法 |
|------|------|---------|
| **Graphic** | UI 渲染组件的抽象基类 | `OnPopulateMesh()` 生成顶点/UV/颜色/索引 |
| **CanvasRenderer** | 数据容器（桥接层） | `SetMesh()` / `SetMaterial()` / `SetTexture()` |
| **Canvas** | 批处理器 + 提交者 | `BuildBatch()` 合并 Mesh，提交 DrawCall |

> ⚠ **勘误 #1**：原文流程为"Graphic → Canvas BuildBatch → CanvasRenderer 提交 → RenderPipeline → GPU"。错误有二：一是 CanvasRenderer 放在了 Canvas 之后，二是将 CanvasRenderer 描述为"提交者"。实际顺序是先存数据到 CanvasRenderer，Canvas.BuildBatch 再从中读取并提交。

---

## 9.1 Built-in Pipeline 渲染流程

### 9.1.1 UI 在 Built-in RP 中的渲染位置

Built-in RP 是 Unity 早期的固定渲染管线。一帧的渲染顺序大致如下：

```
Camera 开始渲染
→ 场景剔除（Culling）
→ Shadow Pass（阴影贴图）
→ 不透明物体渲染（Geometry）
→ Skybox 渲染
→ 透明物体渲染（Transparent）
→ Image Effect（后处理）
→ UI 渲染（Screen Space - Overlay）
→ FrameBuffer 最终输出
```

**Screen Space - Overlay 模式的 UI 在所有 Camera 都完成渲染（包括后处理）之后才绘制**。这就是为什么 Overlay UI 永远出现在场景最上层，且不会受到 Bloom、Color Grading 等后处理效果的影响。

但这不意味着 UI "绕过了渲染管线"——UI 照样生成 Mesh、使用 Shader、提交 DrawCall、由 GPU 执行。只不过它的绘制被安排在管线的最末尾而已。

### 9.1.2 底层渲染流程

三个核心角色的详细职责：

**Graphic**：所有 UI 渲染组件的抽象基类（Image、Text、RawImage 等），核心方法是 `OnPopulateMesh(VertexHelper vh)`——子类重写此方法，向 VertexHelper 中填充顶点坐标、UV、顶点颜色、三角形索引。这些数据最终组成 UI Mesh。

**CanvasRenderer**：位于 UGUI 与底层渲染系统之间的桥接层。Graphic 生成的数据通过 `SetMesh()`、`SetMaterial()`、`SetTexture()` 存入其中。在数据存入阶段，CanvasRenderer 不执行任何渲染操作。

**Canvas**：负责批处理管理。`BuildBatch()` 会遍历当前 Canvas 下所有 CanvasRenderer，将使用相同材质和纹理的 Mesh 合并为大的合并 Mesh，然后生成 DrawCall 提交到 GPU Command Buffer。当 Canvas 下的 UI 发生变化时（元素被标记为 dirty），下一帧会重新执行 BuildBatch，这就是 UI Rebuild 开销的来源。

### 9.1.3 Overlay 模式

Overlay 是最常用的 UI 模式。核心特点：

- **不依赖任何 Camera**，不需要经过 Camera 的空间变换（顶点变换用的是简化正交投影）
- **不参与深度排序**，"层级"完全由 Canvas 的 Sorting Order 和 Hierarchy 中的顺序决定
- 在所有 Camera 渲染结束后统一绘制，因此稳定覆盖场景
- 不会被任何 Camera 后处理影响

### 9.1.4 Camera 模式与 World Space 模式

**Camera 模式**：将 UI 绑定到指定摄像机。UI 会经过该 Camera 的完整渲染流程，因此会受到 Camera Depth / Clear Flags / Post Processing / RenderTexture 等影响。**这就是为什么开启后处理后，Camera 模式的 UI 也会被 Bloom 影响，而 Overlay 模式不会。** 这是两种模式最关键的行为差异。

**World Space 模式**：Canvas 完全进入三维世界坐标体系，行为与普通 MeshRenderer 非常接近——参与深度测试（ZTest）、可被场景遮挡、受透视投影影响、参与三维空间的透明排序。

### 9.1.5 排序机制

Overlay Canvas 之间的排序由以下因素决定：
1. **Sorting Layer** + **Order in Layer**（Canvas 组件上的设置）
2. **Canvas 在 Hierarchy 中的顺序**（当 Sorting Order 相同时回退到此规则）

对于 Camera 模式和 World Space 模式，排序还会受到 Camera Depth 和 RenderQueue 的额外影响。

> ⚠ **勘误 #2**：原文将 Sorting Layer、Depth、Camera Depth、RenderQueue 不加区分地全部列为 Overlay UI 的排序因素。Overlay UI 没有关联的 Camera，Camera Depth 对它无意义。

### 9.1.6 Built-in RP 的局限

- UI 插入阶段固定（只能放在末尾），无法插到不透明和透明之间
- 无法精确控制 UI 与后处理的先后关系
- 难以通过代码干预 UI 所属的 RenderPass
- 渲染扩展能力较弱

---

## 9.2 URP 中的 UI 渲染

### 9.2.1 URP 与 Built-in 的本质区别

URP（Universal Render Pipeline）基于 SRP（Scriptable Render Pipeline），最大的变化是：**不再有写死的渲染顺序**。

URP 将渲染过程拆分为多个独立的 **RenderPass**，每个 Pass 有自己独立的执行阶段（RenderPassEvent）、RenderTarget、RenderState 和 Shader Pass。然后由一个 `ScriptableRenderer` 统一调度这些 Pass 的执行。UI 在 URP 中也被纳入这个 Pass 体系。

标准 URP Forward Renderer 的大致执行顺序：

```
SetupCameraProperties → Depth PrePass → Shadow Pass → Opaque → Skybox → Transparent → Post Processing → [UI Overlay] → Final Blit
```

Overlay UI 仍然在场景渲染完成后绘制，但关键在于：现在这个"最后阶段"本身也是一个明确的 RenderPass，可以通过 `RenderPassEvent` 调整它的位置。

> ⚠ **勘误 #3**：原文 9.2.1 的流程图将 Shadow Pass 排在 Depth PrePass 之前，且将 UI 渲染排在 Final Blit 之后。标准 URP Forward Renderer 中 Depth PrePass 在 Shadow Pass 之前，Overlay UI 在 Final Blit 之前。

### 9.2.2 URP 中 UI 的插入位置与扩展性

UI 作为 RenderPass 体系中的一环，打开了各种扩展可能：
- 在 UI 之前插入自定义特效 Pass
- 将 UI 渲染到一个 RenderTexture 而非直接到屏幕
- 通过 `ScriptableRendererFeature` 在透明阶段绘制特定的 UI
- 通过 `ScriptableRenderPass` 和 `RenderPassEvent` 精确控制渲染阶段

> ⚠ **勘误 #4**：原文称"UI 可以与 Post Processing 协同工作"。这仅适用于 Camera 模式的 UI。Overlay UI 在所有 Camera（包括后处理）之后渲染，不存在"协同"。

### 9.2.3 三种模式在 URP 中的行为

**Overlay 模式**：在所有 Camera（包括 Camera Stack 中的所有 Camera）完成渲染后才绘制。**不会被任何 Camera 的后处理影响。**

> ⚠ **勘误 #5**：原文 9.2.3 声称切换到 URP 后"某些后处理开始影响 UI"，将其归为 Overlay UI 的行为变化。这是严重错误——后处理影响 UI 只发生在 Camera 模式下，不适用于 Overlay。

**Camera 模式**：绑定到特定 Camera，参与该 Camera 的完整渲染链路——包括透明排序、后处理、RenderTexture 输出等。在 URP 中多 Camera 场景下行为尤其复杂（涉及 Base Camera 和 Overlay Camera 的叠加关系）。

**World Space 模式**：行为与普通透明 Mesh 几乎一致，进入透明物体 Pass，参与深度测试和透明排序，使用 Camera 矩阵进行投影。

### 9.2.4 RenderPass 的可编程性

在 URP 中，可以通过以下 API 主动控制 UI 相关的渲染：

```csharp
// 自定义 RenderPass
class MyUIPass : ScriptableRenderPass {
    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
        // 在此处操作 UI 渲染
    }
}

// 通过 RendererFeature 注入
class MyUIFeature : ScriptableRendererFeature {
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        renderer.EnqueuePass(myPass);
    }
}
```

现代 Unity UI 渲染的核心，已经不仅是 Canvas 系统本身，而是 Canvas 如何接入 SRP RenderPass。

---

## 9.3 Camera Stack 与 UI

### 9.3.1 什么是 Camera Stack

Camera Stack 是 URP 的摄像机叠加机制。将 Camera 分为两种角色：

- **Base Camera**：渲染链路的起点，负责创建 RenderTarget、清除颜色/深度缓冲、执行场景渲染、驱动后续 Overlay Camera
- **Overlay Camera**：叠加在 Base Camera 之上，不创建新的 RenderTarget，直接在 Base Camera 的输出上继续绘制

本质就是：**多个 Camera 共享同一个 FrameBuffer，按顺序依次叠加绘制**。

### 9.3.2 哪些 UI 参与 Camera Stack

- **Screen Space - Camera 模式 → 参与**
- **World Space 模式 → 参与**
- **Screen Space - Overlay 模式 → 不参与**（在 Stack 全部执行完后统一绘制）

这是一个重要的区分：Overlay UI 完全在 Camera Stack 体系之外。因此：
- Overlay UI 不会被 Camera 后处理影响
- Overlay UI 不参与 Camera Depth 排序
- Overlay UI 不属于 Camera Stack 的渲染链路

### 9.3.3 Base Camera 与 UI

当 UI 绑定到 Base Camera 时，渲染直接进入主渲染流程，受到 Clear Flags、Post Processing、HDR、MSAA、Render Scale 等参数影响。如果 Base Camera 开启了后处理，Camera UI 也会受到 Bloom、Color Grading 等效果影响。

### 9.3.4 Overlay Camera 与 UI

常见用法：专门创建一个 UI Camera 作为 Overlay Camera 叠加到主 Camera 上。

好处：
- 主 Camera 负责三维场景，UI Camera 只负责 UI/特效/HUD
- 场景后处理（Bloom 等）不会作用于 UI
- 可以给 UI Camera 配独立参数（独立 Culling Mask、独立 Clear Depth）

注意事项：Overlay Camera 不会重新清除颜色缓冲，如果 Clear Flags 或 Depth 配置不正确，容易导致 UI 残留、深度错误、UI 被前一层 Camera 遮挡。

### 9.3.5 Camera Stack 中的排序

最终显示结果由两层规则叠加决定：

**第一层：Camera 层**
- Camera Stack 中 Camera 的先后顺序
- 后执行的 Camera 的内容会覆盖先执行的 Camera

**第二层：Canvas 层**
- Canvas Sorting Order / Sorting Layer
- 但仅在同一个 Camera 内部有效

关键结论：**Camera 的叠加顺序优先级高于 Canvas Sorting Order**。例如，Camera A（先执行）上有一个 Sorting Order=100 的 UI，Camera B（后执行）上有一个 Sorting Order=0 的 UI——Camera B 上的 UI 会覆盖 Camera A 上的 UI，尽管它的 Sorting Order 更低。

### 9.3.6 Camera Stack 与后处理

后处理通常在 Camera 渲染结束阶段执行。UI 在后处理之前还是之后绘制直接影响最终视觉效果：
- UI 属于 Base Camera → 受 Bloom/Motion Blur/Color Grading 影响
- UI 位于单独 Overlay Camera → 可避免场景后处理作用于 UI

因此实际项目中经常采用"场景 Camera + UI Overlay Camera"的结构。

### 9.3.7 性能影响

每增加一个 Camera，都可能额外产生：Culling 开销、RenderPass 开销、RenderTarget 切换、GPU 状态切换、后处理开销。需要平衡 UI 分层结构、Camera Stack 数量、后处理复杂度与 GPU 提交成本。

---

## 9.4 RenderQueue 与排序

### 9.4.1 什么是 RenderQueue

每个材质都有一个 RenderQueue 值，Unity 根据它决定 DrawCall 的提交顺序。数值越小越早绘制。

| 队列 | 值 | 用于 |
|------|-----|------|
| Background | 1000 | 天空盒等背景 |
| Geometry | 2000 | 不透明几何体 |
| AlphaTest | 2450 | 遮挡剔除（如树叶挖洞） |
| Transparent | 3000 | 透明物体（包括 UI） |
| Overlay | 4000 | 覆盖层（镜头光晕等） |

UGUI 默认使用 **Transparent(3000)**，因为 UI 大量依赖 Alpha Blend。

### 9.4.2 UI 为什么在透明队列

不透明物体可以依赖深度缓冲实现正确遮挡：先写深度，后来的像素如果深度测试失败就直接丢弃（Early-Z 优化）。

透明物体不能这样——因为它需要和背后的颜色混合（Alpha Blend），所以：
- **ZWrite Off**（不写深度，否则会挡住后面的透明物体）
- **Blend SrcAlpha OneMinusSrcAlpha**（与已有颜色混合）
- **必须按从远到近的顺序绘制**才能得到正确的混合结果

这就意味着透明物体的绘制顺序至关重要——顺序错了，混合结果就错了。UGUI 的 Image、Text、RawImage 等全部需要透明混合，因此 UI Shader 会使用 Blend、关闭 ZWrite、进入 Transparent Queue。UI 在不透明物体之后绘制，这是它能覆盖场景的重要原因。

### 9.4.3 RenderQueue 对 UI 的影响

透明物体之间无法完全依赖深度缓冲进行正确遮挡，绘制顺序直接影响最终结果。在 UGUI 中，如果两个 UI 使用不同材质、不同 Shader 或不同 RenderQueue，排序结果可能完全不同。

> ⚠ **勘误 #6**：原文举例说"RenderQueue=3100 的 UI 即使 Sorting Order 更低也会后绘制"，并称"RenderQueue 优先级高于 UI 内部排序规则"。对于 Overlay Canvas，跨 Canvas 的排序由 Sorting Order 主导，RenderQueue 不能覆盖 Sorting Order。RenderQueue 主要影响同一 Canvas 内不同材质间的排序，以及 UI 与非 UI 透明物体的混合排序。

### 9.4.4 Canvas 排序与 RenderQueue 的关系

UGUI 的独立排序系统（Sorting Layer、Order in Layer、Override Sorting）主要作用于 Canvas 层级。实际运行时：

1. Unity 先根据 Sorting Layer → Order in Layer → Hierarchy 进行 Canvas 排序
2. 进入渲染阶段后，RenderQueue 再进一步影响 DrawCall 的提交顺序

**Canvas 排序不是最终排序。** 在 World Space UI、Camera UI、UI 与粒子/透明物体混合的场景中，RenderQueue 往往比 Canvas 层级更重要。

### 9.4.5 World Space UI 的排序困境

World Space UI 已完全进入三维透明体系，同时受 RenderQueue、Camera 距离、DrawCall 顺序三者共同影响。即使在 Canvas 系统内排序正确，也可能出现遮挡错误。

根本原因在于透明渲染本身不存在完美排序方案——GPU 只能依赖 RenderQueue、距离排序和 DrawCall 顺序来近似完成透明混合。因此 World Space UI 的排序复杂度远高于 Overlay UI。

### 9.4.6 UI Shader 与 RenderQueue

Shader 中的 `Tags { "Queue"="Transparent" }` 标签直接决定 RenderQueue。可通过 `material.renderQueue = 3100` 手动修改——但要小心，强制修改可能导致：

- Batch 被打断（因为同 Batch 要求相同 RenderQueue）
- Mask 行为异常（Mask 依赖 Stencil，Stencil 状态变化时本身就会创建新材质）
- Stencil 错误

UGUI 内部逻辑默认建立在 Transparent Queue 基础之上，不随意改 RenderQueue 是更安全的选择。

### 9.4.7 UI 与粒子的排序问题

粒子系统默认也属于 Transparent Queue(3000)，与 UI 处于同一透明体系。两者混合时，最终结果同时受 RenderQueue、Camera 距离、Sorting Layer、DrawCall 顺序影响。常见问题包括粒子覆盖 UI、UI 穿插粒子、半透明混合错误。

常见解决手段：
- 给 UI 单独一个 Overlay Camera，物理上分离渲染层
- 手动调整材质的 RenderQueue（UI 设为 3100 高于粒子默认的 3000）
- 用 RenderTexture 把粒子渲染到纹理上，再放到 UI 中显示
- 使用 Overlay Canvas（始终在所有 Camera 之后渲染）

本质上都是在重新控制透明渲染顺序。

### 9.4.8 RenderQueue 的本质

RenderQueue 决定的是"DrawCall 进入 GPU 的先后顺序"。而 GPU 对透明物体的最终结果高度依赖绘制顺序。UGUI 虽然是 UI 系统，但它最终属于 GPU 透明渲染体系的一部分。真正复杂的 UI 排序问题，最终都必须从 RenderPipeline 与 GPU 渲染顺序的角度分析。

---

## 9.5 SRP Batcher

### 9.5.1 SRP Batcher 做什么

SRP Batcher 是 SRP 体系下的 GPU 提交优化机制。核心目标：减少 CPU 向 GPU 提交 DrawCall 时的状态切换开销。

传统渲染中，每次提交 DrawCall 前 CPU 需要向 GPU 上传大量常量数据（UnityPerMaterial、UnityPerDraw、Shader Uniform 等）。SRP Batcher 的解决思路：将这些数据缓存在 GPU 的常量缓冲区（CBUFFER）中。如果连续的 DrawCall 使用相同的 Shader Variant 和兼容的 CBUFFER 布局，GPU 就可以复用已缓存的数据，不需要 CPU 重新上传。

**关键认知**：SRP Batcher 不减少 DrawCall 数量——它减少的是每次 DrawCall 的 CPU 提交开销（RenderThread 压力）。

### 9.5.2 UGUI 为什么难以受益

SRP Batcher 最适合的场景是：大量物体、相同 Shader、稳定的 Material。而 UGUI 恰恰相反：

- **Mesh 是动态合并的**：Canvas.BuildBatch 每次都重新生成合并 Mesh，不存在稳定的单物体 Mesh
- **材质状态频繁变化**：Mask 会改变 Stencil 状态，导致同一 Shader 产生不同材质实例
- **动态材质赋值**：`image.material = someMat` 会创建新的 Material Instance
- **顶点数据频繁变化**：Graphic 会动态修改 Mesh 数据

结论：即使 UI Shader 本身兼容 SRP Batcher，UGUI 也未必能获得明显收益。这不是 bug，是 UGUI 的工作方式决定的。

### 9.5.3 Shader 兼容性

一个 Shader 是否兼容 SRP Batcher，主要取决于 CBUFFER 是否符合 SRP 规范：

```hlsl
CBUFFER_START(UnityPerMaterial)
float4 _Color;
float4 _MainTex_ST;
CBUFFER_END
```

所有 Material 属性必须进入统一的 CBUFFER，否则 Shader 无法进入 SRP Batcher。

UGUI 中常见不兼容情况：
- 使用旧版 UI Shader（不是基于 SRP 架构设计）
- 使用第三方 UI 特效 Shader
- 使用 GrabPass
- 使用多 Pass Shader

在 Frame Debugger 中可以看到 "SRP Batcher：Not Compatible"。

### 9.5.4 Mask 和动态材质的影响

**Mask** 是 UGUI 中最容易破坏 SRP Batcher 的系统。Mask 通过修改 Stencil State（Stencil Ref / Stencil Compare）来实现裁剪，会导致同一 Shader 产生不同的材质实例，从而打断 Batch、增加 Material 数量、使 SRP Batcher 失效。

**动态材质**：修改 Material 属性、使用 Outline/Shadow 等 UI 特效插件，都会创建新的 Material Instance。一旦材质实例过多，即使 Shader 相同也无法有效复用 GPU 常量缓冲，SRP Batcher 收益迅速下降。

> ⚠ **勘误 #7**：原文将 `Graphic.materialForRendering`（只读 getter）列为导致材质实例化的原因。访问该属性不会创建新实例，真正导致实例化的是给 `Graphic.material` 赋值。

### 9.5.5 UGUI 真正的优化重点

对 UI 来说，最重要的优化手段永远是 **Canvas Batch 管理**：

- **合理拆分 Canvas**：变化频繁的 UI 放在独立 Canvas，避免一个元素变化导致整个 Canvas 重建
- **使用图集**：同一张图集的元素可以被合并到同一个 DrawCall
- **减少 Mask 使用**：每个 Mask 至少打断一次 Batch，复杂 UI 中 Mask 是 DrawCall 数量的主要推手
- **避免动态材质**：尽量共享 Material，减少材质实例数

即使 SRP Batcher 完全失效，UI 性能仍可能正常——因为 UI 的核心瓶颈通常不在 Shader 状态切换。

### 9.5.6 SRP Batcher 对 UI 的真实意义

SRP Batcher 对 UGUI 不是完全无意义，但它是额外优化项而非核心机制。UGUI 属于高动态渲染体系，与 SRP Batcher 擅长优化的"大量稳定 Mesh Renderer"场景不完全一致。

> ⚠ **勘误 #8**：原文第 9 章小结声称"SRP Batcher 使大量 UI 在频繁更新场景下具备更稳定性能表现"，与正文 9.5 节的分析直接矛盾。正文多个子节明确指出 UGUI 几乎无法有效利用 SRP Batcher。正确表述：SRP Batcher 对 UGUI 收益有限，UI 性能核心在于 Canvas Batch 与 Rebuild 控制。

---

## 9.6 UI 在 GPU 渲染阶段的位置

### 9.6.1 GPU 怎么看 UI

GPU 不认识 Image、Text、Button 这些概念。到它这里，UI 已经被转换为标准渲染数据：

- **Vertex Buffer**（顶点坐标 + UV + 颜色）
- **Index Buffer**（三角形索引）
- **Texture**（纹理采样）
- **Shader**（顶点/片元着色器）
- **RenderState**（Blend 模式、ZWrite 状态、Stencil 状态等）

所以从 GPU 视角看：**UI 就是一组需要做 Alpha Blend 的透明 Mesh 而已**，跟场景里的半透明玻璃、粒子特效没有本质区别。

### 9.6.2 GPU 管线中的处理流程

```
Vertex Shader:
  输入：顶点坐标、UV
  输出：裁剪空间坐标 + 插值后的 UV/颜色
  → Overlay 模式直接用屏幕空间坐标（简化的正交投影）
  → Camera/World Space 模式需经过 MVP 矩阵变换

Clipping（裁剪）：
  丢弃屏幕外的三角形

Rasterization（光栅化）：
  将三角形转换为屏幕上的像素片段（Fragment）

Fragment Shader：
  纹理采样 → Alpha 计算 → Mask 裁剪（Stencil 比较）→ 颜色计算

Blending（混合）：
  Blend SrcAlpha OneMinusSrcAlpha
  新颜色 × Alpha + 已有颜色 × (1 - Alpha)

FrameBuffer：
  最终像素写入帧缓冲
```

> ⚠ **勘误 #9**：原文在 Vertex Shader 阶段统一用"MVP 矩阵变换"来描述。这只适用于 Camera 和 World Space 模式。Overlay 模式的顶点已经是屏幕空间坐标，使用简化的正交投影，不经过传统 MVP 管线。

### 9.6.3 UI 为什么容易产生 Overdraw

UI 是 Overdraw 的重灾区。原因：

- **ZWrite Off**——不写深度，GPU 无法知道某个像素是否已经被覆盖过
- **透明混合**——不能像不透明物体那样利用 Early-Z 剔除被遮挡像素
- **层层叠加**——弹窗 + 遮罩 + 内容层，同一个像素可能被画了 3~5 遍

即使一个 UI 被完全遮挡、最终不可见，它的 Fragment Shader 仍然可能已经被执行了。在移动平台上，大量全屏透明 UI 叠加会迅速推高 Overdraw，让 GPU 成为瓶颈。

### 9.6.4 FillRate 问题

FillRate = GPU 每秒能填充的像素数。UI 对 FillRate 的消耗非常明显，因为：

- 大面积半透明区域（弹窗遮罩、全屏背景）
- 多层叠加，每一层都要执行 Fragment Shader
- 高分辨率屏幕下像素数翻倍

UI 的 GPU 开销大多数来自 **Fragment（像素处理）** 而非 Vertex（顶点处理）。一个全屏半透明 Image 即使纹理简单，GPU 也必须对每个像素执行 Fragment Shader。

### 9.6.5 Blend 阶段

`Blend SrcAlpha OneMinusSrcAlpha` 要求 GPU 先读取 FrameBuffer 中已有的颜色，再做混合计算，最后写回去。这个"读-算-写"的过程比不透明物体的简单覆盖要昂贵得多。因此：

- Blend 让 GPU 难以进行 Early-Z 优化
- 大量半透明/动态特效/模糊/发光 UI 会进一步提高 Blend 成本
- 透明 UI 的 GPU 成本通常高于不透明物体

### 9.6.6 深度缓冲

大多数 Overlay UI 默认 `ZWrite Off`，无法利用深度缓冲进行遮挡优化——这是 Overdraw 的根本原因之一。

World Space UI 有所不同：它可能参与 ZTest 和深度比较，可被场景物体遮挡。但大部分 UI Shader 依然关闭 ZWrite，因为 UI 的设计出发点就是透明混合而非深度写入。

### 9.6.7 DrawCall 的本质

UI 的每次 Batch 拆分（材质变化、Texture 变化、Mask 变化、Stencil 变化、Shader Keyword 变化）本质上都生成新的 DrawCall。GPU 不会区别对待这些 DrawCall——UI 的 DrawCall 和场景 Mesh 的 DrawCall 性质完全相同，都会切换 RenderState、Shader、Texture，向 GPU 提交命令。

因此复杂 UI 系统同样可以产生非常高的 GPU 开销。

### 9.6.8 UI 在 GPU 中的真实本质

UGUI 最终是：
- **一组透明 Mesh**（数据来源于 Canvas 系统而非 MeshFilter）
- **一组 DrawCall**
- **一组 Fragment Shader**
- **一组 Blend 操作**

与普通透明物体最大的区别仅在于数据来源。当数据进入 GPU 后，UI 与透明 MeshRenderer 的本质差异已经非常小。因此，真正理解 UGUI 渲染必须理解：DrawCall、RenderState、Blend、FillRate、Overdraw、GPU Pipeline——因为 **UI 的最终执行位置从来不在 Canvas，而在 GPU。**

---

## 本章总结

UI 并非独立运行的渲染系统，而是深度嵌入 Unity 渲染管线中的参与者。

| 层级 | 核心机制 | 性能关注点 |
|------|---------|-----------|
| CPU - 重建 | CanvasUpdateRegistry → Graphic.Rebuild → SetMesh | Rebuild 范围控制、Canvas 合理拆分 |
| CPU - 提交 | Canvas.BuildBatch → 合并 Mesh → DrawCall | 材质/纹理一致性（减少 Batch 打断）|
| CPU - 状态 | Material 切换、SRP Batcher | UI 很难受益于 SRP Batcher，重点在 Canvas Batch |
| GPU - 渲染 | Vertex Shader → Fragment Shader → Blend | Overdraw、FillRate、Blend 开销 |

### 三种模式的核心差异

| | Overlay | Camera | World Space |
|------|---------|--------|-------------|
| 依赖 Camera | 否 | 是 | 是 |
| 受后处理影响 | 否 | 是 | 是 |
| 参与深度排序 | 否 | 部分 | 是 |
| 参与 Camera Stack | 否 | 是 | 是 |

### 两代管线的架构对比

| 维度 | Built-in RP | URP |
|------|------------|-----|
| 结构 | 固定渲染流程，UI 作为末端插槽 | RenderPass 可编排体系，UI 成为可控节点 |
| 可扩展性 | 弱 | 强（RendererFeature / 自定义 RenderPass）|
| Camera 叠加 | 靠 Camera Depth 简单覆盖 | Base + Overlay 堆叠，支持分离渲染 |

### 关键结论

从 CPU 侧的批处理构建，到渲染管线的阶段调度，再到 GPU 端的像素输出，每一层机制共同决定了 UI 的最终性能与视觉结果。**UI 不是独立于渲染体系的特殊系统，而是 GPU 透明渲染体系中的一个普通参与者。** 理解这一点，是理解所有 UI 渲染问题（排序、性能、后处理交互）的基础。

---

## 勘误汇总

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🔴 严重 | 核心流程 | 流程：Graphic → Canvas BuildBatch → CanvasRenderer 提交 | 错误。数据先到 CanvasRenderer（SetMesh），Canvas.BuildBatch 再读取并提交 |
| 2 | 🔴 严重 | 9.2.3 | Overlay UI 在 URP 中会被后处理影响 | Overlay UI 在所有 Camera Pass 之后渲染，不受后处理影响 |
| 3 | 🔴 严重 | 本章小结 | SRP Batcher 使 UI 具备更稳定性能表现 | 与正文矛盾，UGUI 几乎无法有效利用 SRP Batcher |
| 4 | 🟡 中等 | 9.4.3 | RenderQueue 优先级高于 UI 内部排序规则，能覆盖 Sorting Order | 跨 Canvas 排序由 Sorting Order 主导，RenderQueue 不能覆盖 |
| 5 | 🟡 中等 | 9.1.5 | Sorting Layer、Depth 等被不加区分列为 Overlay UI 排序因素 | Depth 和 Camera Depth 对 Overlay UI 不适用 |
| 6 | 🟡 中等 | 9.2.2 | "UI 可与 Post Processing 协同工作" | 仅 Camera 模式适用，Overlay 不适用 |
| 7 | 🟢 轻微 | 9.2.1 | URP 流程：Shadow → Depth PrePass → … → Final Blit → UI | Depth PrePass 在 Shadow 前；UI 在 Final Blit 前 |
| 8 | 🟢 轻微 | 9.5.5 | "使用 MaterialForRendering"导致材质实例化 | materialForRendering 是只读 getter，访问不创建实例 |
| 9 | 🟢 轻微 | 9.6.2 | Overlay UI 顶点经过"MVP 矩阵变换" | Overlay 是屏幕空间坐标，使用正交投影，不经过传统 MVP |
