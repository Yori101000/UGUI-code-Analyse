# 第17章 UI Shader 机制

> 本文是对原文第 17 章的结构化重写，合并了原文 17.1 与 17.2 中完全重复的 Shader 结构讲解（Transparent Queue、ZWrite Off、Blend、顶点颜色等在两节中重复解释），精简了 17.5（Stencil）与第 13 章重复的内容，将其聚焦在 Shader 代码层面如何实现 Stencil。对照 UGUI 源码修正了 5 处错误/不准确表述。

---

## 概述

前几章走完了 UGUI 的 CPU 渲染链路（顶点生成→Mesh 构建→Canvas Batch），本章补上最后一环——**GPU 端怎么把这些顶点变成屏幕上的像素**。

Shader 在 UGUI 中的定位：

```
第9、16章（Mesh 构建）                     第17章（Shader）
OnPopulateMesh → VertexHelper → Mesh → CanvasRenderer.SetMesh()
                                              ↓
                                         Material + Shader
                                              ↓
                                         GPU 渲染管线
                                    Vertex → Fragment → Blend → 屏幕
```

前几章关注的顶点、合批、Mask/Stencil——它们最终都通过 Shader 的输入数据、Stencil 参数、Blend 指令在 GPU 上执行。不理解 UI Shader，就缺了最后一块拼图。

UI Shader 和普通 3D Shader 的核心区别：

| 维度 | 3D Shader | UI Shader |
|------|----------|-----------|
| 核心任务 | 光照、阴影、PBR、空间深度 | 透明混合、Mask 裁剪、Canvas 合批适配 |
| 深度写入 | ✅ ZWrite On | ❌ ZWrite Off |
| 混合模式 | Opaque（覆盖）或特殊 Blend | 几乎总是 `Blend SrcAlpha OneMinusSrcAlpha` |
| 顶点数据 | POSITION+NORMAL+TANGENT+TEXCOORD0-7+BONES | POSITION+COLOR+TEXCOORD0（仅 3 个语义） |
| Vertex Shader | 骨骼蒙皮、法线变换、阴影坐标 | 仅 `UnityObjectToClipPos` + UV/Color 传递 |
| Fragment Shader | PBR、GI、Shadow、Reflection | `tex2D * IN.color` |
| 性能瓶颈 | 光照计算、阴影、顶点数 | Overdraw、Blend 状态切换、Mask 断批 |

---

## 17.1 Default UI Shader 完整解析

UGUI 中所有 Image、Text、RawImage，在未指定自定义材质时，最终都使用 `UI/Default` Shader。它不追求视觉效果——它追求的是**稳定适配整个 Canvas 渲染体系**。

直接看完整代码逐行拆解：

```glsl
Shader "UI/Default"
{
    Properties
    {
        [PerRendererData] _MainTex ("Sprite Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
    }

    SubShader
    {
        Tags
        {
            "Queue"="Transparent"
            "RenderType"="Transparent"
            "CanUseSpriteAtlas"="True"
        }

        Stencil
        {
            Ref [_Stencil]
            Comp [_StencilComp]
            Pass [_StencilOp]
            ReadMask [_StencilReadMask]
            WriteMask [_StencilWriteMask]
        }

        Cull Off
        Lighting Off
        ZWrite Off
        ZTest [unity_GUIZTestMode]
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityUI.cginc"

            struct appdata_t
            {
                float4 vertex   : POSITION;
                float4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 vertex   : SV_POSITION;
                fixed4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;

            v2f vert(appdata_t IN)
            {
                v2f OUT;
                OUT.vertex = UnityObjectToClipPos(IN.vertex);
                OUT.texcoord = TRANSFORM_TEX(IN.texcoord, _MainTex);
                OUT.color = IN.color * _Color;
                return OUT;
            }

            fixed4 frag(v2f IN) : SV_Target
            {
                fixed4 color = tex2D(_MainTex, IN.texcoord) * IN.color;
                return color;
            }
            ENDCG
        }
    }
}
```

### 逐行解析

**① `Queue=Transparent` 与 `RenderType=Transparent`**

UI 工作在透明渲染队列，在所有不透明物体之后渲染。因为 UI 需要支持半透明和层级叠加，不能走 Opaque 的深度排序路径。这个队列值也影响 Unity 的渲染排序行为。

**② `CanUseSpriteAtlas=True`**

告诉 SpriteAtlas 系统"这个 Shader 兼容图集"。缺少这个 Tag，图集系统在打包时可能不会把使用了该 Shader 的 Sprite 识别为可图集化的资源，从而影响图集合并。需要注意：这个 Tag 影响的是**编辑阶段的图集打包决策**，不是运行时的 DrawCall 直接控制。

**③ Stencil 块**

```glsl
Stencil
{
    Ref [_Stencil]                  ← Material 参数控制
    Comp [_StencilComp]             ← CompareFunction（Always/Equal 等）
    Pass [_StencilOp]               ← StencilOp（Replace/Keep 等）
    ReadMask [_StencilReadMask]
    WriteMask [_StencilWriteMask]
}
```

**关键点**：这些值不是硬编码在 Shader 里的。UGUI 的 Mask 组件通过 `StencilMaterial.Add()` 创建一个新的 Material 实例，把这些参数写入 Material 的 `[PerRendererData]` 属性。GPU 渲染时从 Material 读取这些值。同一个 Shader 可以同时服务于"被 Mask 遮罩的 UI"和"没被遮罩的 UI"——区别仅在于 Material 的 Stencil 参数不同。

这也解释了为什么 Mask 打断合批：**同一个 Shader 但不同的 Material 实例（Stencil 参数不同），就是不同的 GPU 渲染状态，不能合批。**（详见第 13 章 13.1.7）

**④ `Cull Off`**

不剔除任何面。UI 是 2D 平面，正反面都可能需要显示（如 UI 翻转时）。

**⑤ `ZWrite Off`**

**不写入深度缓冲。** UI 不依赖深度缓冲决定遮挡关系——遮挡由 Canvas 的 Hierarchy 顺序和 Sorting Order 决定。如果开启 ZWrite，一个半透明 UI 写入深度后，它后面的 UI 即使应该显示，也会被深度测试错误剔除。

这与第 9 章渲染管线中"UI 使用透明队列+层级排序"的说法一致。

**⑥ `ZTest [unity_GUIZTestMode]`**

默认 `LEqual`，但通过 Material 参数控制。这个属性主要用于处理 UI 与 3D 物体混合时的深度关系（如 World Space Canvas）。

**⑦ `Blend SrcAlpha OneMinusSrcAlpha`**

标准 Alpha 混合。公式：

```
最终颜色 = 当前像素颜色 × 当前 Alpha + 屏幕已有颜色 × (1 - 当前 Alpha)
```

UGUI 中所有透明能力（半透明 Image、字体抗锯齿边缘、Fade 动画）都依赖这一行。如果改成 `Blend Off`，字体边缘立刻出现锯齿。

**⑧ 顶点输入 `float4 color : COLOR`**

顶点颜色数据来自 `UIVertex.color`。它的来源链：
- `Graphic.color`（Inspector 中的 Image/Text 颜色）
- Text 每个字符的字体颜色
- Mesh Effect（Shadow/Outline）修改后的顶点颜色
- CanvasGroup 修改的 Alpha（统一控制子 UI 透明度）

所有颜色数据都通过顶点传递，不产生额外 DrawCall。

**⑨ Vertex Shader 中的坐标变换**

```glsl
OUT.vertex = UnityObjectToClipPos(IN.vertex);
OUT.texcoord = TRANSFORM_TEX(IN.texcoord, _MainTex);
OUT.color = IN.color * _Color;
```

UI 的 Vertex Shader 只做三件事：坐标变换、UV 传递、颜色传递。没有骨骼蒙皮、没有法线变换、没有阴影坐标计算——因为 UI 是 2D 平面，不需要这些。

`IN.color * _Color`：顶点颜色 × 材质 Tint。`_Color` 对应 Inspector 中 Image/Text 的 Color 属性。

**⑩ Fragment Shader 核心**

```glsl
fixed4 color = tex2D(_MainTex, IN.texcoord) * IN.color;
```

**UGUI 的 Fragment Shader 核心逻辑就这一行。** 采样贴图 → 乘上顶点颜色 → 输出。所有复杂的视觉效果（Mask 裁剪、描边、阴影、渐变）要么在 CPU 侧 ModifyMesh 中修改了顶点数据，要么在 Shader 的 Stencil/Blend 阶段处理。

---

## 17.2 顶点数据从 UIVertex 到 Shader 的完整传递

UGUI 在 CPU 侧构建的顶点数据如何到达 GPU Shader？这条链路连接了第 9~16 章的所有内容：

```
UIVertex（CPU 内存中构建）
  ├── position  → POSITION 语义
  ├── color     → COLOR 语义
  ├── uv0       → TEXCOORD0 语义（主纹理 UV）
  ├── uv1       → TEXCOORD1 语义（PositionAsUV1 等扩展用）
  ├── normal    → NORMAL 语义（TMP 的 SDF 计算需要）
  └── tangent   → TANGENT 语义（TMP 的 SDF 计算需要）
      ↓
VertexHelper.AddVert(position, color, uv)
      ↓
ModifyMesh 链修改 VertexHelper 中的顶点
      ↓
VertexHelper.FillMesh(mesh)
  → Mesh.vertices    = UIVertex.position
  → Mesh.colors32    = UIVertex.color
  → Mesh.uv          = UIVertex.uv0
      ↓
CanvasRenderer.SetMesh(mesh)
      ↓
GPU 接收语义化顶点数据
  appdata_t:
    float4 vertex   : POSITION;    ← 来自 Mesh.vertices
    fixed4 color    : COLOR;       ← 来自 Mesh.colors32
    float2 texcoord : TEXCOORD0;   ← 来自 Mesh.uv
      ↓
Vertex Shader 执行 → 输出裁剪空间坐标 + 传递 color/uv
      ↓
GPU 光栅化 → 对 Vertex Shader 输出进行插值
  （颜色插值、UV 插值——顶点间的值自动平滑过渡）
      ↓
Fragment Shader 接收插值后的像素级数据
  color = tex2D(_MainTex, IN.uv) * IN.color
      ↓
Stencil Test（如配置了 Stencil）
      ↓
Blend 阶段与屏幕已有颜色混合 → 写入帧缓冲
```

**关于 GPU 插值的关键概念**：GPU 在光栅化阶段会对 Vertex Shader 输出的数据进行**逐像素插值**。顶点 A 颜色为红（1,0,0）、顶点 B 颜色为蓝（0,0,1），三角形内部像素的颜色就是从红到蓝平滑过渡——这就是顶点渐变不需要额外 Shader 逻辑的原因，GPU 自动完成。

**额外通道 uv1/uv2/uv3 的用途**：

- **uv1**：`PositionAsUV1` 效果使用——把顶点位置写入 uv1，供 Shader 在 Fragment 阶段读取位置信息做效果（如扭曲、波浪、UV 动画）
- **normal/tangent**：TMP 的 SDF Shader 需要法线和切线来计算距离场的屏幕空间衍生（第 15 章）

---

## 17.3 Blend 与 Overdraw

### 17.3.1 UI 的 Alpha 来源链

UI 的最终 Alpha 不是来自单一系统，而是多层叠加：

```
最终 Alpha = 纹理 Alpha × Vertex Color Alpha × Material Tint Alpha × CanvasGroup.alpha
```

例如一张不透明的图片，放在 `color.a = 0.5f`、CanvasGroup.alpha = 0.5f 下：

```
最终 Alpha = 1.0（纹理不透明）× 0.5（Image.color.a）× 0.5（CanvasGroup.alpha）= 0.25
```

CanvasGroup 修改的就是 `UIVertex.color.a`，然后通过 Shader 中的 `IN.color` 传递到 Fragment Shader。所以 CanvasGroup 不产生额外 DrawCall——它只是修改了顶点颜色中的 Alpha。

### 17.3.2 三种常用 Blend 模式

| Blend 模式 | Shader 指令 | 计算公式 | 适用场景 |
|-----------|------------|---------|---------|
| 标准透明 | `SrcAlpha OneMinusSrcAlpha` | `Src×SrcA + Dst×(1-SrcA)` | 默认 UI、常规 Image/Text |
| Premultiplied Alpha | `One OneMinusSrcAlpha` | `Src + Dst×(1-SrcA)` | Spine 动画、粒子、高质量 UI（无黑边） |
| Additive（叠加） | `One One` | `Src + Dst` | 发光、流光、能量特效、高亮 |

Premultiplied Alpha 和标准 Alpha 的区别：

```
标准 Alpha：     RGB=(1,0,0), A=0.5 → 混合时颜色值 RGB×0.5 = (0.5,0,0)
Premultiplied：  RGB=(0.5,0,0), A=0.5 → 颜色已经乘过了，Blend 时直接相加
```

标准 Alpha 在半透明边缘可能出现黑边（因为 RGB × 0.5 后变暗，叠加到背景上颜色偏深）。
Premultiplied Alpha 不会出现这个问题，但纹理必须在导入时（或 Shader 中）预先乘好 Alpha。

Additive Blend 适合发光效果，但容易过曝——颜色不断累加，最终变白。

### 17.3.3 Overdraw：UI 最大的 GPU 成本

UI 的 GPU 压力主要来自 Blend，而不是 Shader 本身有多复杂。因为每绘制一个半透明像素，GPU 都要读取屏幕已有颜色做混合计算，而 UI 天然大量重叠。

一个典型弹窗的 Overdraw 情况：

```
┌──────────────────────────┐
│  全屏半透明背景（遮罩）  │ ← 全屏 Overdraw（覆盖整个画面）
│  ┌──────────────────┐  │
│  │ 弹窗背景         │  │ ← 第二次绘制（弹窗区域内）
│  │  ┌────┐ ┌────┐ │  │
│  │  │图标│ │文字│ │  │ ← 第三次/第四次（图标+文字区域）
│  │  └────┘ └────┘ │  │
│  └──────────────────┘  │
└──────────────────────────┘
```

同一像素可能被 Blend 混合了 4 次。这就是为什么 UI 性能优化不能只看 DrawCall，还要看 Overdraw。

**两个要注意的点：**

1. **Mask 和 RectMask2D 都不减少 Overdraw**（第 13 章已说明）——被裁剪的像素仍然执行完 Fragment Shader，只在 Blend/Stencil 阶段前被丢弃
2. **大量半透明 UI 重叠时，DrawCall 不高但 GPU 可能已经满载**——因为每个像素被反复混合

---

## 17.4 Stencil 在 Shader 中的实现

本节侧重 Shader 代码层面如何实现 Stencil，与第 13 章（Mask 的组件结构和传播机制）形成互补。Stencil Buffer 的基本概念（什么是 Stencil Buffer、写入+测试的两阶段机制）在第 13 章已详细讲过，这里不再重复。

### 17.4.1 Shader 中的 Stencil 块

Default UI Shader 中的 Stencil 块使用 Material 属性，而非硬编码值：

```glsl
Stencil
{
    Ref [_Stencil]                   ← 参考值（如 1、2、3）
    Comp [_StencilComp]              ← 比较函数（Always、Equal 等）
    Pass [_StencilOp]                ← 通过后的操作（Replace、Keep 等）
    ReadMask [_StencilReadMask]      ← 读取掩码
    WriteMask [_StencilWriteMask]    ← 写入掩码
}
```

这些 `[_XXX]` 方括号语法意味着值来自 Material 的同名属性。UGUI 的 `StencilMaterial.Add()` 创建 Material 实例时写入这些属性：

```csharp
// Mask 自身：写入 Stencil
StencilMaterial.Add(
    baseMaterial,
    stencilRef,              → _Stencil = ref
    StencilOp.Replace,       → _StencilOp = Replace（写入）
    CompareFunction.Always,  → _StencilComp = Always（始终通过）
    ColorWriteMask.All
);

// 子节点：测试 Stencil
StencilMaterial.Add(
    baseMaterial,
    stencilRef,
    StencilOp.Keep,          → _StencilOp = Keep（不修改）
    CompareFunction.Equal,   → _StencilComp = Equal（只有等于才通过）
    ColorWriteMask.All
);
```

**所以一个不涉及 Mask 的普通 UI 使用的是没有 Stencil 参数的默认 Material，而一个被 Mask 遮罩的 UI 使用的是带有 Stencil 参数的另一个 Material 实例。** 即使它们使用同一个 Shader，也因为 Material 实例不同而无法合批。

### 17.4.2 嵌套 Mask 的 Stencil 值计算

第 13 章已经解释了 `MaskUtilities.GetStencilDepth()` 遍历 transform.parent 统计 Mask 深度的机制。从 Shader 角度来看，不同深度对应不同的 `_Stencil` 值：

```
深度 0（无 Mask）：_Stencil 不设置，不参与 Stencil 测试
深度 1（1 层 Mask）：_Stencil = 1
深度 2（2 层嵌套）：_Stencil = 3  ← 使用 (1 << depth) - 1 位运算
```

UGUI 使用位掩码来编码 Stencil 深度：

```
depth=1 → ref = (1 << 1) - 1 = 1  → 二进制 00000001
depth=2 → ref = (1 << 2) - 1 = 3  → 二进制 00000011
depth=3 → ref = (1 << 3) - 1 = 7  → 二进制 00000111
```

子节点测试时通过 `Comp Equal` 检查 Stencil 值是否等于 ref——只有所有外层 Mask 覆盖的区域才满足条件。这也是嵌套 Mask 裁剪区域逐层缩小的本质。

### 17.4.3 Early Stencil Test 的补充说明

关于 Stencil Test 的执行位置：现代 GPU 支持 **Early Stencil Test（早期模板测试）**——在 Fragment Shader 执行之前进行 Stencil 测试。如果测试失败，Fragment Shader 根本不会执行，可节省 GPU 周期。

但 UGUI 的 Stencil 配置中如果使用了 `StencilOp.Replace` 等写入操作，GPU 可能无法使用 Early Stencil Test（因为 Fragment Shader 可能丢弃像素，写入操作需要确保只有最终通过的像素才写 Stencil）。具体行为取决于 GPU 架构和 Stencil 操作配置，不能一概说"Stencil Test 在 Fragment Shader 之后"。

---

## 17.5 自定义 UI Shader 快速参考

### 新建自定义 UI Shader 的模板

```glsl
Shader "UI/CustomEffect"
{
    Properties
    {
        [PerRendererData] _MainTex ("Sprite Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
        _EffectAmount ("Effect Amount", Range(0,1)) = 0.5
    }

    SubShader
    {
        Tags
        {
            "Queue"="Transparent"
            "RenderType"="Transparent"
            "CanUseSpriteAtlas"="True"     ← 必须保留，否则图集可能不打包
        }

        Cull Off
        Lighting Off
        ZWrite Off
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            struct appdata_t
            {
                float4 vertex   : POSITION;
                float4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 vertex   : SV_POSITION;
                fixed4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
            };

            sampler2D _MainTex;
            fixed4 _Color;
            float _EffectAmount;

            v2f vert(appdata_t IN)
            {
                v2f OUT;
                OUT.vertex = UnityObjectToClipPos(IN.vertex);
                OUT.texcoord = IN.texcoord;
                OUT.color = IN.color * _Color;
                return OUT;
            }

            fixed4 frag(v2f IN) : SV_Target
            {
                fixed4 color = tex2D(_MainTex, IN.texcoord) * IN.color;
                // 在这里添加自定义效果
                // 如颜色偏移、UV 扭曲、Alpha 调整等
                color.rgb = lerp(color.rgb, color.rgb * _EffectAmount, 0.5);
                return color;
            }
            ENDCG
        }
    }
}
```

**必须保留的配置**（否则 UGUI 渲染可能异常）：

| 配置 | 原因 |
|------|------|
| `CanUseSpriteAtlas=True` | 确保 SpriteAtlas 系统识别你的 Shader |
| `ZWrite Off` | UI 不写深度，否则遮挡关系错误 |
| `Blend SrcAlpha OneMinusSrcAlpha` | UI 需要透明混合 |
| `Cull Off` | UI 2D 平面可能需要双面可见 |

**顶点颜色乘法的保留**：`tex2D(...) * IN.color` 这行必须保留或用类似方式处理顶点颜色，否则 `Graphic.color`、`CanvasGroup` 的透明度调整、Text 字体颜色等功能全部失效。

### 使用额外 UV 通道的例子（PositionAsUV1 效果）

```glsl
// 在 appdata_t 中添加 uv1
struct appdata_t
{
    float4 vertex   : POSITION;
    float4 color    : COLOR;
    float2 texcoord : TEXCOORD0;
    float2 uv1      : TEXCOORD1;    ← 来自 UIVertex.uv1
};

// Fragment 中使用 uv1 做位置相关效果
fixed4 frag(v2f IN) : SV_Target
{
    fixed4 color = tex2D(_MainTex, IN.texcoord) * IN.color;
    // uv1 包含顶点位置信息 → 可以做基于位置的效果（波浪、扭曲等）
    float wave = sin(IN.uv1.y * 10 + _Time.y) * 0.02;
    color.rgb += wave;
    return color;
}
```

`PositionAsUV1` 是一个开启了 `additionalShaderChannels` 的选项（在 Graphic 组件上设置），它把顶点位置写入了 `UIVertex.uv1`，供自定义 Shader 读取做效果。

---

## 本章总结

```
             UGUI Shader 核心链路

UIVertex（CPU 侧构建）
  ├─ position  ─→ POSITION  → UnityObjectToClipPos → 裁剪空间坐标
  ├─ color     ─→ COLOR     → × _Color → IN.color（插值后到 Fragment）
  └─ uv0       ─→ TEXCOORD0 → tex2D    → 贴图采样

Fragment Shader 核心：return tex2D(_MainTex, IN.uv) * IN.color;
                ↓
     Stencil Test（由 Material 的 Stencil 参数控制）
                ↓
     Blend SrcAlpha OneMinusSrcAlpha（与屏幕已有颜色混合）
                ↓
     写入帧缓冲
```

**UI Shader 本质**：一套专门适配 Canvas 渲染体系的轻量 GPU 渲染程序。它不追求视觉效果（不涉及光照/PBR），追求的是**稳定、高效、适配合批**。

**本章与前几章的关系：**

| 章节 | 内容 | 在本章对应的 Shader 部分 |
|------|------|------------------------|
| 第9章 渲染管线 | Canvas.BuildBatch → GPU | Shader 的 Blend + ZWrite Off 决定 UI 如何参与透明队列渲染 |
| 第13章 Mask 裁剪 | Stencil Buffer 概念、Mask 组件机制 | 17.4 Stencil 在 Shader 中的实现（参数传递方式） |
| 第15章 Text | TMP 的 SDF 渲染 | TMP 使用 additionalShaderChannels 传递 normal/tangent |
| 第16章 Mesh 扩展 | ModifyMesh 修改顶点数据 | 修改后的 position/color/uv 在 shader 中体现效果 |

---

## 勘误汇总

| # | 严重程度 | 原文章节 | 原文声称 | 实际情况 |
|---|---------|---------|---------|---------|
| 1 | 🟢 轻微 | 17.2.8 | 缺少 `CanUseSpriteAtlas` Tag 会导致"图集合批失效、DrawCall 数量暴增" | 缺少该 Tag 时 SpriteAtlas 打包系统不打包使用该 Shader 的 Sprite，影响的是图集打包阶段而非直接导致运行时 DrawCall 暴增 |
| 2 | 🟢 轻微 | 17.5.5 | "Stencil Test 发生在 Fragment Shader 之后" | 现代 GPU 支持 Early Stencil Test（Fragment Shader 之前执行），失败则 Fragment Shader 不执行。写入操作（Replace）可能阻止 Early Stencil，但并非"总是在之后" |
| 3 | 🟢 轻微 | 17.1 / 17.2 | UI Shader 结构（17.1）与 Default UI Shader 解析（17.2）内容高度重复 | 两者在 Transparent Queue、ZWrite Off、Blend、顶点颜色、顶点输入结构、Fragment 核心逻辑上完全重复。17.2 只是用 Default UI Shader 作为示例把 17.1 又讲了一遍 |
| 4 | 🟢 轻微 | 17.5（Stencil 全节）与第 13 章 | Stencil Buffer 写入/测试/嵌套 Mask/位运算概念 | 内容与第 13 章几乎完全重叠。本章应侧重 Shader 代码层面的 Stencil 参数传递实现 |
| 5 | 🟢 轻微 | 17.6 全节 | UI vs 3D Shader 差异 11 个小节 | 每个差异点在前文均已隐含说明，属汇总性重复 |
