# MLTW — Muse Link to WeChat

让你的 Muse 住进微信。MLTW 通过腾讯公开的 ClawBot 插件和 iLink 协议，把 Muse 和你自己的微信连起来：在微信里给你的 Muse 发文字任务，它真实执行后把结果回传到微信。

## 当前形态：MLTW-skill

从 v3.0 起，MLTW 只以 Muse skill 的形式发布，名字叫 **MLTW-skill**。旧的文档版（work.md）已停止更新，它的全部内容已经并入 skill 包内的完整参考指南。

## 安装（约 10 分钟）

1. 在本仓库的 Releases 里下载 `MLTW-skill.zip` 和 `Install-prompts.md`
2. 新开一个 Muse 对话，把 zip 作为附件上传，再把 `Install-prompts.md` 里的整段文字发给它
3. 你的 Muse 会先审查代码、再安装部署；最后的微信扫码登录由你本人完成

部署完成后，微信里发来的消息约 1–3 分钟内得到回复；桥的健康状态可用包内的自检脚本 `doctor.py` 一键检查。

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
