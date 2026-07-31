# 第27章 UGUI 与 URP/HDRP 渲染管线

> 本章分析 UGUI 在不同渲染管线（Built-in、URP、HDRP）下的行为差异，以及渲染管线对 UI 批处理、后处理、Camera Stack 的影响。

## 27.1 UGUI 在 Built-in 渲染管线中

Built-in 管线中 UGUI 的渲染链路最直接：

```
Graphic.OnPopulateMesh → CanvasRenderer.SetMesh
  → Canvas.BuildBatch()（native）
    → 提交 DrawCall 到 GPU
      → Vertex Shader → Fragment Shader → FrameBuffer
```

Overlay 模式的 UI 在所有 Camera 渲染完成后再绘制，不经过任何后处理。

## 27.2 UGUI 在 URP 中

### 27.2.1 URP 渲染流程中的 UI 位置

URP 使用可编程渲染管线（SRP），UI 的渲染由 `UniversalRenderPipeline.RenderSingleCamera` 中的特定 Pass 处理：

```
URP 帧渲染流程：
  1. ShadowMap Pass
  2. DepthPrePass（可选）
  3. Opaque Pass（不透明物体）
  4. Transparent Pass（透明物体）
  5. PostProcessing Pass（后处理）
  6. UI Overlay Pass  ← UI 在此处渲染
```

### 27.2.2 ScreenSpaceOverlay 模式

URP 中 Overlay UI 通过 `URP's RenderGraph` 的 `DrawOverlay` 步骤绘制。行为与 Built-in 一致：在后处理之后，覆盖全屏。

### 27.2.3 ScreenSpaceCamera 模式

UI 参与 Camera 的渲染流程，受该 Camera 的后处理影响。在 URP 中需注意：

```
Camera A（depth=0，带 Bloom 后处理）
  └── UI Canvas（Camera 模式，绑定 Camera A）
      → UI 被 Bloom 影响（不显眼但增加亮度）

Camera B（depth=1，不带后处理）
  └── UI Canvas（Camera 模式，绑定 Camera B）
      → UI 不受 Bloom 影响
```

### 27.2.4 URP 的 Render Objects Feature

URP 的 `Render Objects` 可以通过 Layer 过滤在特定时机额外渲染 UI。常见应用：
- 在 Opaque 和 Transparent 之间渲染 UI（实现"UI 被场景半透明物体遮挡"的效果）
- 特定 Camera 的 UI 叠加

### 27.2.5 URP 合批与 UGUI

URP 使用 SRP Batcher 优化材质属性提交，但 UGUI 的 Mesh 由 `Canvas.BuildBatch()` 独立管理：
- UGUI 内部合批：由 Canvas 的 `BuildBatch` 完成（材质/纹理/Stencil 相同则合并）
- SRP Batcher：不影响 UGUI 的 DrawCall 合并——UGUI 的 DrawCall 在进入管线渲染前已经合并完成

## 27.3 UGUI 在 HDRP 中

### 27.3.1 HDRP 的 UI 渲染

HDRP 渲染流程更复杂，UI 的插入位置：

```
HDRP 帧渲染流程：
  1. Shadow Pass
  2. Depth/PrePass
  3. GBuffer Pass（或 Forward Pass）
  4. Transparent Pass
  5. PostProcessing Pass（Bloom、ToneMapping 等）
  6. UI Overlay ← Overlay UI
```

### 27.3.2 HDRP Camera Stack

HDRP 使用 `HDCamera` 和 Camera Stack 管理多 Camera 渲染。UI 在 Camera Stack 中的行为：

```
Base Camera（渲染场景）
  └── Overlay Camera（渲染 UI）
      → UI 与场景叠加，受 Base Camera 的 ToneMapping 影响
```

### 27.3.3 HDRP 特殊问题

| 问题 | 原因 | 解决 |
|------|------|------|
| Overlay UI 闪烁 | HDRP 的 MSAA 与 UI 交互 | 降低 MSAA 或使用 Camera 模式 |
| 后处理影响 UI | UI 在 ToneMapping 前渲染 | 使用 Overlay 模式或在 PostProcess 之后渲染 |
| UI 写入深度 | UI Shader 默认 ZWrite Off | 自定义 UI Shader 按需开启 |
| HDR Color Buffer | HDRP 默认使用 HDR 缓冲区 | UI 颜色在校色前为 HDR，看起来过亮 |

## 27.4 UI Overlay 在不同管线的行为对比

| 行为 | Built-in | URP | HDRP |
|------|---------|-----|------|
| 渲染时机 | 所有 Camera 之后 | UI Overlay Pass | UI Overlay Pass |
| 受后处理影响 | 否 | 否 | 否（默认） |
| Camera Stack 支持 | 有限 | 完整 | 完整 |
| MSAA 兼容 | 是 | 是 | 需配置 |
| HDR 缓冲区 | 否 | 否（默认 LDR） | 是 |
| SRP Batcher | 不适用 | 不影响 UGUI | 不影响 UGUI |

## 27.5 实践建议

- **Game UI**：始终使用 Overlay 模式，不受后处理影响，性能最好
- **CG 过场 UI**：Camera 模式，与场景后处理风格统一
- **UI 受场景物体遮挡**：使用 WorldSpace 模式或 Camera 模式 + 深度测试
- **URP 项目**：注意 Render Objects Feature 可能意外渲染 UI
- **HDRP 项目**：HDRP 中 UI 颜色可能因 HDR 缓冲区而失真，需在校色后渲染
