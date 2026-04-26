# 生图模型 & 视频模型接口完整文档

---

## 一、模型总览

### 生图模型

| 模型名称 | 公开名称 | 账号池 | 模式 | 说明 |
|---|---|---|---|---|
| `grok-imagine-image-lite` | Grok Imagine Image Lite | basic+ | fast | 通过 Chat 端点生成，不支持宽高比控制，速度快 |
| `grok-imagine-image` | Grok Imagine Image | super+ | auto | WebSocket 速度模式（speed） |
| `grok-imagine-image-pro` | Grok Imagine Image Pro | super+ | auto | WebSocket 质量模式（quality/pro） |

### 图片编辑模型

| 模型名称 | 公开名称 | 账号池 | 说明 |
|---|---|---|---|
| `grok-imagine-image-edit` | Grok Imagine Image Edit | super+ | 上传参考图 + 文字提示编辑图片 |

### 视频模型

| 模型名称 | 公开名称 | 账号池 | 说明 |
|---|---|---|---|
| `grok-imagine-video` | Grok Imagine Video | super+ | 文生视频 / 图生视频，支持分段拼接 |

---

## 二、生图接口

### 2.1 独立生图端点

**`POST /v1/images/generations`**

标准 OpenAI 兼容接口，仅返回非流式结果。

**请求头**
```
Authorization: Bearer <api_key>
Content-Type: application/json
```

**请求体**
```json
{
  "model": "grok-imagine-image",
  "prompt": "a cute cat sitting on a cloud",
  "n": 1,
  "size": "1024x1024",
  "response_format": "url"
}
```

**参数说明**

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `model` | string | 必填 | 模型名称，见上方模型表 |
| `prompt` | string | 必填 | 图片描述提示词 |
| `n` | int | 1 | 生成数量。lite 模型最多 4 张，其他最多 10 张 |
| `size` | string | `1024x1024` | 图片尺寸，见下方尺寸表 |
| `response_format` | string | `url` | 返回格式：`url` 或 `b64_json` |

**支持的 `size` 及对应宽高比**

| size | 宽高比 |
|---|---|
| `1024x1024` | 1:1 |
| `1280x720` | 16:9 |
| `720x1280` | 9:16 |
| `1792x1024` | 3:2 |
| `1024x1792` | 2:3 |

> `grok-imagine-image-lite` 不支持宽高比控制，`size` 参数被忽略。

**成功响应**
```json
{
  "created": 1714000000,
  "data": [
    { "url": "https://..." }
  ]
}
```

若 `response_format=b64_json`：
```json
{
  "created": 1714000000,
  "data": [
    { "b64_json": "<base64字符串>" }
  ]
}
```

---

### 2.2 通过 Chat 接口生图

**`POST /v1/chat/completions`**

支持流式和非流式，返回 Chat 格式响应（Markdown 图片链接）。

**请求体**
```json
{
  "model": "grok-imagine-image",
  "messages": [
    { "role": "user", "content": "画一只可爱的猫" }
  ],
  "stream": false,
  "image_config": {
    "n": 2,
    "size": "1024x1024",
    "response_format": "url"
  }
}
```

**`image_config` 参数**

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `n` | int | 1 | 生成数量（1-10，lite 模型 1-4） |
| `size` | string | `1024x1024` | 同上方尺寸表 |
| `response_format` | string | `url` | `url` 或 `b64_json` |

**流式响应** (`stream: true`)

返回 SSE 事件流，包含进度 thinking chunk 和图片内容 chunk：
```
data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"thinking":"图片正在生成 50% (0/2)"}}]}

data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"content":"![image](https://...)"}}]}

data: [DONE]
```

**非流式响应**
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "grok-imagine-image",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "![image](https://...)\n\n![image](https://...)",
      "reasoning_content": "图片正在生成 50% (0/2)\n图片正在生成 100% (2/2)"
    }
  }]
}
```

---

### 2.3 图片编辑接口

#### 方式一：独立端点（multipart/form-data）

**`POST /v1/images/edits`**

```
Content-Type: multipart/form-data
Authorization: Bearer <api_key>
```

**表单字段**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `model` | string | 是 | 必须为 `grok-imagine-image-edit` |
| `prompt` | string | 是 | 编辑指令描述 |
| `image[]` | file | 是 | 参考图片文件（支持多张，最多 5 张） |
| `mask` | file | 否 | 暂不支持，传入会报错 |
| `n` | int | 否 | 生成数量，1-2，默认 1 |
| `size` | string | 否 | 仅支持 `1024x1024` |
| `response_format` | string | 否 | `url` 或 `b64_json` |

**成功响应**
```json
{
  "created": 1714000000,
  "data": [
    { "url": "https://..." },
    { "url": "https://..." }
  ]
}
```

#### 方式二：通过 Chat 接口编辑

**`POST /v1/chat/completions`**

```json
{
  "model": "grok-imagine-image-edit",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "把背景换成海边" },
        { "type": "image_url", "image_url": { "url": "https://..." } },
        { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,..." } }
      ]
    }
  ],
  "stream": false,
  "image_config": {
    "n": 1,
    "size": "1024x1024",
    "response_format": "url"
  }
}
```

> `image_url` 支持：HTTP/HTTPS URL、Base64 Data URI、本地代理 URL（`/v1/files/image?id=...`）

---

## 三、视频接口

### 3.1 异步任务模式（推荐）

#### 创建视频任务

**`POST /v1/videos`**

```
Content-Type: multipart/form-data
Authorization: Bearer <api_key>
```

**表单字段**

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `model` | string | 是 | — | 必须为 `grok-imagine-video` |
| `prompt` | string | 是 | — | 视频描述提示词 |
| `seconds` | int | 否 | 6 | 视频时长（秒），支持：6、10、12、16、20 |
| `size` | string | 否 | `720x1280` | 视频尺寸，见下方尺寸表 |
| `resolution_name` | string | 否 | 由 size 决定 | `480p` 或 `720p` |
| `preset` | string | 否 | `custom` | 风格预设：`fun`、`normal`、`spicy`、`custom` |
| `input_reference[]` | file | 否 | — | 参考图片文件上传（图生视频），最多 5 张 |
| `input_reference_url[]` | string | 否 | — | 参考图片 URL 或 Base64 Data URI（与 `input_reference[]` 合并，总数不超过 5 张） |

**支持的视频 `size`**

| size | 宽高比 | 默认分辨率 |
|---|---|---|
| `720x1280` | 9:16 | 720p |
| `1280x720` | 16:9 | 720p |
| `1024x1024` | 1:1 | 720p |
| `1024x1792` | 9:16 | 720p |
| `1792x1024` | 16:9 | 720p |

**`seconds` 与分段规则**

| seconds | 分段策略 |
|---|---|
| 6 | 单段 6s |
| 10 | 单段 10s |
| 12 | 两段：6s + 6s |
| 16 | 两段：10s + 6s |
| 20 | 两段：10s + 10s |

**`preset` 风格说明**

| preset | 上游 flag |
|---|---|
| `fun` | `--mode=extremely-crazy` |
| `normal` | `--mode=normal` |
| `spicy` | `--mode=extremely-spicy-or-crazy` |
| `custom` | `--mode=custom` |

**成功响应（立即返回，任务异步执行）**
```json
{
  "id": "video_a1b2c3d4...",
  "object": "video",
  "created_at": 1714000000,
  "status": "queued",
  "model": "grok-imagine-video",
  "progress": 0,
  "prompt": "a sunset over the ocean",
  "seconds": "6",
  "size": "720x1280",
  "quality": "standard"
}
```

**任务状态流转**：`queued` → `in_progress` → `completed` / `failed`

---

#### 查询视频任务状态

**`GET /v1/videos/{video_id}`**

```
Authorization: Bearer <api_key>
```

**进行中响应**
```json
{
  "id": "video_a1b2c3d4...",
  "object": "video",
  "status": "in_progress",
  "progress": 45,
  "created_at": 1714000000
}
```

**完成响应**
```json
{
  "id": "video_a1b2c3d4...",
  "object": "video",
  "status": "completed",
  "progress": 100,
  "created_at": 1714000000,
  "completed_at": 1714000300
}
```

**失败响应**
```json
{
  "id": "video_a1b2c3d4...",
  "object": "video",
  "status": "failed",
  "error": {
    "code": "video_generation_failed",
    "message": "..."
  }
}
```

> 任务在内存中保存 1 小时（3600s）后自动过期。

---

#### 下载视频内容

**`GET /v1/videos/{video_id}/content`**

```
Authorization: Bearer <api_key>
```

返回 `video/mp4` 文件流，文件名为 `{video_id}.mp4`。

> 任务状态必须为 `completed` 才能下载，否则返回 409 错误。

---

### 3.2 通过 Chat 接口生成视频

**`POST /v1/chat/completions`**

同步等待视频生成完成后返回（适合短视频）。

**文生视频**
```json
{
  "model": "grok-imagine-video",
  "messages": [
    { "role": "user", "content": "生成一段海浪拍打礁石的视频" }
  ],
  "stream": false,
  "video_config": {
    "seconds": 6,
    "size": "1280x720",
    "resolution_name": "720p",
    "preset": "normal"
  }
}
```

**图生视频**（消息中包含 `image_url` 块）
```json
{
  "model": "grok-imagine-video",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "让这张图动起来" },
        { "type": "image_url", "image_url": { "url": "https://..." } }
      ]
    }
  ],
  "stream": false,
  "video_config": {
    "seconds": 6,
    "size": "720x1280"
  }
}
```

**`video_config` 参数**

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `seconds` | int | 6 | 视频时长，支持 6/10/12/16/20 |
| `size` | string | `720x1280` | 视频尺寸 |
| `resolution_name` | string | 由 size 决定 | `480p` 或 `720p` |
| `preset` | string | `custom` | 风格预设 |

**流式响应** (`stream: true`)
```
data: {"choices":[{"delta":{"thinking":"视频正在生成 30%"}}]}
data: {"choices":[{"delta":{"content":"https://assets.grok.com/..."}}]}
data: [DONE]
```

**非流式响应**
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "https://assets.grok.com/...",
      "reasoning_content": "视频正在生成 30%\n视频正在生成 70%\n视频正在生成 100%"
    }
  }]
}
```

> `content` 字段的具体格式由配置 `features.video_format` 决定（见下方配置说明）。

---

## 四、文件服务接口

### 获取本地缓存图片

**`GET /v1/files/image?id={file_id}`**

无需认证，直接返回图片文件（`image/jpeg` 或 `image/png`）。

### 获取本地缓存视频

**`GET /v1/files/video?id={file_id}`**

无需认证，直接返回 `video/mp4` 文件。

---

## 五、关键配置项

在 `config.defaults.toml` 中控制生图/视频行为：

```toml
[features]
enable_nsfw = true          # 是否允许生成 NSFW 图片

# 图片返回格式
# grok_url  — 直接返回 Grok CDN URL（默认）
# local_url — 下载后存本地，返回 /v1/files/image?id=... URL
# grok_md   — Markdown 内嵌 Grok CDN URL
# local_md  — Markdown 内嵌本地代理 URL
# base64    — Markdown 内嵌 Base64 Data URI
image_format = "grok_url"

# 视频返回格式
# grok_url  — 直接返回 Grok CDN URL（默认）
# local_url — 下载后存本地，返回 /v1/files/video?id=... URL
# grok_html — HTML <video> 标签，src 为 Grok CDN URL
# local_html — HTML <video> 标签，src 为本地代理 URL
video_format = "grok_url"

[image]
timeout = 60          # 图片生成超时（秒）
stream_timeout = 60   # 流式图片生成超时（秒）

[video]
timeout = 60          # 视频生成超时（秒）

[cache.local]
image_max_mb = 0      # 本地图片缓存上限（MB），0=不限
video_max_mb = 0      # 本地视频缓存上限（MB），0=不限
```

---

## 六、错误响应格式

所有接口统一错误格式：

```json
{
  "error": {
    "message": "错误描述",
    "type": "invalid_request_error",
    "param": "size",
    "code": "model_not_found"
  }
}
```

**常见错误**

| 场景 | HTTP 状态 | type |
|---|---|---|
| 参数校验失败 | 422 | `invalid_request_error` |
| 模型不存在 | 422 | `invalid_request_error` |
| 无可用账号 | 429 | `rate_limit_error` |
| 上游服务异常 | 502 | `upstream_error` |
| 视频未就绪 | 409 | `validation_error` |

---

## 七、底层协议说明

### 生图底层协议

- **`grok-imagine-image` / `grok-imagine-image-pro`**：使用 WebSocket 连接 `wss://grok.com/ws/imagine/listen`，通过 `conversation.item.create` 消息发送请求，接收进度事件和最终图片 URL。
- **`grok-imagine-image-lite`**：使用 HTTP Chat 端点，在请求体中设置 `imageGenerationCount: 2`，从 SSE 流中解析图片事件。

### 图片编辑底层协议

1. 上传参考图到 Grok 资产服务
2. 调用 `POST /rest/media/post/create` 创建媒体帖子
3. 向 Chat 端点发送包含 `imageEditModelConfig` 的 SSE 请求
4. 从 SSE 流中解析 `streamingImageGenerationResponse` 事件获取最终图片

### 视频生成底层协议

1. 上传参考图（图生视频时）
2. 调用 `POST /rest/media/post/create` 创建媒体帖子
3. 向 Chat 端点发送包含 `videoGenModelConfig` 的 SSE 请求
4. 长视频（>10s）自动分段，后续段通过 `isVideoExtension: true` 拼接
5. 从 SSE 流中解析 `streamingVideoGenerationResponse` 事件获取最终视频 URL
