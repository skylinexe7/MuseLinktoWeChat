# MLTW — Muse Link to WeChat

让你的 Muse 住进微信。MLTW 通过腾讯公开的 ClawBot 插件和 iLink 协议，把 Muse 和你自己的微信连起来：在微信里给你的 Muse 发文字任务，它真实执行后把结果回传到微信。

## 当前形态：MLTW-skill

从 v3.0 起，MLTW 只以 Muse skill 的形式发布，名字叫 **MLTW-skill**。旧的文档版（work.md）已停止更新，它的全部内容已经并入 skill 包内的完整参考指南。

## 安装（约 10 分钟）

1. 在本仓库的 Releases 里下载 `MLTW-skill.zip` 和 `Install-prompts.md`
2. 新开一个 Muse 对话，把 zip 作为附件上传，再把 `Install-prompts.md` 里的整段文字发给它
3. 你的 Muse 会先审查代码、再安装部署；最后的微信扫码登录由你本人完成

部署完成后，微信里发来的消息约 1–3 分钟内得到回复；桥的健康状态可用包内的自检脚本 `doctor.py` 一键检查。

## 从旧版本升级

先说清一个原则：升级换的是「部署和运维的知识层」，桥本体不用重装——桥的代码、登录状态（`state/session.json`）、收件箱都在 `~/workspace/wechat-bridge/` 里，升级全程不用重新扫码。开始前先下载最新版 `MLTW-skill.zip` 和 `Install-prompts.md`。

### 支线 A：从 md 文档版（v1.x / v2.0）升级

当年照着 work.md 部署的，按这条走：

1. 新开一个 Muse 对话，上传 `MLTW-skill.zip`，把 `Install-prompts.md` 整段发给它，并补一句：「我是从 md 文档版升级，不要新装、不要重新扫码，按升级处理」。
2. 让它审查后把 `mltw_skill` 装进 `~/workspace/custom-skills/`，然后对照新版逐项核对你现有的桥，缺什么补什么：
   - serve 还在用 nohup 手工跑的 → 按参考指南 §3.5 改成 systemd，并生成 `state/proxy.env`（不补会「假活」：进程在、消息收不到）
   - 回复还靠定时任务轮询的 → 换成哨兵 hook（30 秒扫一次）+ 固定 30 分钟巡检，原来那个轮询任务要停用或换成新版巡检，两套同时跑会把一条消息回两遍
   - 一切写 outbox 的指令里把回复字段写死为 `to_user_id` 和 `text` 等六个字段（字段名写错会发空消息，见 P15）
   - 有定时/主动发送任务的 → 发送时一律现取最新一条收件箱消息的 `context_token`，不要复用存下来的旧 token（见 P16）
3. 跑包里的 `python3 scripts/doctor.py ~/workspace/wechat-bridge`，零 FAIL 才算完成；再从微信发一条测试消息，确认只收到一条回复、延迟约 1–3 分钟。
4. 旧的 work.md 可以留着做纪念，但不要再让 Muse 照它部署或排障——它已停更，缺了 P16 等新内容。

### 支线 B：从 skill 版（MLTW-skill-beta）升级

装过 MLTW-skill-beta（技能名 `mltw_skill_beta`）的，按这条走：

1. 新开对话，把新版 `MLTW-skill.zip` 发给你的 Muse，说明是「从 beta 版升级」。
2. 让它把 `~/workspace/custom-skills/` 里的旧 `mltw_skill_beta` 文件夹删掉，再装入新的 `mltw_skill`——新版改了名，两个文件夹不要同时留着。桥目录和登录状态不动，不用重新扫码。
3. 对照 `CHANGELOG.md` 里 0.2.0 和 1.0.0 两节，把运行时差异补上，主要是 P16 的发送规则：有定时/主动发送任务的，改成发送时现取最新 token。systemd 模板的 `__HOME__` 占位符只影响新装，已装好的不用重装。
4. 跑 `doctor.py` 零 FAIL，再发一条微信测试消息验证。

以后 1.0.0 之后的小版本升级更简单：整夹覆盖 `mltw_skill`，再跑一遍 `doctor.py` 就行。

## 能力与限制

- 仅支持绑定账号本人的 1 对 1 私聊、纯文字回复
- 不能进群、不能主动推送、不能同步历史消息
- 这不是官方消息通道，依赖 ClawBot/iLink，接口或插件政策变化可能导致失效
- 登录 token 只保存在你自己的机器上，不要外传

## 反馈

- 邮箱：skylinexe7@gmail.com
- QQ 交流群：554020882

## License

MIT
