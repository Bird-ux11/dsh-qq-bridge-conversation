# 改造记录（相对上游）

上游的更新记录在 [CHANGELOG.md](CHANGELOG.md)，这里只记本仓库相对上游做了什么。

## 1.0.7-conversation.1

基于上游 `1.0.7`。

**把 QQ 消息的身份信息一起交给 agent**

- `agent.followup(...)` 的 `source` 里多了 `identityVault` 字段：`qqId`、`chatType`、`chatId`、`groupId`、`displayName`、`upstreamFirstMessageId`、`observedAt`、`userTextSpan`、`replyOnlyMarked`。
- 身份库（`dsh-identity-vault`）靠它把「谁在哪个群说了这句话」记进这个人物自己的档案；在此之前只能知道「有一条 QQ 消息进来了」。
- 校验不过就返回 `null`，`source` 退回上游的 `{ kind: 'qq' }`——改造失败时与上游行为一致。
- 改动位置：`lib/host.js` 的 `agent.followup(...)` 一处，`+44 / -1` 行，用 `/* dsh-identity-vault:qq-source:v2:begin */` … `:end */` 标出来。
- 新增两个账本文件：`lib/host.js.identity-vault.original`（上游原版）与 `lib/host.js.identity-vault.adapter.json`（版本、标记名、两个 sha256）。
