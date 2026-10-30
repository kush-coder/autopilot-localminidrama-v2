# 低价 Seedance2 视频厂商预设设计

日期：2026-10-02  
状态：已定稿（待实现）

## 背景

用户接入 [ohmybb 视频工作台 API](https://workbench.ohmybb.xyz/docs/api) 后，任务提交/轮询可成功，但成片地址为需鉴权的 `/v1/videos/{id}/content`。浏览器 `<video>` 与当前本地下载均不带 `Authorization`，导致无法播放。手工配置还易出现端点不一致、Sora 默认时长档（4/8/12）与对方档位（5/10/15）冲突。

目标：在 AI 配置的视频厂商中增加「低价 Seedance2」预设，用户只需填写 API Key；Base URL、协议、模型、时长/分辨率映射与带密钥本地下载均自动处理。

## 决策摘要

| 决策 | 选择 |
|------|------|
| 上游接口 | 文档第九节 **Sora 兼容**（`/api/openai`） |
| 实现策略 | 预设厂商 + **复用现有 `sora` 协议** + 厂商特判 |
| 时长 | 按该厂商自动映射到 5/10/15 |
| 鉴权下载 | **仅**对该新厂商开启 |

## 厂商预设

| 项 | 值 |
|----|-----|
| 显示名 | 低价 Seedance2 |
| 内部 id | `cheap_seedance2` |
| 服务类型 | `video` |
| `api_protocol` | `sora` |
| Base URL | `https://workbench.ohmybb.xyz/api/openai` |
| 提交端点 | `/v1/videos`（可留空，走 Sora 默认） |
| 查询端点 | `/v1/videos/{taskId}`（可留空，走 Sora 默认） |
| 模型列表 | `wb-seedance-2-fast`、`wb-seedance-2.5` |
| 默认模型 | `wb-seedance-2-fast` |

### 前端改动点

文件：`frontweb/src/components/AIConfigContent.vue`

1. `providerConfigs.video` 增加一项 `{ id: 'cheap_seedance2', name: '低价 Seedance2', models: [...] }`
2. `providerProtocolMap.cheap_seedance2 = 'sora'`
3. `getBaseUrlForProvider`：`cheap_seedance2` → `https://workbench.ohmybb.xyz/api/openai`
4. 选择该厂商时：自动填协议、Base URL、模型；端点可留空或填上述默认

用户只需填写 API Key（`vw_…`）并保存。

### 后端改动点（配置默认）

文件：`backend-node/src/services/aiConfigService.js`（若有按 provider 补默认 endpoint 的逻辑）

- 识别 `cheap_seedance2` 时，与 Sora/openai 视频一致：默认 `/v1/videos` 与 `/v1/videos/{taskId}`（若当前逻辑已因 `api_protocol=sora` 覆盖，则可不重复）

## 时长与分辨率映射

仅当 `provider === 'cheap_seedance2'`（大小写不敏感）时，在 `callSoraVideoApi`（或紧邻的调用前）覆盖通用 Sora 的 4/8/12 与 Sora size 表。

### 时长

| 请求秒数 | `wb-seedance-2-fast` | `wb-seedance-2.5` |
|----------|----------------------|-------------------|
| ≤5 | 5 | 5 |
| 6–10 | 10 | 10 |
| ≥11 | 15 | 10（无 15 档） |
| 未传 | 10 | 10 |

`seconds` 仍以字符串形式写入 multipart（与现有 Sora 一致）。

### 分辨率

| 比例 | size |
|------|------|
| 16:9 | `1280x720` |
| 9:16 | `720x1280` |
| 1:1 | `1024x1024` |
| 4:3 | `960x720` |
| 3:4 | `720x960` |
| 其他 | `1280x720` |

参考图：继续走现有 Sora 的 `input_reference` multipart 逻辑；若对方对图片尺寸有要求，可沿用现有 resize 到目标 `size` 的行为。

## 成片带密钥下载

### 问题

任务返回的 `video_url` 指向同源 `/content`，必须带 `Authorization: Bearer <api_key>`。当前 `downloadVideoToLocal` 为裸 `GET`。

### 行为

1. 轮询仍走现有 Sora 分支，解析出 `video_url`。
2. `finalizeSuccessfulVideo` → `downloadVideoToLocal` 时：若该次任务对应配置的 `provider` 为 `cheap_seedance2`，则下载请求附带 `Authorization: Bearer ${api_key}`。
3. 下载成功：写入 `local_path`，前端播 `/static/...`。
4. 下载失败（401/404 等）：记日志（标明「鉴权下载失败」）；对该厂商**不**把任务标为可播成功，也**不**把需鉴权的远程 URL 交给前端当 `<video src>`（避免黑屏假成功）。可保留 `video_url` 供排查，但 `local_path` 为空且状态需让用户感知失败或可重试下载。

### 传参方式

优先扩展 `downloadVideoToLocal(storagePath, videoUrl, videoGenId, log, projectSubdir, options?)`，`options` 含 `{ apiKey }` 或 `{ authHeader }`；由 `finalizeSuccessfulVideo` / 调用链根据 config.provider 决定是否传入。避免全局改变其他厂商下载行为。

### 明确不做

- 不为所有同源 `/content` URL 通用带 Key（仅本厂商）
- 不自动追加 `?compat=1`（H.265 兼容转码）；后续可单开需求

## 错误处理

| 场景 | 行为 |
|------|------|
| Key 无效 / 未填 | 提交或下载 401 → 任务失败，提示检查 API Key |
| 映射后上游仍 400 | 透传对方错误信息 |
| 任务失败 / 额度不足 | 沿用现有轮询失败路径 |
| 成片下载 401/404 | 不标为可播成功；日志写清鉴权下载失败 |
| `wechat_unbound` | 透传提示（免费号需网页端绑微信） |

## 测试计划

1. UI：选「低价 Seedance2」后 Base URL / `api_protocol` / 模型自动正确。
2. 单元：时长映射表（含 2.5 无 15 秒、未传默认 10）。
3. 单元：该厂商下载请求含 `Authorization`；其他厂商下载请求不含。
4. （有额度时）端到端：生成 → 轮询 completed → 本地下载成功 → `/static` 可播。

## 涉及文件（预期）

- `frontweb/src/components/AIConfigContent.vue` — 预设、协议、Base URL
- `backend-node/src/services/videoClient.js` — Sora 提交时厂商特判时长/分辨率
- `backend-node/src/services/videoService.js` — 下载带 Key、调用链传参
- 可选：`backend-node/src/services/aiConfigService.js` — 默认 endpoint
- 可选：`backend-node/test/*.test.js` — 映射与下载鉴权单测

## 非目标

- 不实现原生 `/api/v1/videos` 另一套协议
- 不改其他 Sora 中转站的默认 4/8/12 行为
- 不做通用视频代理播放端点
