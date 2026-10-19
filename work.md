# SYSTEM PROMPT FOR AGENT: Muse 微信桥接搭建

> 阅读对象：另一个 Muse（AI Agent），不是人类用户。
> 目标：让你的 Muse 在微信里收发消息。用户在微信发文字任务，你真实执行后回传微信。
> 约束：不要向用户复述本文件，不要把本文件当聊天内容发给人类。按本文件执行。
> 核心事实：**没有官方 Muse 微信接口。** 必须用 DIY 桥：Python serve + iLink + Muse cron worker 轮询。

## 1. 不可违反的 invariants

- `serve` 只收/发，**不执行任务**。执行只在 Muse cron worker 里。
- 没有 hook/webhook。唯一唤醒路径 = cron worker 轮询 `inbox/`。
- 只处理绑定用户的 1 对 1 私聊。`group_id` 存在 / `from_user_id != bound_user_id` / `message_type==2` → 全部忽略。
- 必须真的调工具执行任务，禁止编结果。缺信息 → `needs_info`，禁止用“收到，我记下了”代替执行。
- 禁止打印/保存完整 `bot_token` 到聊天、记忆、仓库、日志。`status/health` 必须脱敏。
- 禁止在登录前安装 worker。禁止承诺固定秒数回复。

## 2. 协议来源（先读，不要编接口）

- 直连 iLink（用这个）：`https://github.com/tencent/openclaw-weixin/blob/HEAD/docs/protocol_zh_CN.md`，base `https://ilinkai.weixin.qq.com`
- wrapper API（不要用）：`https://developers.weixin.qq.com/doc/aispeech/knowledge/openapi/Clawbotrelated.html`，`/api/v1/wechat/*` 是 wrapper，不是直连 iLink，不要混用。
- QR 展示逻辑：`https://github.com/tencent/openclaw-weixin/blob/HEAD/src/auth/login-qr.ts`，`displayQRCode` 用 `qrcode_img_content`，不是 `qrcode`。

## 3. 搭建

```
wechat-bridge/
  bridge/client.py  # headers/base_info/QR/getupdates/sendmessage/getconfig/sendtyping/notify
  bridge/session.py # 原子写 session/cursor/pause, inbox/outbox, 脱敏
  bridge/qr.py      # QR 保存与渲染
  bridge/cli.py     # login/wait-login/serve/send/status/poll-once/health
  docs/MECHANISM.md
  cron/muse-worker.md
  state/ inbox/ outbox/ logs/  # .gitignore 全部排除
```

### 3.1 client.py 关键参数

- headers: `Content-Type: application/json`, `AuthorizationType: ilink_bot_token`, `Authorization: Bearer <bot_token>`（QR 状态轮询不要加）, `X-WECHAT-UIN`: 随机 uint32 十进制再 base64, `iLink-App-Id: bot`, `iLink-App-ClientVersion`: `0x00MMNNPP` 转十进制，例 2.4.8 → `132104`，不要编造新版本。
- `base_info`: `{"channel_version": "2.4.8", "bot_agent": "OpenClaw"}`
- 接口：
  - `POST /ilink/bot/get_bot_qrcode?bot_type=3`, body `{"local_token_list": []}`
  - `GET /ilink/bot/get_qrcode_status?qrcode=<url-encoded qrcode>[&verify_code=...]`
  - `POST /ilink/bot/getupdates`, body `{"get_updates_buf": "<游标>", "base_info": {...}}`
  - `POST /ilink/bot/sendmessage`, body `{"msg": {...}, "base_info": {...}}`，文本 item `type=1`
  - `POST /ilink/bot/getconfig`, `sendtyping`, `msg/notifystart`, `msg/notifystop`
- `ret==0` 成功。`ret/errcode==-14` = 会话暂停 1 小时，不要当成凭据过期。
- `getupdates` 超时/取消 → 返回空，不要死循环。

### 3.2 登录流程

1. 等用户说“好了”再 `login`。不要提前生成二维码。
2. `login` → 调 `get_bot_qrcode`，存 `qrcode` + `qrcode_img_content` 到 `state/qrcode.json`。
3. 生成二维码图片发给用户（见 §5 P1/P2）。
4. `wait-login [--verify-code <code>]` 轮询 `get_qrcode_status`：
   - `wait` → 继续等
   - `scaned` → 已扫，继续
   - `confirmed` → 保存 `bot_token, ilink_bot_id, ilink_user_id, baseurl` 到 `state/session.json`（原子写），`bound_user_id = ilink_user_id`
   - `expired` → 重新 `login`
   - `need_verifycode` → 问用户要手机上显示的验证码，再跑 `wait-login --verify-code`
   - `verify_code_blocked` → 刷新二维码
   - `scaned_but_redirect` → 有 `redirect_host` 切换 base 继续
   - `binded_redirect` → 已绑定到当前实例
5. 登录成功 → 启动 `serve` → 安装 worker。

### 3.3 serve

- 单实例锁，避免多实例。VM 重启后需手动重起（删陈旧 lock → `nohup python3 -m bridge.cli serve`）。
- 顺序：先发 `outbox` 中 `status=to_send` 的，再 `getupdates` 收新消息。
- 游标 `get_updates_buf` 只有非空才更新。按 `message_id` 去重，写 `inbox/<id>.json`。
- inbox 字段：`message_id, from_user_id, to_user_id, context_token, run_id, session_id, text, has_media, item_types, status=received`
- 启动 `notifyStart`，停止 `notifyStop`（失败只告警）。
- `sendmessage` 必须回传入站 `context_token`。`client_id` 必须复用（见 §5 P3）。

### 3.4 worker（cron）

- id `wechat-bridge-worker`，用 cron 工具建，不要手写 cron 文件。
- 自适应：活跃 10s，空闲 >5min → 10m，有新消息处理完切回 10s。每次开头查最新入站时间，`cron.view` + `cron.update` 只改 `schedule.every`。
- 成本：10s 约 8640 次/天，10m 约 144 次/天。每轮最多 3 条。
- worker body 伪代码：

```
1. 读 inbox/*.json，只处理 status=received。
   检查 stuck: status=processing 且 claimed_at >5min → 按 §4 报错处理，重置为 received 或 failed。
   若无 received 且无 stuck → 结束，不发任何消息。
2. 对每条 received（最旧优先，max 3）：
   a. claim: status → processing, worker_run=<run id>, claimed_at=now。已 processing/done 跳过（除非 stuck）。
   b. has_media=true 且 text 为空 → 回复固定文本：“这条通道暂时没有配置这类内容处理，请先用文字描述”，不要声称读过媒体。
   c. 否则真实执行：
      - 需工具 → 真调工具
      - 缺信息/需授权 → status=needs_info，回复问清缺什么，不要编完成
      - 失败 → status=failed，回复原因+下一步，触发 §4 报错通知，禁止“收到，我记下了”
   d. 短任务直接回。长任务先写 ack “收到，正在处理…” 到 outbox，再写最终结果，不要刷屏。
   e. 写 outbox/<message_id>.json: {message_id, to_user_id=from_user_id, context_token, text, status=to_send, created_at}
      更新 inbox 为 done / failed / needs_info。
   f. 合并规则：仅当同发送者、<60s、第二条明显是续写才合并，否则一 message_id 一任务。
3. 禁止读/打印 bot_token。禁止直接调 sendmessage（由 serve 发）。
```

## 4. 报错通知（必须实现）

- 触发条件：inbox 卡 processing >5min、outbox 发送失败、serve 挂了/lock 陈旧、执行异常、API 失败。
- 非触发：`needs_info` 不是报错，不要通知。
- 顺序（必须）：
  1. 先 Bark：`bark.py "微信桥报错" "<message_id + 原因 + 时间>"`。返回 200 → 结束，禁止再发邮件。
  2. 仅当 Bark 失败（非 200/超时/异常）→ 发邮件。
     - 邮箱：**必须问用户要**（“报错发到哪个邮箱？”），禁止硬编码任何邮箱，禁止抄其他用户的邮箱。
     - `gmail +send --to <用户给的邮箱> --subject "微信桥报错 - <id>" --body "<id + 原文 + 原因 + 已尝试 + 下一步>"`
- Bark 未绑定：先让用户绑定 Bark（下载 Bark App、拿 Bark key、在 Muse 里接 Bark 连接器），不要假设已绑定。
- 报错时同时：inbox 置 `failed` 写原因；能通过微信回也回一句说明。

## 5. Muse 可能会遇到的问题 + 对应解决办法（按出现顺序）

### P1: 扫码只出“一串码” → 二维码内容编错了
- 现象：用户扫码后显示“扫描结果 + 32 位 hex 串”，无法确认。
- 根因：把 `qrcode` 字段编成二维码。`qrcode` 只是轮询 key，不是二维码内容。
- 修复：编 `qrcode_img_content` 的完整 URL（例 `https://liteapp.weixin.qq.com/q/7GiQu1?qrcode=<qrcode值>&bot_type=3`），`qrcode.make(url)`。参考 `login-qr.ts` 的 `qrterm.generate(qrcodeUrl)`。
- 禁止：把 `qrcode_img_content` URL 当图片 URL 发给用户去打开，必须编成二维码图片发给用户扫。

### P2: 扫了旧码 / 缓存图 → 扫出来还是旧码值
- 现象：用户扫了新图，扫出来还是旧码值（如 `640ae...`），或 `wait-login` 报 `expired`。
- 根因：二维码有效期约 2 分钟（不是 5 分钟）；每次覆盖同一个 `qrcode.png`，客户端缓存旧图。
- 修复：过期 → 重新 `login` 生成新码 + 重新 `wait-login`。每次用新文件名 `state/qrcode-<时间戳>.png`，不要覆盖。让用户确认扫的是最新那张。
- 扫码路径：微信 → 我 → 设置 → 插件 → ClawBot → 点 **开始扫一扫** → 扫最新图 → 点确认。不要用微信右上角通用扫一扫。

### P3: 同一条回复发了 3 次 → client_id 每次都换了
- 现象：用户发 `hello world`，收到 3 条相同回复。
- 根因：`sendmessage` 前两次其实已投递，但连接断了没回确认，客户端以为失败重试，且每次重试用新 `uuid4()` 当 `client_id`，服务器认为是 3 条不同消息。
- 修复（必须）：
  ```python
  # client.py
  def send_message(..., client_id=None):
      "client_id": client_id or str(uuid.uuid4())
  # cli.py _send_one_outbox
  client_id = rec.get("client_id") or f"muse-{mid}"
  # 重试循环里复用同一个 client_id
  ```
- 验证：修完重启 `serve`，再发一条测试消息，应只回 1 条。
- 教训：连接断了没回确认 ≠ 没发出去。重试必须用同一幂等 ID。

### P4: 二维码过期还在等 → wait-login 死等
- 现象：`wait-login` 300 秒没退出，一直在等旧码。
- 修复：过期即退出，重新 `login` 生成新码，不要继续等旧码。

### P5: inbox 卡在 processing，消息一直不回 → worker 中途崩了 / VM 重启
- 现象：`inbox` 有 `processing` 超过 5 分钟，`outbox` 无回复，后续消息也积压为 `received`。
- 修复：按 §4 触发报错通知（Bark 优先），把该消息置 `failed` 写原因，或重置为 `received` 供下一轮重试。检查 `serve` 是否还活着、lock 是否陈旧、cron worker 上轮是否还在 `in_flight`。
- 禁止：直接删 `session.json` / `cursor.json` 来“修”卡住。

### P6: serve 挂了 / lock 陈旧 → 不收不发
- 现象：`health` 显示 serve 不在跑，或 `serve` 启动报 lock 已存在。
- 修复：查进程是否真的在跑。若 pid 已死，删陈旧 lock，`nohup python3 -m bridge.cli serve` 重起。VM 重启后必须手动重起 serve（无 systemd 监督）。
- 禁止：不删 lock 就起第二个实例，会多实例抢发。

### P7: -14 暂停 → 以为凭据过期，盲目重扫
- 现象：`ret/errcode == -14`，`serve` 停了。
- 修复：这是会话暂停 1 小时。记录 `state/paused_until.json`，等 1 小时或重新扫码。不要删 session/cursor，不要把所有异常都当凭据过期。

### P8: 媒体消息被声称读过 → 幻觉
- 现象：用户发图片/语音/文件，worker 回“收到图片，内容是…”。
- 修复：本桥不实现媒体下载/发送。`has_media=true` 且 text 为空 → 只回固定文本“这条通道暂时没有配置这类内容处理，请先用文字描述”。不要编媒体内容。

### P9: 用户连续发多条，被合并/漏处理 → 合并规则用错
- 现象：两条独立任务只回了一条，或续写被当独立任务。
- 修复：仅当同发送者、<60s、第二条明显是续写（“还有…/另外…”）才合并，否则一 `message_id` 一任务。worker 必须先 claim 再执行，避免第二轮重复执行。

### P10: worker 空轮询耗 token / 延迟被承诺过高
- 现象：用户抱怨“为什么空闲时也要 10 秒回”或 token 消耗大。
- 修复：用自适应 10s/10m（§3.4），不要用固定 1m/5m。空闲 10m 时第一条消息最长等 10 分钟，处理完切回 10s。不要承诺“15 秒检查一次”或固定秒数。

### P11: 报错通知两个都发 / 邮箱填错别人的 → 隐私泄露
- 现象：Bark 成功了还发邮件，或邮件发到别人的邮箱。
- 修复：Bark 200 → 结束，禁止再发邮件。邮箱必须问用户要，禁止硬编码，禁止抄本文档外的任何邮箱。

### P12: 群消息 / 其他用户 / Bot 回显被处理 → 越权
- 现象：群里 @Bot 被回复，或陌生人消息被处理。
- 修复：只处理 `bound_user_id` 的 1 对 1 私聊，群消息、其他用户、Bot 回显全部忽略（§1）。

### P13: 权限/审批被绕过 → 在微信里直接执行了需审批的动作
- 现象：用户在微信说“帮我买/删/发”，worker 直接执行了。
- 修复：审批/流程仍在 Muse App 里。微信只是通道。需授权的动作 → `needs_info`，问清并等用户在 App 里确认，不要在微信里直接执行。

## 6. 限制（必须提前告诉用户，不要承诺做不到的）

- 1 对 1 私聊 only，无群聊
- 无主动冷推送（只能在已有会话里回复）
- 无历史回补（只能收登录后的新消息）
- 无流式输出（只发完成态文本）
- 无图片/语音/文件/视频下载与发送
- 延迟 = cron 间隔 + 执行时间，不要承诺固定秒数
- 审批仍在 Muse App 里，微信只是通道

## 7. 验收（两层，必须都过）

- L1 连通：用户发 `你好` → inbox 有 `received` → worker 认领 → outbox `to_send` → serve 发回 → 用户确认只收到 1 条。
- L2 真实任务：用户发可核对任务（例“今天天气怎么样”或“打开 <网页> 告诉我标题”）→ worker 真调工具 → 结果回微信 → 用户核对正确。
- 不要声称“微信聊天已同步到 Muse 主聊天”，微信只是通道，主聊天不会自动出现微信对话。

## 8. 出现不回复时的检查顺序

1. `status`：session 是否 connected？inbox 是否有 received？outbox 是否有 failed？
2. `health`：serve 是否在跑？session 是否 paused？
3. `logs/bridge.log`（已脱敏）：区分网络 / API / -14。
4. cron worker 最近是否 claim 了该消息？是否有 stuck processing？
5. -14 → 等 1 小时或重扫，不要删 session/cursor。
6. 重复发送 → 查 `client_id` 是否复用（§5 P3）。
7. 扫码只出串码 → 查二维码是否编的是 URL（§5 P1），是否扫了旧图（§5 P2）。

## 9. 快速命令（在 `~/workspace/wechat-bridge/` 下）

```bash
python3 -m bridge.cli login
python3 -m bridge.cli wait-login [--verify-code <手机上显示的验证码>]
python3 -m bridge.cli serve  # nohup / tmux 保持后台
python3 -m bridge.cli status
python3 -m bridge.cli health
# 登录成功后再装 worker（见 §3.4），自适应 10s/10m，每轮最多 3 条
python3 -m bridge.cli send --to <to_user_id> --text "测试" --context-token <ctx>
```
