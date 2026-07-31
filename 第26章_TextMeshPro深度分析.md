# 第26章 TextMeshPro 深度分析

> TextMeshPro（TMP）是 Unity 官方文本解决方案，现已取代 Legacy Text 成为默认组件。TMP 与 UGUI 的 Legacy Text 在实现架构上有本质区别。

## 26.1 SDF 渲染原理

### 26.1.1 传统位图文字的局限

Legacy Text 使用字体图集（Font Atlas），每个字符存储为位图，放大时出现锯齿。

### 26.1.2 SDF（Signed Distance Field）

TMP 的核心技术是 SDF——每个字符不存储像素颜色，而存储**到轮廓边界的距离**：

```
SDF 纹理的像素值含义（以 8-bit 存储为例）：

  ┌─────┬─────┬─────┬─────┬─────┐
  │ -127 │ -64 │   0 │ +64 │ +127 │
  └─────┴─────┴─────┴─────┴─────┘
    外部    外部   边界   内部    内部
  （越负越远）（靠近边界）│（靠近边界）（越正越深）

正值 → 像素在字形内部
负值 → 像素在字形外部
0    → 恰好落在轮廓边界上
```

Shader 通过插值这些距离值，在任意缩放下都能恢复出清晰的轮廓边缘——这就是 TMP 即使放大数倍也不模糊的原因。

### 26.1.3 SDF 的计算

SDF 纹理生成在**导入字体资产时**完成，不在运行时执行：

```
TTF/OTF 字体文件
  → 提取 glyph 轮廓曲线（矢量路径）
  → 对每个像素计算到最近轮廓的距离
  → 写入 SDF 纹理（通常是 8-bit 或 16-bit 的 float 纹理）
```

SDF 参数通过 UIVertex.uv2 传递给 Shader。

## 26.2 TMP 的源码结构

TMP 在 UGUI 仓库中的路径：

```
com.unity.ugui/Runtime/TMP/
├── TMP_Text.cs                  ← 抽象基类，类似 Graphic 的地位
├── TextMeshProUGUI.cs           ← UGUI 组件（挂载在 GameObject 上）
├── TMP_FontAsset.cs             ← 字体资产
├── TMP_SpriteAsset.cs           ← Sprite 图集（Emoji 支持）
├── TMP_TextInfo.cs              ← 文本布局信息
├── TMP_MeshInfo.cs              ← Mesh 数据结构
├── TMP_CharacterInfo.cs         ← 字符信息
├── TMP_LineInfo.cs              ← 行信息
├── MaterialManager.cs           ← 材质管理
├── ShaderUtilities.cs           ← Shader 工具
└── TMP_Settings.cs              ← 全局设置
```

### 26.2.1 继承关系

```
MonoBehaviour
  └── TMP_Text（实现 ICanvasElement、IMaterialModifier 等）
       └── TextMeshProUGUI（UGUI 集成）
       └── TextMeshPro（World Space 3D 文本）
```

TMP 不继承 `Graphic`——它是一个独立的文本系统，但实现了 `ICanvasElement` 接口，因此同样走 `CanvasUpdateRegistry` 的调度流程。

## 26.3 字符缓存系统

TMP 的核心性能优化在于**字符缓存**：

```
TMP_TextInfo
├── characterInfo[]      ← 每个字符的排版信息（索引、位置、尺寸、UV）
├── meshInfo[]           ← Mesh 数据（顶点、UV、三角形）
└── lineInfo[]           ← 行信息（行高、基线偏移等）
```

### 26.3.1 增量更新

当文本内容变化时，TMP 不是全量重建，而是复用已有字符缓存：

```csharp
// TMP_Text.cs（简化）
public void SetText(string text)
{
    // 1. 解析富文本标签
    // 2. 复用现有字符缓存，只重新排布变化的部分
    // 3. 更新 TMP_TextInfo
    // 4. 标记顶点 dirty，触发 Mesh 重建
}
```

这与 Legacy Text 的每帧 `TextGenerator.Populate()` 形成对比——TMP 的排版信息缓存避免了重复的字体加载和字形定位。

### 26.3.2 多 Atlas 纹理

TMP 支持**多 Atlas 纹理**——当字符集超过单个 Atlas 容量时自动创建新纹理：

```
Font Asset
├── atlasTextures[0]  ← 主 Atlas（ASCII + 常用字符）
├── atlasTextures[1]  ← 扩展 Atlas（生僻字）
└── atlasTextures[2]  ← 扩展 Atlas（Emoji）
```

这会导致跨 Atlas 的字符无法合批——同一段文字中 ASCII 字符和 Emoji 各产生一个 DrawCall。

## 26.4 材质系统

TMP 使用多种预设材质：

| 材质类型 | 效果 | 顶点倍数 |
|---------|------|---------|
| `TMP_SDF-Mobile` | 基础 SDF | 1× |
| `TMP_SDF-Mobile Overlay` | 叠加 SDF | 1× |
| `TMP_SDF-Mobile Masking` | 带 Mask 支持 | 1× |
| `TMP_Bitmap-Mobile` | 回退到位图 | 1× |
| 自定义 Outline | SDF + 描边 | ~2× |
| 自定义 Glow | SDF + 发光 | ~2× |

TMP 支持**5 层材质**（Face、Outline、Underlay、Overlay、Glow），启用越多层每字符生成顶点数越多。

## 26.5 TMP vs Legacy Text

| 维度 | Legacy Text | TextMeshPro |
|------|------------|-------------|
| 渲染方式 | 位图字体图集 | SDF |
| 缩放质量 | 放大锯齿 | 平滑 |
| 源码位置 | `UI/Core/Text.cs` | `Runtime/TMP/TMP_Text.cs` |
| 基类 | `MaskableGraphic` | `TMP_Text`（独立体系） |
| 字符缓存 | 无，每帧 `TextGenerator.Populate()` | `TMP_TextInfo` 缓存 |
| 富文本 | 有限支持 | 完整支持 |
| 字体资产 | `Font` | `TMP_FontAsset` |
| 批处理条件 | 同材质同纹理 | 同 Atlas 纹理（跨 Atlas 断批） |
| UV 通道 | uv0 | uv0 + uv2（SDF 参数） |
| Canvas 设置 | None | TexCoord2 |

## 26.6 性能要点

- TMP 的 `SetText` 在字符数不变时不触发全量排版——这是相比 Legacy Text 的最大优势
- 多 Atlas 会导致 DrawCall 增加——尽量使用单 Atlas 覆盖全部字符
- 每层材质（Outline、Underlay 等）增加顶点数——不必要时不启用
- TMP 的 `materialForRendering` 支持 `IMaterialModifier` 链（与 UGUI 一致）
