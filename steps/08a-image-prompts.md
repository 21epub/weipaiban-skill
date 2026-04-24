# Step 8a：图片提示词规划

根据 Step 7 的分类结果，为每张需要替换的图片构造**分类专属的生成 Prompt**。本步骤是纯推理步骤，不调用即梦 API。

## 前置读取

从任务目录读取以下文件：

1. **`meta.json`**：获取 `theme`（用户主题）、`taskDir`
2. **`image-classifications.json`**：每张图片的 `category`、`visualDescription`、`toneAlignment`、`needsTransparent`、`width`、`height`、`ratio`、`skip` 等
3. **`template-profile.json`**：仅读取 `original_palette`（saturation/brightness/contrast/temperature）和 `image_plan`
4. **`color-changes.json`**：仅读取 `palette` 字段（主色调方案，用于 prompt 中引用配色）

## 前置依赖检查

图片生成流程依赖以下工具，在继续之前必须先确认可用：

### 1. 即梦技能（jimeng-ai）

- 检查即梦技能是否已安装，如果未安装，通过 clawhub 或 find-skill 技能找到 jimeng-ai 技能并安装
- 确认 `JIMENG_ACCESS_KEY` 和 `JIMENG_SECRET_KEY` 环境变量已配置

### 2. Python 与 rembg

- 确认 `python3` 可用：`python3 --version`
- 确认 rembg CLI 可用：`rembg --help`（若 PATH 不可见，可用 `python3 -m rembg --help`）
- 如果 rembg 不可用，执行 `pip3 install "rembg[cpu,cli]"` 进行安装

**任一依赖缺失都应阻止继续，必须先帮助用户补齐。**

## 初始化进度计划

基于分类结果，构建并**立即展示**图片处理计划：

1. 统计所有图片：总数 N 张，其中需生成 M 张，跳过 K 张
2. 向用户展示处理计划表：

| #   | 元素名称                 | 来源     | 分类   | 需透明  | 初始状态            |
| --- | ------------------------ | -------- | ------ | ------- | ------------------- |
| 1   | {name}                   | 独立图片 | {分类} | {是/否} | ⏳ 待处理 / ⏭️ 跳过 |
| 2   | {轮播名称} > {asset标题} | 轮播组件 | {分类} | {是/否} | ⏳ 待处理 / ⏭️ 跳过 |

并告知用户：「共 M 张图片需要生成（含 X 张轮播图片），预计耗时约 M-2M 分钟。」

## 尺寸归一化为 ratio（必须执行）

图片生成工具只支持固定宽高比，不能直接传任意分辨率。  
因此在构造每张图片的 prompt 参数时，必须基于原图尺寸（`width`/`height`）计算并写入受支持的 `ratio`。

支持的 `ratio` 仅限以下集合：

| 分辨率 | 1:1 | 4:3 | 3:4 | 16:9 | 9:16 | 3:2 | 2:3 | 21:9 |
| ------ | --- | --- | --- | ---- | ---- | --- | --- | ---- |

### 计算规则

对每张 `skip=false` 的图片：

1. 计算原始比例：`rawRatio = width / height`
2. 将 `rawRatio` 与候选比例集合逐一比较，选择**绝对差值最小**的那个作为最终 `ratio`
3. 若出现差值并列，按更贴近原图方向优先：
   - 横图（`width >= height`）优先横向比例：`4:3`、`16:9`、`3:2`、`21:9`
   - 竖图（`width < height`）优先纵向比例：`3:4`、`9:16`、`2:3`
4. 如果 `width` 或 `height` 缺失，才允许回退到 `image-classifications.json` 里已有的 `ratio`；回退值不在支持集合时，默认使用 `1:1`

> 目标是保持构图接近原图，同时保证传给图片生成工具的参数始终合法。

## 透明背景处理策略

**核心原则：永远不要让即梦直接生成透明背景图片。**

即梦等图片生成模型在生成透明/alpha 通道图片时质量很差（边缘粗糙、主体不完整）。因此本流程采用**两阶段策略**：

1. **生成阶段（本步骤）**：要求模型生成**不透明的纯白色背景**图片
2. **去背景阶段（Step 8c）**：使用 `rembg i -m u2netp` 将背景去除并输出透明图

### needsTransparent 对 prompt 的影响规则

对每张 `needsTransparent=true` 的图片，**无论其分类是什么**，prompt 都必须：

1. **追加白色背景指令**：在 prompt 中明确包含 `"纯白色背景(#FFFFFF)，不透明，不要任何图案、纹理或渐变"`
2. **追加排除词**：在 negativePrompt 中追加 `"透明背景、半透明、alpha通道、棋盘格背景"`
3. **主体居中**：确保主体在画面中居中且完整，与白色背景有清晰边界

对 `needsTransparent=false` 的图片，不添加上述约束。

## 分类专属 Prompt 策略

**每个 Prompt 末尾都必须追加"风格锚点片段"**，将模板画像的客观特征注入生成指令：

> **风格锚点片段模板**：`"整体画面保持 {original_palette.saturation} 饱和度、{original_palette.brightness} 明度、{original_palette.contrast} 对比层次，与模板原有设计调性协调"`

### 场景背景图

- Prompt：`"{主题}主题场景背景图，{视觉描述中的关键场景元素}，大气开阔的构图，丰富的细节和层次感，柔和的光影效果，色调以{配色方案主色调}为基调，适合作为微信图文背景，高分辨率，超高清画质，{风格锚点片段}"`
- 排除：文字、水印、logo、人物特写、杂乱

### 人物/角色

- Prompt：`"{主题}主题{人物描述}，{匹配原图风格：卡通/写实/Q版/插画}风格，纯白色背景(#FFFFFF)，不透明，不要任何图案、纹理或渐变，居中构图，表情生动自然，细节精致，色彩与{配色方案}协调，{风格锚点片段}"`
- 排除：变形、多余手指、模糊面部、杂乱背景、透明背景、半透明

### 装饰/插图

- Prompt：`"{主题}主题装饰元素，{元素用途描述}，简洁精致的{匹配原图风格：插画/扁平/线条/水彩}风格，纯白色背景(#FFFFFF)，不透明，干净无杂色，不要任何图案、纹理或渐变，适合作为页面点缀装饰，色彩使用{配色方案中的点缀色}，矢量感，干净利落，{风格锚点片段}"`
- 排除：复杂背景、写实风格、文字、透明背景、半透明

### 文字类图片

- Prompt：`"{根据 name 和视觉描述推断的文字内容}，{匹配原图风格：书法/艺术字/创意字体}风格，{主题}主题意境，纯白色背景(#FFFFFF)，不透明，不要任何图案、纹理或渐变，色彩使用{配色方案中的强调色}，精致的排版和字体设计，{风格锚点片段}"`
- 排除：模糊、变形文字、难以辨认、透明背景、半透明

### 通用配图

- 若 `needsTransparent=true`：
  - Prompt：`"{主题}，{图片用途（name 字段）}，{视觉描述中的风格}风格，色彩鲜明，与主题相关，高品质，适合微信图文，纯白色背景(#FFFFFF)，不透明，不要任何图案、纹理或渐变，主体居中且完整，{风格锚点片段}"`
  - 排除：低质量、模糊、水印、变形、透明背景、半透明、alpha通道
- 若 `needsTransparent=false`：
  - Prompt：`"{主题}，{图片用途（name 字段）}，{视觉描述中的风格}风格，色彩鲜明，与主题相关，高品质，适合微信图文，{风格锚点片段}"`
  - 排除：低质量、模糊、水印、变形

### 色调关系特殊规则

- 如果 `tone_alignment=aligned`：优先使用模板画像的原始色温（`original_palette.temperature`）作为主导
- 如果 `tone_alignment=independent`：可大胆融入新主题的色相，但**风格锚点片段的饱和度/明度/对比约束仍然必须保留**

## 逐张构造 Prompt

按 `image-classifications.json` 中的顺序（先独立 image，再 slideshow 内的 asset），为每张 `skip=false` 的图片：

1. **（必须）计算归一化 ratio**：
   - 取该图的 `width` 和 `height`，计算 `rawRatio = width / height`
   - 将 `rawRatio` 与支持集合 `{1:1(1.0), 4:3(1.333), 3:4(0.75), 16:9(1.778), 9:16(0.5625), 3:2(1.5), 2:3(0.667), 21:9(2.333)}` 逐一比较，选择**绝对差值最小**的作为最终 `ratio`
   - 若差值并列：横图（width≥height）优先横向比例，竖图（width<height）优先纵向比例
   - 仅当 `width` 或 `height` 缺失时，才回退到 `image-classifications.json` 已有的 `ratio`；回退值不在支持集合时默认 `1:1`
   - **⚠️ 最终 ratio 必须是支持集合中的值，不能使用原始比例（如 “5:8”）**
2. 根据其 `category` 选择对应的 Prompt 模板
3. 填入该图的 `visualDescription`、`toneAlignment`、主题、配色等具体参数
4. 生成完整 prompt 字符串和排除词（negativePrompt）
5. 将计算好的 `ratio` 写入该图的 prompt 条目中

## 后置写入

将所有构造好的 prompt 写入任务目录的 `image-prompts.json`：

```json
{
  "styleAnchor": "整体画面保持 {saturation} 饱和度、{brightness} 明度、{contrast} 对比层次，与模板原有设计调性协调",
  "prompts": [
    {
      "index": 0,
      "elementId": "元素ID",
      "assetId": null,
      "name": "背景图",
      "source": "image",
      "category": "场景背景图",
      "ratio": "2:3",
      "needsTransparent": false,
      "prompt": "完整的生成提示词...",
      "negativePrompt": "文字、水印、logo..."
    }
  ]
}
```

> **⚠️ ratio 校验**：写入前逐条检查 `ratio` 字段，确保每张图片的 ratio 都属于支持集合 `{1:1, 4:3, 3:4, 16:9, 9:16, 3:2, 2:3, 21:9}`。如果发现不在集合中的值（如 `"5:8"`、`"750:1200"` 等原始比例），说明归一化步骤被遗漏，必须重新计算。

同时初始化 `image-progress.json`：

```json
{
  "total": 7,
  "toGenerate": 5,
  "completed": 0,
  "failed": 0,
  "skipped": 2,
  "images": [
    {
      "index": 0,
      "elementId": "元素ID",
      "assetId": null,
      "name": "背景图",
      "source": "image",
      "slideshowElementId": null,
      "category": "场景背景图",
      "needsTransparent": false,
      "status": "pending",
      "jimengUrl": null,
      "localPath": null,
      "cdnSrc": null,
      "bgRemoved": false,
      "error": null
    }
  ]
}
```

跳过的图片 `status` 设为 `"skipped"`。

更新 `meta.json` 的 `currentStep` 为 `"08a"`。然后自动进入 Step 8b。
