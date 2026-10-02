# 完整打通方案总结

## 最终效果

LocalMiniDrama 通过 `minimax_h3` 协议发送带参考图的视频生成请求，经 Rust 代理转发到本地 ComfyUI，执行 H3 工作流生成视频。

---

## 一、问题根源

LocalMiniDrama 虽然支持 H3 协议，但 `videoService.js` 里有一段逻辑：

```javascript
image_url: hasOmniRefs ? undefined : row.image_url,
first_frame_url: hasOmniRefs ? undefined : row.first_frame_url,
last_frame_url: hasOmniRefs ? undefined : row.last_frame_url,
```

当分镜有参考图时（`hasOmniRefs = true`），这三个字段被**主动置为 undefined**，导致参考图传不到代理，代理只能用 `placeholder.png` 占位，ComfyUI 因图片尺寸不达标而报错。

---

## 二、核心修改（LocalMiniDrama 侧）

**文件**：`backend-node/src/services/videoService.js`

**修改**：把上面三行改成：

```javascript
image_url: row.image_url || (reference_urls && reference_urls[0]) || undefined,
first_frame_url: row.first_frame_url,
last_frame_url: row.last_frame_url,
```

这样参考图就能传到 `callVideoApi`，进而传给 `callMinimaxH3VideoApi`。

---

## 三、代理侧不需要改

你的 Rust 代理（ComfyUI-OpenAI-API-Refactored）里的 `video.rs` 本来就有这段：

```rust
let reference_urls: Vec<String> = video_req.content.iter()
    .filter(|c| c.item_type == "image_url")
    .filter_map(|c| c.image_url.as_ref()?.url.clone())
    .collect();
```

它不区分 `role`，只要 `type == "image_url"` 就会收集参考图。LocalMiniDrama 改完后，H3 协议会把图片组装成：

```json
{"type": "image_url", "image_url": {"url": "..."}, "role": "first_frame"}
```

代理自然就能收到，下载、上传到 ComfyUI 的 input 目录，注入工作流的 `LoadImage` 节点。

---

## 四、LocalMiniDrama AI 配置

| 字段 | 值 |
|---|---|
| 接口规范 | `minimax_h3` |
| Base URL | `http://127.0.0.1:8080/v1` |
| 提交端点 | `/videos/generations` |
| 查询端点 | `/tasks/{task_id}` |
| 模型 | `video_minimax_h3_i2v` |

---

## 五、代理配置

代理监听 `0.0.0.0:8080`，工作流目录里有 `video_minimax_h3_i2v`，节点 114 是 `LoadImage`，标题为 `Reference Image 1`。

---

## 六、重新打包 exe 的完整流程

因为 LocalMiniDrama 是 Electron 打包应用，改完源码后必须重新打包：

**1. 安装 Node.js**（推荐 LTS 版本，如 22 或 24）

**2. 在 `backend-node` 同级目录安装依赖**：

```cmd
cd C:\Users\jiang\LocalMiniDrama
npm install
```

**3. 安装前端依赖**：

```cmd
cd frontweb
npm install
```

**4. 配置 `.npmrc`**（解决下载慢的问题）：

```
electron_mirror=https://npmmirror.com/mirrors/electron/
electron_builder_binaries_mirror=https://npmmirror.com/mirrors/electron-builder-binaries/
```

**5. 打包**：

```cmd
cd desktop
npm run dist
```

**6. 产物在 `desktop\release\` 下**，运行新 exe。

**7. 如果打包中途卡在下载 Electron 或 nsis-resources**，手动下载对应文件放入缓存目录：
- Electron：`%LOCALAPPDATA%\electron\Cache\`
- nsis-resources：`%LOCALAPPDATA%\electron-builder\Cache\nsis\nsis-resources-3.4.1\`

---

## 七、验证成功的标志

代理日志里依次出现：

```
📥 Video request: ... "type":"image_url" ... "role":"first_frame"
📸 视频生成参考图数量: 1
📌 节点 114 分配参考图 1: http://...
🚀 Submitting payload: ... "image":"2a084884302347ef.png"
poll_history_for_videos attempt=N ... running: 1 and pid_found: true
📥 poll_history_for_videos 返回视频数量: 1
✅ Video task vid-xxx completed
```

LocalMiniDrama 前端拿到视频，任务状态变成 `completed`。

---

## 八、后续可优化项

- **加速**：工作流里 `Boolean (Enable Lightning LoRA)` 改成 `true`，走 8 步采样
- **多参考图**：在 ComfyUI 工作流里增加更多 `LoadImage` 节点（`Reference Image 1/2/3...`），代理会自动按顺序分配
- **数据备份**：定期备份 `%APPDATA%\localminidrama-desktop\backend\data\drama_generator.db`
