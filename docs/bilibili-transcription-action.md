# B 站无字幕视频转文字：GitHub Actions 复用说明

这份说明记录 2026-09-14 实测跑通的方案，目标是：**即使 B 站视频没有原生 CC 字幕，也可以通过 GitHub Actions 自动抓取视频音频并做 ASR 转写。**

对应 workflow：`.github/workflows/bilibili-transcribe.yml`

## 1. 已验证结论

实测视频：`https://www.bilibili.com/video/BV1ne4y1c7tX/`

测试确认的真实处理链路不是“读取 B 站字幕”，而是：

1. GitHub Actions 启动 Ubuntu runner。
2. Playwright/Chromium 打开 VocaScript 的 Bilibili 转写页面。
3. 输入 B 站视频 URL。
4. VocaScript 通过 `/api/media-download/info` 识别 Bilibili 视频。
5. 通过 `/api/media-download/process` 下载媒体。
6. `/api/progress` 显示下载、音频优化和完成状态。
7. 服务把视频转换成音频文件（本次测试得到约 328 KB 的 M4A）。
8. 自动调用 `/api/transcribe` 对音频做 ASR。
9. `/api/transcribe` 返回 `text/event-stream`（SSE）。
10. 收到 `event: complete` 后，`data` 中包含完整 `transcript` 数组、时间戳、speaker 和文本。

因此，这套方法**不依赖 B 站原生字幕轨**。

本次实测中，VocaScript 返回的路由信息显示转写提供方为 Gemini、模型为 `gemini-2.5-flash`。这只是当次实际结果，第三方以后可能切换模型或提供方，workflow 不应依赖固定模型名。

## 2. 最简单的使用方式

### 方法 A：GitHub Actions 页面手工运行

进入仓库：

`Actions -> Bilibili audio transcription -> Run workflow`

填写：

- `bilibili_url`：完整 B 站视频 URL。
- `max_wait_seconds`：通常保持默认 `420` 即可。

运行完成后，在该 run 的 Artifacts 中下载：

- `transcript.txt`：带时间戳和 speaker 的文本。
- `transcript_plain.txt`：纯正文。
- `transcript.json`：结构化分段结果。
- `metadata.json`：标题、时长、UP 主等安全元数据。
- `result.json`：成功状态、段数、路由信息、访客额度状态等。
- `network_summary.json`：诊断用的接口路径和 HTTP 状态，不保存敏感请求体。
- `01-request.png` / `02-final.png`：自动化过程截图。

### 方法 B：Issue 自动触发

新建 Issue：

- 标题必须以 `[BiliTranscribe]` 开头。
- 正文中放一个 `bilibili.com/video/...` 或 `b23.tv/...` 链接。

例如：

```text
标题：[BiliTranscribe] 测试视频
正文：https://www.bilibili.com/video/BVxxxxxxxxxx/
```

Issue 打开后，workflow 会自动识别链接并运行。

这个入口主要是为了以后让 ChatGPT/GitHub 连接器也能复用：如果不能直接调用 `workflow_dispatch`，可以创建一个符合规则的 Issue 来触发 Actions。

## 3. 最关键的完成判断

**不要用页面里出现 `Transcript`、`Transcribe`、`Download` 等文字作为成功判断。**

VocaScript 页面静态说明里本身就包含 `transcript` 这个词，实验第一版正是因此误判“已经完成”。

可靠判断应该是：

```text
POST /api/transcribe
Content-Type: text/event-stream
...
event: complete
data: {"transcript":[...], "router":{...}}
```

workflow 已按这个规则实现：只有解析到 SSE 的 `event: complete`，并且其中存在非空 `transcript` 数组，才认定转写成功。

## 4. 已踩过的坑

### 4.1 Actions run 不一定立刻出现在查询结果中

通过 GitHub API 写入 workflow 或创建触发事件后，马上查询 Actions API，可能短暂得到 `total_count: 0`。本次实测几秒后 run 正常出现并成功执行。

因此：

- 不要在事件创建后的第一瞬间就判定“Actions 没执行”。
- 应稍后重新查询 workflow runs。

### 4.2 GitHub-hosted runner 直接访问 B 站 API 可能不稳定

实验中直接访问 B 站 API 来判断“原生字幕轨数量”时，拿到的内容不是预期 JSON，出现 `JSONDecodeError`。

可能原因是 B 站对数据中心 IP、请求头、风控策略等有限制。

因此当前 workflow **不依赖 B 站官方 API 来判断有没有字幕**，而是直接让第三方抓取音频并做 ASR。

### 4.3 页面状态在转写阶段会变化

媒体下载阶段能看到：

- Processing Progress
- Downloading
- Converting
- Ready

但进入 Transcript 页面后，这些状态字段会消失。因此不能一直靠页面上的 `Status` 或 `Processing Progress` 来判断最终完成。

最终仍然应以 `/api/transcribe` SSE 的 `event: complete` 为准。

### 4.4 `/api/transcribe` 是流式响应

它不是普通的一次性 JSON，而是 Server-Sent Events。

本次实际响应包含：

- `event: progress`
- `event: router`
- 多个 `event: data`
- `event: validation-errors`
- `event: complete`

只有 `event: complete` 的 JSON 才适合作为最终结构化结果。

### 4.5 访客额度是动态的

本次实验期间 `/api/transcribe/can-start` 返回过剩余访客次数，但这个数字会变化，第三方规则也可能调整。

不要把“每天固定几次”写死在业务逻辑里。当前 workflow 只记录接口返回的 `ok`、`remaining`、`resetAt`，不假设固定额度。

## 5. 当前 workflow 的安全处理

为了以后复用时尽量少泄露信息，稳定版 workflow 做了这些处理：

- 不在 Actions 日志中打印完整转写正文。
- 不保存 VocaScript 返回的 `acquisitionGrant` 等临时授权字段。
- 不保存 cookies、localStorage、sessionStorage。
- 不保存完整 HTML。
- 不上传抓到的原始视频或音频。
- 只上传 transcript、非敏感元数据、接口状态和截图。
- Artifact 默认保留 14 天。

注意：这个仓库目前是 **public**。即使脚本主动减少日志内容，也不应该用它处理私密、内部或敏感视频。

## 6. 隐私、版权和第三方依赖

这个方案会把视频 URL / 音频交给第三方 VocaScript 处理。使用前应注意：

- 只处理你有权处理的公开或授权内容。
- 对内部、隐私、涉密或敏感音视频不要使用公共仓库 + 第三方转写方案。
- 第三方的页面结构、接口、额度、模型、服务条款都可能变化。
- 如果 VocaScript 改版，优先检查 `/api/media-download/*` 与 `/api/transcribe` 的网络行为，而不是只改 CSS selector。

## 7. 故障排查顺序

以后如果跑不通，建议按以下顺序定位：

1. Actions 是否真的启动；不要因为刚触发时查询为 0 就立即判失败。
2. URL 输入框和 Process/Transcribe 按钮是否还能找到。
3. `/api/media-download/info` 是否返回 200，并能识别标题、时长、extractor=`BiliBili`。
4. `/api/media-download/process` 是否返回 queued ID。
5. `/api/progress` 是否从 downloading -> converting -> completed。
6. 是否出现 `/api/transcribe/can-start`，额度是否允许。
7. `POST /api/transcribe` 是否返回 200 + `text/event-stream`。
8. SSE 中是否最终出现 `event: complete`。
9. `event: complete` 的 `transcript` 是否为非空数组。

如果第 3 步就失败，通常是抓取 B 站媒体的问题；如果第 7/8 步失败，则更可能是第三方转写额度、服务异常或接口改版。

## 8. 2026-09-14 成功样本的关键数据

成功样本的观测值，仅用于以后排查对照，不应硬编码：

- BVID：`BV1ne4y1c7tX`
- 视频时长：约 31.7 秒
- 下载媒体量：约 676 KB
- 优化后 M4A：336,107 bytes（页面显示约 328.23 KB）
- extractor：`BiliBili`
- `/api/media-download/info`：200
- `/api/media-download/process`：200 / queued
- `/api/progress`：downloading -> converting -> completed
- `/api/transcribe`：200 / `text/event-stream`
- 最终事件：`event: complete`
- 最终 transcript：7 个分段

## 9. 后续复用约定

以后需要转 B 站无字幕视频时，优先复用本 workflow，而不是重新临时写脚本。

推荐请求方式：

```text
把这个 B 站视频转成文本：<URL>
```

执行逻辑应优先理解为：

```text
触发 Bilibili audio transcription Action
-> 等待 run 完成
-> 获取 artifact
-> 读取 transcript.json / transcript.txt
-> 再做摘要、知识提取、经验总结等后处理
```

如果第三方服务发生变化，则先依据本说明的“故障排查顺序”修复 workflow，并同步更新这份文档。
