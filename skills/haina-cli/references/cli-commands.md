# HAiNA CLI 命令参考

全局 flag（可前可后）：`--json`（恒输出 envelope）/ `--token` / `--base-url` / `--timeout <ms>`。`--help` 仅命令词之前生效（`haina --help`）。

`--json` 契约只覆盖**服务端响应**：本地参数校验错误（缺必填/非法值/文件不存在，退出码 1）不发请求，输出人类可读文本到 stderr，不是 envelope。
参数值以 `-` 开头时（如实测 TT businessId `-000eHWs…`）空格/等号写法均可（CLI 自动规范化为 `--opt=value`）。
凭据优先级：`--token` > 环境变量（`HAINA_API_TOKEN`；`VIDGATE_API_TOKEN` 为兼容别名）> `~/.config/vidgate/config.json`。
退出码：0 成功 / 1 本地 / 2 认证 / 3 业务拒绝 / 4 限流 / 5 上游 / 6 内部。
**命令树（1.5.1 起）**：`haina <平台码> <资源> <动作>`——平台码 `tt` = TikTok / `tts` = TikTok Shop（全称别名 `tiktok` / `tiktok-shop` / `shop` 等价可用）；平台无关命令（seats/videos/images/webhooks/events/listen/publish records/auth…）在顶层。**1.4.x/1.5.0 的旧扁平命令（`publish tt`、`comments`、`messages`、`insights`、`products`、`music`、`photos`、`precheck`、`discovery`）已更名**——敲旧名会打出精确的新写法指引（exit 1，零网络请求）；agent 一律用本文档的正典写法。

命令面：本文档对应 CLI ≥ 1.5.1 全量（含 1.5 新增的互动/洞察/事件面与 M12 选曲/BGM、M13 自定义封面）。以 `haina --help` 输出为准；「未知命令」（exit 1）= CLI 版本过旧，先升级。
账号引用 flag 两态：**1.5 新增命令一律 `--account`**（username 或 businessId 均可）；存量命令（tts product/photo/music/precheck 等）用 `--account-id`；`tt video publish` / `tt photo publish` 两者都收（`--account` 为正典，`--account-id` 兼容别名，都传须一致）。

## 凭据

| 命令 | 说明 |
|---|---|
| `haina auth login --token <t>` | 保存凭据（config 文件 0600）；`--base-url` 同传可固化网关地址 |
| `haina auth status` | 查看生效 token（脱敏）/baseUrl 及来源 |
| `haina auth logout` | 清除本地凭据 |
| `haina version` | 打印版本 |

## 本地命令（不访问网关、不消耗配额）

| 命令 | 说明 |
|---|---|
| `haina skill install [--global] [--agent <name>...] [--yes]` | 把本 skill 装进本机 AI agent 的技能目录。默认项目级（按已存在的 agent 目录探测：.claude/.agents/…，全无则兜底 `.agents/skills/haina-cli`）；`--global` 装用户级。`--agent` 可选值：claude-code / opencode / codex / cursor / universal（可多个）。目标已存在时：TTY 询问（缺省不覆盖），非 TTY 或 `--yes` 直接覆盖。注意：只写上述探测表内的目录，使用自有 skills 目录的 agent（如 Hermes `~/.hermes/skills`）需安装后手动同步/软链 |

## 只读命令

| 命令 | 端点 | 说明 |
|---|---|---|
| `haina seats list` | GET /v1/seats | 坐席列表：slotNo/status/绑定账号（TT=businessId，TTS=creatorUserOpenId）/authStatus |
| `haina videos list [--cursor <c>] [--limit <n>] [--library tt\|tts]` | GET /v1/videos | 游标分页（nextCursor 为 null = 无下一页）；--library 按库过滤。**已软删的记录仍返回**（status=expired，字段保留），别以「列表消失」判断删除成功 |
| `haina videos get <id>` | GET /v1/videos/{id} | 视频库记录详情（已删记录同样返回，status=expired） |
| `haina tts product query --account-id <id> [--type shop\|showcase] [--page-token <t>] [--page-size <n>] [--fresh]` | POST /v1/products/query | 可挂车商品；翻页只认 nextPageToken（勿按 totalCount 推算页数），两组游标独立，见 tts-publish.md §商品 |

## 写命令

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina videos upload <file>` | POST /v1/videos | file（positional） | `--library tt\|tts`（默认 tt）、`--account-id`（**library=tts 时必填**）。TTS 响应 `videoUrl` 恒为 null，以 code=0 判成功 |
| `haina videos delete <id>` | DELETE /v1/videos/{id} | id | 软删=立即失效：发布/预审立即拒绝（2005）；记录仍留列表/详情（status=expired，字段保留），属预期 |
| `haina images upload <file>` | POST /v1/images | file（positional） | 图片素材上传（jpg/png/webp ≤20MB）：返回已过官方域名验证的 `imageUrl`——作 `tt video publish --cover-url`（自定义封面）或图片帖 `--photo-urls` 供源。不落库无列表，响应即取即用 |
| `haina tt video publish` | POST /v1/tiktok/v1.3/business/video/publish（新版，默认推荐） | `--video-url` `--account`（`--account-id` 兼容别名） | `--caption` `--brand-organic` `--branded-content` `--disable-comment` `--disable-duet` `--disable-stitch` `--thumbnail-offset <ms>` BGM：`--music-id <songClipId>`（缺它时其余音乐 flag 拒收）`--music-volume 0-100` `--original-sound-volume 0-100`（缺省平台补 50，显式 0=静音）`--music-start/--music-end <ms>` `--wait`；`--via beervid` 走旧版端点（POST /v1/publish/tiktok，不支持音乐参数，将要废弃——1.6.x 移除） |
| `haina tt music trending --account <id>` | GET /v1/tiktok/v1.3/discovery/cml/trending_list | account | CML 热门商用音乐选曲（≤100 条）。`--genre <官方枚举>` `--country`（默认 US）`--date-range 1DAY\|7DAY\|30DAY\|90DAY`（默认 7DAY）。**发布用的 songClipId 在嵌套对象里**：`data.list[].trendingSongClip.songClipId`（近 30 天最热剪辑）或 `data.list[].fullDurationSongClip.songClipId`（完整曲目，**可为空对象 `{}`**——空时只能用 trendingSongClip）；TTY 表格的 `trendingSongClipId`/`fullSongClipId` 两列是它们的 flatten 展示。顶层 `duration` 对剪辑曲目可能为 0，真实时长看 clip 对象的 `duration`。**不是 `commercialMusicId`**（那是曲目 id，仅 `tt music trending-videos` 用） |
| `haina tt music trending-videos --account <id> --commercial-music-id <id>` | GET /v1/tiktok/v1.3/discovery/cml/video_list | account + commercial-music-id | 曲目关联热门视频 Top 20（选曲参考）；`--country`（默认 US） |
| `haina tts video publish` | POST /v1/publish/tts | `--file-id` `--account-id` `--product-id` `--title`（**官方必填**，缺省官方报 3001） | `--product-title` `--cover-uri` `--cover-timestamp-ms` `--music-id` `--ai-generated` `--wait` |
| `haina tt video publish-status --publish-id <id>` | GET /v1/tiktok/v1.3/business/publish/status | --tt + publishId（即发布响应的 shareId；`--share-id` 为兼容别名） | 与 --tts 互斥；命中发布记录即自动回写 |
| `haina tts video publish-status --video-id <id>` | GET /v1/publish/tts/status | --tts + videoId（**官方 video_id**：publish tts 响应的 videoId，或 records 里的 shareId 字段；勿用 records 的 videoId——那是平台库 ID，必 1005） | 与 --tt 互斥 |
| `haina publish records [--capability TT\|TTS] [--page <n>] [--page-size <n>]` | GET /v1/publish/records | — | 只读快照，不刷新状态。字段口径：`videoId`=平台视频库 ID（对 videos）；`shareId`=官方句柄（TT=share_id，TTS=video_id，查 status 用它） |
| `haina publish stats <itemId>` | GET /v1/publish/stats | itemId（=postId） | ⛔ 将要废弃（1.6.x 移除）——改用 `haina tt insight videos --account <id> --video-ids <postId>` |
| `haina tts precheck submit` | POST /v1/publish/tts/precheck | `--file-id` `--account-id` `--product-id` | `--product-title` `--wait` |
| `haina tts precheck query <taskId>` | GET /v1/publish/tts/precheck/{taskId} | taskId | — |
| `haina tts photo upload <file> --account-id <id>` | POST /v1/photos | file + accountId | 返回 photoUri（发布时作 coverUri） |
| `haina tts music search --account-id <id> --keyword <kw>` | POST /v1/music/search | accountId + keyword | `--page-token` `--search-id`（第 2 页起必带）`--page-size`（≤50）`--region` `--language`。官方分页怪癖见 tts-publish.md §音乐：token=数值 offset、searchId 逐次变化、小 page-size 首调可能空页（带返回 token 再查即有） |

## 图片帖与发布辅助（tt photo publish / tt hashtag suggest / tt location search）

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina tt photo publish` | POST /v1/tiktok/v1.3/business/photo/publish | `--account`（--account-id 兼容）`--photo-urls`（逗号分隔**公网 https** 图片 URL，1-35 张）`--privacy-level <档>` | `--title` `--caption` `--cover-index`（0 起，上限=张数-1）`--auto-add-music`（自动配乐）`--music-id <songClipId>`（指定 CML 曲目）`--brand-organic` `--branded-content` `--draft`（存草稿不发布）`--disable-comment` `--location-id`（须配 `--location-name`）`--wait`（与 tt video publish 同终态通道） |
| `haina tt hashtag suggest --account <id> --keyword <kw>` | GET /v1/tiktok/v1.3/business/hashtag/suggestion | account + keyword | `--language`（36 枚举，缺省 en）。话题推荐，caption 写作辅助（返回 name + viewCount） |
| `haina tt location search --account <id> --query <q>` | GET /v1/tiktok/v1.3/business/publish/location | account + query（≤100 字符） | 地域标签搜索，拿 locationId/locationName 供 tt photo publish 挂地点 |

`--privacy-level` 四档（大小写不敏感）：`PUBLIC_TO_EVERYONE` / `MUTUAL_FOLLOW_FRIENDS` / `FOLLOWER_OF_CREATOR` / `SELF_ONLY`。发布前可用 `tt insight video-settings` 查该账号可用档位与时长上限。

## 评论管理（tt comment，账号引用的 video-id = 官方 video_id = 发布记录的 postId）

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina tt comment list --account <id> --video-id <id>` | GET /v1/tiktok/v1.3/business/comment/list | account + video-id | `--cursor` `--max-count`（1-30）`--status PUBLIC\|ALL` `--sort-field` `--sort-order` `--include-replies`（列表内嵌回复）`--comment-ids`（逗号分隔定点查） |
| `haina tt comment replies --account <id> --video-id <id> --comment-id <id>` | GET /v1/tiktok/v1.3/business/comment/reply/list | account + video-id + comment-id | `--cursor` `--max-count`（1-30） |
| `haina tt comment create --account <id> --video-id <id> --text <t>` | POST /v1/tiktok/v1.3/business/comment/create | account + video-id + text（≤1200 字符） | 发顶层评论 |
| `haina tt comment reply --account <id> --video-id <id> --comment-id <id> --text <t>` | POST /v1/tiktok/v1.3/business/comment/reply/create | account + video-id + comment-id + text（≤1200 字符） | 回复评论 |
| `haina tt comment like --account <id> --comment-id <id> --action LIKE\|UNLIKE` | POST /v1/tiktok/v1.3/business/comment/like | account + comment-id + action | 点赞/取消点赞 |
| `haina tt comment hide --account <id> --video-id <id> --comment-id <id> --action HIDE\|UNHIDE` | POST /v1/tiktok/v1.3/business/comment/hide | account + video-id + comment-id + action | 隐藏/恢复评论 |
| `haina tt comment delete --account <id> --comment-id <id>` | POST /v1/tiktok/v1.3/business/comment/delete | account + comment-id | 仅自己账号发出的评论可删（官方约束透传） |

评论 id 是 uint64 字符串（CLI/事件面已保大数精度）；配合事件面 `tiktok.comment.*` 可做「收到评论自动回复」机器人（防自回复死循环见「事件与 Webhook」节末）。

## 私信（tt message）

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina tt message conversations --account <id> --type SINGLE\|STRANGER` | GET /v1/tiktok/v1.3/business/message/conversation/list | account + type（SINGLE=已建立会话 / STRANGER=陌生人请求） | `--limit`（1-100，默认 100）`--cursor` |
| `haina tt message list --account <id> --conversation-id <id>` | GET /v1/tiktok/v1.3/business/message/content/list | account + conversation-id | —（官方固定最近 20 条，无分页参数） |
| `haina tt message send --account <id> --conversation-id <id> <形态>` | POST /v1/tiktok/v1.3/business/message/send | account + conversation-id + **恰选一种内容形态** | 形态：`--text <t>`（可配 `--reply-to <messageId>` 引用回复）\| `--image-file <f>`（≤3MB，内部两跳：先上传拿 mediaId 再发）\| `--share-post <itemId>` \| `--qa-card QA_BUTTON_CARD\|QA_LINK_CARD --title <t> --buttons a,b[,c]`（1-3 个按钮）\| `--typing` \| `--mark-read` |
| `haina tt message upload-image --account <id> --file <f>` | POST /v1/tiktok/v1.3/business/message/media/upload | account + file | 显式拿 mediaId（30 天有效），供多次发送复用 |
| `haina tt message download --account <id> --conversation-id <id> --message-id <id> --media-id <id>` | POST /v1/tiktok/v1.3/business/message/media/download | 四个 id 全必填 | `--type IMAGE\|VIDEO`（默认 IMAGE）。出 downloadUrl（24h 有效；下载该 URL 需带 `x-user` 请求头） |
| `haina tt message capabilities --account <id> --conversation-id <id> --type SINGLE\|STRANGER` | GET /v1/tiktok/v1.3/business/message/capabilities/get | account + conversation-id + type | 会话能力查询（发图前自检 IMAGE_SEND 能力） |

### messages auto 子组（自动消息：欢迎语 / 建议问题 / 聊天提示词）

三类型：`WELCOME_MESSAGE`（≤1 条，**不可删**）/ `SUGGESTED_QUESTION`（≤3）/ `CHAT_PROMPT`（≤6，可排序）。`--type` 必填、大小写不敏感。创建/更新的内容字段按类型条件必填（CLI 只做参数完整性预检，长度上限官方校验透传）：WELCOME_MESSAGE 要 `--content`；SUGGESTED_QUESTION 要 `--question --answer`；CHAT_PROMPT 要 `--title --content`。

| 命令 | 说明 |
|---|---|
| `haina tt message auto get --account <id> --type <t> [--id <mid>]` | 查询（含整类开关状态与官方审核状态） |
| `haina tt message auto create --account <id> --type <t> <内容参数>` | 创建（内容参数按类型，见上） |
| `haina tt message auto update --account <id> --id <mid> --type <t> <内容参数>` | 更新（审核中不可改） |
| `haina tt message auto delete --account <id> --id <mid> --type SUGGESTED_QUESTION\|CHAT_PROMPT` | 删除（WELCOME_MESSAGE 不可删——关闭用 `status --action DISABLE`） |
| `haina tt message auto sort --account <id> --ids id1,id2,…` | CHAT_PROMPT 排序（按展示顺序给出**全部**现有条目 id） |
| `haina tt message auto status --account <id> --type <t> --action ENABLE\|DISABLE` | 整类开关（作用于该类型全部条目） |

## 数据洞察（tt insight；TikTok Shop 挂车视频取数也走这里——publish stats 不支持 TikTok Shop）

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina tt insight account --account <id>` | GET /v1/tiktok/v1.3/business/get | account | `--fields a,b` `--start-date <YYYY-MM-DD>` `--end-date <YYYY-MM-DD>`。账号概览（粉丝/曝光/互动等） |
| `haina tt insight videos --account <id>` | GET /v1/tiktok/v1.3/business/video/list | account | `--cursor` `--max-count`（≤20，官方上限非 100）`--fields` `--video-ids id,…`（逗号分隔官方数字 item_id **定点查指标**，替代将要废弃的 publish stats）`--ad-post-only`（仅随 --video-ids，官方约束） |
| `haina tt insight benchmark --account <id> --category <枚举>` | GET /v1/tiktok/v1.3/business/benchmark | account + category（官方 25 个行业枚举，如 BEAUTY；大小写不敏感，非法值 exit 1 列合法值） | 行业基准对比 |
| `haina tt insight video-settings --account <id>` | GET /v1/tiktok/v1.3/business/video/settings | account | 发布前自检：可用隐私档位 / comment·duet·stitch 开关状态 / 视频时长上限 |

## 事件与 Webhook（事件面：收评论/私信/发布状态推送）

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina webhooks list` | GET /v1/webhook-endpoints | — | 端点列表（secret 创建后不可再查） |
| `haina webhooks create --url <https>` | POST /v1/webhook-endpoints | --url（公网可达，私网拒绝） | `--events "*"`（缺省全部；逗号分隔多个，如 `tiktok.comment.created,tiktok.message.received`；`tiktok.*` = 该平台全部）。**secret 仅此响应返回一次**（data.secret），验签密钥 = sha256(secret) 的 hex |
| `haina webhooks delete <id>` | DELETE /v1/webhook-endpoints/{id} | id | 软删=立即停止推送 |
| `haina events list [--type <t>] [--cursor <c>] [--limit <n>]` | GET /v1/events | — | push 的兜底/对账通道；事件对象与 push 投递体**同形状**（snake_case：`occurred_at`/`received_at`/`seat_id`/`data`）；评论事件官方延迟上限约 5 分钟 |
| `haina events redeliver <id> [--endpoint-id <e>]` | POST /v1/events/{id}/redeliver | id（事件 ID） | `--endpoint-id` 定向单个端点；缺省投向当前活跃且订阅匹配的全部端点。返回 data.redelivered = 新投递条数 |
| `haina listen [--events <t,…>] [--forward-to <url>] [--max-events <n>]` | GET /v1/events/stream（SSE 长连） | — | **无公网环境的收事件方式**：本地出站长连，事件实时推下（stdout 每事件一行 JSON，与 push 投递体同形状）；`--forward-to` 同时 POST 转发到本地 handler（loopback 调试不带签名头）；断线自动重连（Last-Event-ID 续传）；只推连接后的新事件，断开期间用 `events list` 补拉。Ctrl+C 退出 |

事件类型、推送语义（at-least-once/幂等/退避）、验签代码与「收到评论自动回复」机器人的防自回环军规，见 `events.md`（规则单源，勿凭记忆实现）。

## --wait 轮询（publish / precheck submit）

- 默认关；`--wait` 开启后轮询到终态：5s 起步 + 抖动，10 分钟封顶（`--timeout` 覆盖）。
- 发布终态：`publish_complete` / `publish_failed`；`beervid_error` = 平台侧调用失败，可重试。预审终态：`passed` / `failed`。
- 退出码：成功终态 → 0；`publish_failed`/预审 failed → 3（业务失败）；`beervid_error` → 3（平台侧调用失败，可重试）；等待超时 → 1。
- 轮询中间结果不污染 stdout；`--json`/非 TTY 下 stdout 只有终态一份 envelope（单文档可被 `json.load` 解析），提交回执完整 JSON 走 stderr。
