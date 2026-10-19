# Muse WeChat Bridge

把 Muse 接入微信，在微信里发文字任务，Muse 执行后把结果回传微信。
反馈/建议邮箱：skylinexe7@gmail.com

## 配置流程

### 1. 把 work.md 发给 Muse

先把仓库里的 `work.md` 发给你的 Muse，让它按里面的步骤搭建桥。不要自己手动搭，`work.md` 是给 Muse 看的完整搭建指南。

### 2. 微信里启用 ClawBot

微信 → 我 → 设置 → 插件 → 启用 **ClawBot**

### 3. 扫描 Muse 提供的二维码登录

Muse 会生成二维码发给你。

扫码方式：进 ClawBot → 点 **开始扫一扫** → 扫 Muse 提供的最新二维码 → 点确认。

> 二维码约 2 分钟过期，过期让 Muse 重新生成。扫出来是一串码说明扫了旧码，让 Muse 检查。

## 使用

在微信里直接给 ClawBot 发文字，例如：

```
你好
今天天气怎么样
```

## 原理

```
微信 → iLink → serve（收发） → inbox → Muse 定时任务 worker（执行） → outbox → serve → 微信
```

- 没有 webhook，`serve` 只负责收发，执行由 Muse 定时任务轮询 `inbox` 完成。
- 只支持 1 对 1 私聊，不支持群聊、图片 / 语音 / 文件。

## 常见问题

- 不回复：让 Muse 检查 `status` / `health`，看 `serve` 是否在跑、`inbox` 是否有积压。
- 重复发 3 次：让 Muse 检查 `client_id` 是否复用。
- `-14`：会话暂停 1 小时，等 1 小时或重新扫码。
