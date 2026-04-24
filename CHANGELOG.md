# Changelog

本项目遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 规范，版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.1.0] - 2026-04-24

本次发布将 Step 8（图片生成）从「必需阶段」调整为「可选阶段」，明确区分必需 / 可选依赖，并引入「用户同意后才安装」的交互原则。缺失即梦凭据或 rembg 时，作品可保留模板原图降级完成。

### 新增

- 新增参考文档 [references/runtime-dependencies.md](references/runtime-dependencies.md)，集中说明凭据、外部服务、本地二进制、磁盘写入和数据流向
- SKILL.md frontmatter 增加 `optionalEnv` / `optionalSkills` / `optionalBinaries` / `externalServices` / `writes` 元数据字段，便于 openclaw 识别可选依赖和运行时副作用
- README.md frontmatter 增加 `optional` 字段
- SKILL.md 新增「运行时依赖」章节，区分必需依赖、可选依赖、降级行为
- Step 8a 新增「⏸️ 依赖缺失交互点」：向用户展示依赖检测结果表，提供「安装 / 跳过图片生成 / 取消」三选一

### 变更

- Step 8a/b/c/d 在步骤表中标注为「可选」，并在表格上方补充说明
- Step 8a 的依赖处理从「自动检测并安装」改为「只检测、展示结果、请求用户选择」，不再主动执行 `pip install` 或安装其他技能
- Step 8c 改为固定使用 `u2netp` 轻量模型（`rembg -m u2netp`），明确首次下载 `~/.u2net/u2netp.onnx`（约 4.7MB）
- Step 8c 当 rembg 不可用时，改为跳过背景去除并使用即梦原图 URL 上传（标记 `bgRemoved=false`），而非尝试安装
- 断点恢复规则：08a 改为识别 `meta.json` 中的 `skipImageGeneration` 标志
- README.md「安全与权限」章节重写为「必需 / 可选 / 降级行为」三段式

### 移除

- 移除 Step 8a 和 Step 8c 中自动执行 `pip3 install "rembg[cpu,cli]"` 的行为（改为仅在用户明确选择「安装」后提示并确认）

## [1.0.0] - 2026-04-24

- 微排版技能初始版本

[1.1.0]: https://github.com/21epub/weipaiban-skill/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/21epub/weipaiban-skill/releases/tag/v1.0.0
