---
name: haina-cli
description: 通过 HAiNA CLI 发布视频/图片帖到 TikTok / TikTok Shop（挂车带货）、管理评论（回复/点赞/隐藏）、收发私信与自动消息、查数据洞察、接事件推送、查询坐席与商品、TikTok Shop 预审。Use when the user wants to upload or publish videos/photo posts to TikTok / TikTok Shop, manage comments (reply/like/hide), send direct messages or auto-messages, check insights/analytics, receive event webhooks, query seats or shoppable products, run TikTok Shop precheck, upload covers, search music, or check publish status via the HAiNA CLI API platform.
metadata:
  author: beervid
  version: "0.10.1"
---

# HAiNA CLI 视频发布

HAiNA CLI 是 TikTok / TikTok Shop 视频发布 API 平台。本 skill 教你用官方 CLI（HAiNA CLI，`haina` 命令）完成发布全流程。

## 0. 前置自举（每次会话先检查）

```bash
which haina || npm install -g @hainaio/cli   # 要求 Node ≥22
haina auth status                            # 退出码非 0 = 未配置凭据
haina version                                # 核对能力版本（见下「版本门」）
```

- 旧安装的命令名是 `vidgate`（@vidgate/cli），与 `haina` 完全等价——看到用户机器上只有 vidgate 时照用不误，但新安装一律 `@hainaio/cli`。
- 无凭据 → 提示用户提供 API Token（控制台「API Keys」页创建）：`haina auth login --token <t>`，或 `export HAINA_API_TOKEN=vg_live_...`（`VIDGATE_API_TOKEN` 为兼容别名）。

### 版本门（CLI × Skill 兼容）

**本 skill 对应 CLI ≥ 1.5.1**（skill 与 CLI 同版本发布，浮动通道——GitHub 镜像 / tarball 直链——永远是最新 skill）。

- `haina version` ≥ 1.5.1：全量能力可用（含 CML 选曲 `tt music trending` 与发布 BGM `--music-id` 系列）。
- CLI 1.5.0：除 discovery 族与发布 BGM 外的能力可用；**CML 选曲/BGM 需升级**。
- CLI 1.4.x：核心链路可跑（坐席/视频库/商品/TikTok / TikTok Shop 发布/预审/封面/音乐），但**没有**评论、私信、洞察、webhooks、events、listen、publish photo/hashtags/locations、discovery 这些 1.5+ 新增面——**建议直接升级**：`npm update -g @hainaio/cli`（或 `npm i -g @hainaio/cli@latest`）。
- **版本错配信号**：命令报「未知命令」（exit 1 + usage 输出）= 当前 CLI 不含该命令 → 先 `haina version` 确认，再引导升级；不要反复重试或猜测 flag。
- **命令树说明（1.5.1 起）**：正典命令带平台码（`tt video publish` / `tts video publish`…，见 `cli-commands.md`）；旧扁平命令（`publish tt` / `comments` / `insights`…）在 ≥1.5.1 上已更名——敲旧名会打出精确的新写法指引（exit 1 零请求）；旧 skill 的缓存副本教的是旧名，请 `haina skill install --yes` 刷新。
- 反向（CLI 新 + skill 旧）：**flag 级**兼容别名保留（`--share-id`/`--account-id` 等），但 1.5.1 起**命令名**已按平台分组更名——旧 skill 里的旧命令会被 CLI 打出精确新写法（exit 1 零请求），按指引改写或直接 `haina skill install --yes` 刷新 skill。
- 拿不准能力边界时，以 `haina --help` 输出为最终事实。

## 1. 凭据纪律（安全红线）

- Token 一律经环境变量或 `auth login` 注入；**禁止**写进任何源码、脚本、git 提交。
- Token 明文只在控制台创建时可见一次；丢失就引导用户重建，不要尝试找回。

## 2. 输出纪律

- **永远带 `--json`**：恒输出服务端 envelope `{ code, message, data }`；**以 `code === 0` 判成败**（HTTP 200 也可能是官方失败）。
- 退出码：0 成功 / 1 本地参数 / 2 认证 / 3 业务拒绝 / 4 限流配额 / 5 上游官方 / 6 内部。429 限流 CLI 已自动退避。
- 全局 flag（`--json` `--token` `--base-url` `--timeout`）可前可后。

## 3. 平台心智模型（先读，再动手）

- **坐席（Seat）**：1 个 TikTok 账号 = 1 坐席；一个坐席可绑两种能力。**TikTok 能力的账号 ID 是 businessId；TikTok Shop 能力的账号 ID 是 creatorUserOpenId**——两个 ID 都从 `haina seats list --json` 拿，别混用。
- **账号引用 flag**：新命令（comments/messages/insights/publish photo 等）一律 `--account`（username 或 businessId 均可）；存量命令（products/publish tts/precheck/photos/music）用 `--account-id`。
- **视频库分两库**：TikTok 库上传返回 `videoUrl`（公网 URL，**7 天有效**，发布用）；TikTok Shop 库上传返回 `fileId`（**一次性消耗**，发布/预审用）。
- **账号健康**：坐席 `authStatus=expired` 说明授权失效 → 引导用户去控制台坐席页重新绑定，不要反复重试发布。

## 4. 流程路由

### TikTok 发布（TikTok 普通视频）

```bash
haina seats list --json                    # 拿 TikTok 账号 businessId
haina videos upload <file> --json          # → videoUrl（7 天有效）
haina tt video publish --video-url <url> --account-id <id> --caption "文案 #话题" --wait --json
```

挂官方商用音乐（CML）BGM：**先选曲再发布**——

```bash
haina tt music trending --account <id> --genre POP --json   # 选曲：发布用 trendingSongClipId/fullSongClipId 列（不是 commercialMusicId）
haina tt video publish --video-url <url> --account <id> --music-id <songClipId> \
  --music-volume 50 --original-sound-volume 100 --wait --json
```

### TikTok Shop 挂车（TikTok Shop 带货视频）

```bash
haina seats list --json                                        # 拿 TikTok Shop 账号 creatorUserOpenId
haina videos upload <f> --library tts --account-id <id> --json # → fileId（一次性！）
haina tts product query --account-id <id> --json                  # 拿可挂车 productId
haina tts precheck submit --file-id <f> --account-id <id> --product-id <p> --product-title <锚点> --wait --json  # 可选但建议
haina tts video publish --file-id <f> --account-id <id> --product-id <p> --product-title <锚点> --wait --json
```

### 图片帖（TikTok Photo Post，1-35 张公网 https 图）

```bash
haina tt location search --account <id> --query "New York" --json   # 可选：拿 locationId 挂地点
haina tt photo publish --account <id> --photo-urls "https://…/1.jpg,https://…/2.jpg" \
  --privacy-level PUBLIC_TO_EVERYONE --caption "文案 #话题" --wait --json
```

### 评论管理（回复/点赞/隐藏/删除）

```bash
haina tt comment list --account <id> --video-id <postId> --json
haina tt comment reply --account <id> --video-id <postId> --comment-id <cid> --text "回复内容" --json
haina tt comment like --account <id> --comment-id <cid> --action LIKE --json
```

### 私信与自动消息

```bash
haina tt message conversations --account <id> --type SINGLE --json              # 会话列表
haina tt message send --account <id> --conversation-id <cid> --text "你好" --json
haina tt message auto get --account <id> --type WELCOME_MESSAGE --json          # 自动消息（欢迎语/建议问题/聊天提示词）
```

### 数据洞察（发布效果回收）

```bash
haina tt insight account --account <id> --json                       # 账号概览
haina tt insight videos --account <id> --video-ids <postId> --json   # 视频指标定点查（publish stats 的替代）
```

### 事件推送（评论/私信/发布状态实时通知）

```bash
haina listen --events "tiktok.*" --json        # 本机/开发：SSE 长连收事件，无需公网
haina webhooks create --url https://<你的回调> --events "tiktok.*" --json   # 生产：push 到自有端点（secret 只返回一次，妥善保存）
```

做「收到评论自动回复」类机器人前，必读 `references/cli-commands.md` 事件节的防自回复死循环要点。

**深入细节按需读 references/**：
- `references/cli-commands.md` — 全部命令与 flag 完整参考
- `references/tt-publish.md` — TikTok 视频规格/官方限频/字段语义/URL 验证/草稿/数据回收
- `references/tts-publish.md` — fileId 规则/商品分页/预审双 check 读法/封面与音乐
- `references/tiktok-engagement.md` — 评论与私信规则（权限门槛/发送窗口/自动消息审核状态机）
- `references/tiktok-insights.md` — 数据洞察规则（T+24~48h 延迟/100 粉门槛/字段 scope 分级）
- `references/events.md` — 事件推送（订阅/验签代码/at-least-once 消费语义/防自回环军规）
- `references/errors.md` — 错误码 × 退出码 × 处置动作全表
- `references/concepts.md` — 平台概念与限制全集（有效期/配额/限流）

## 5. TikTok Shop 四条硬规则（官方规则，违反必失败）

1. **fileId 一次性消耗**：发布调用即消耗，**失败也不可复用**；重发必须重新上传。
2. **fileId 绑定上传账号**：upload 与 publish 的 `--account-id` 必须一致。
3. **锚点文案（productTitle）**：≤30 字符，不含标点和 emoji（官方校验，违规透传 3001）。
4. **预审可选但建议**：配额 50 次/天/账号；violation FAIL 不建议发布（是否发布由用户决定并承担后果）。

## 6. 通用排错姿势

1. 先看退出码定位错误类别，再读 envelope 的 `message`（3001 时含官方原因）。
2. `code=1005` 先怀疑账号/资源归属：用 `seats list` 核对 ID 口径（businessId vs creatorUserOpenId）。核对无误仍 1005 → **停止重试**（归属判定不因重试改变），携带 `seats list` 输出与请求参数联系平台排查。
3. 发布失败（`publish_failed`）读 `failReason` / `beervid.reason` 向用户解释官方原因，不要自行猜测。
4. 完整处置表见 `references/errors.md`。

## 7. 无 CLI 降级

用户拒绝安装 CLI 时：优先按本 skill 包内 `references/endpoints.md` 直连 HTTP（端点/参数/错误码速查，与 openapi spec 同源，skill 单独安装也自洽）；envelope 与错误码语义同上（`code === 0` 判成败）。控制台文档站「API 参考」页（/docs，免登录）可作补充参考。
