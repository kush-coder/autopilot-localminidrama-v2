# 低价 Seedance2 实现计划

日期：2026-10-02  
规格：`docs/superpowers/specs/2026-10-02-cheap-seedance2-video-provider-design.md`

## 任务

1. **前端预设** — `AIConfigContent.vue`：厂商项、协议、Base URL、选中时清端点（走 Sora 默认）
2. **后端默认 endpoint** — `aiConfigService.js`：`cheap_seedance2` → `/v1/videos` + `/v1/videos/{taskId}`
3. **时长/分辨率映射** — `videoClient.js`：导出映射函数；`callSoraVideoApi` 在 provider 为 `cheap_seedance2` 时使用
4. **完成无 URL 时拼 content** — Sora 轮询：`cheap_seedance2` 且 completed 无 `video_url` 时，拼 `{base}/v1/videos/{id}/content`；相对路径 `video_url` 拼到 Base 源站
5. **鉴权下载** — `videoService.js`：`downloadVideoToLocal` 可选 Bearer；`finalizeSuccessfulVideo` 对该厂商要求本地下载成功，否则失败
6. **单测** — `test/cheapSeedance2Video.test.js` 映射与 content URL

## 验收

- 用户只填 Key 即可提交
- 时长落到 5/10/15
- 成片可落到 `/static` 并播放
