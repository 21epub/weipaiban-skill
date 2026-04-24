# Step 8c：图片背景去除

对需要透明背景的图片执行 `rembg` 去背景处理，默认模型固定为 `u2netp`，以兼顾性能与稳定性。

**处理命令**：`rembg i -m u2netp <input> <output>`

**本步骤无需子代理**：处理是确定性操作（执行固定 CLI 命令），主代理直接逐张执行即可。

## 前置读取

从任务目录读取以下文件：

1. **`meta.json`**：获取 `taskDir`
2. **`image-progress.json`**：筛选 `needsTransparent=true` 且 `status="generated"` 的条目

如果没有符合条件的图片（所有图片都不需要透明，或已处理完毕），直接跳到 Step 8d。

## 前置依赖检查

开始逐张处理前，必须先确认依赖可用：

1. `python3 --version` 可正常返回
2. `rembg --help` 可正常返回（若 PATH 不可见，可用 `python3 -m rembg --help`）
3. 如果 `rembg` 不可用，安装 CPU 版本 CLI：`pip3 install "rembg[cpu,cli]"`

> 首次运行会下载模型并缓存到 `~/.u2net/`，首张图可能明显更慢。建议先对单张测试图预热一次，再进行批量处理。

## 处理流程

### 逐张处理

对每张符合条件的图片：

1. 确认输入文件 `{taskDir}/images/{elementId}_generated.png` 存在
2. 执行去背景命令（保持串行，不并发）：

   ```bash
   rembg i -m u2netp "{taskDir}/images/{elementId}_generated.png" "{taskDir}/images/{elementId}_nobg.png"
   ```

3. **如果成功**（命令退出码为 0，且输出文件存在且可读）：
   - 更新 `image-progress.json`：`status` → `"bg_removed"`，`localPath` → `"{taskDir}/images/{elementId}_nobg.png"`，`bgRemoved` → `true`
   - 向用户反馈：`✅ [{序号}] {元素名称} - 去背景完成`
4. **如果失败**（命令报错、超时或输出文件缺失）：
   - 不阻塞后续处理。标记 `bgRemoved` → `false`，`status` 保持 `"generated"`（Step 8d 将使用即梦原图 URL 上传）
   - 向用户反馈：`⚠️ [{序号}] {元素名称} - 去背景失败，将使用原图上传`

## 后置写入

所有需去背景的图片处理完毕后，确认 `image-progress.json` 已更新。更新 `meta.json` 的 `currentStep` 为 `"08c"`。然后自动进入 Step 8d。
