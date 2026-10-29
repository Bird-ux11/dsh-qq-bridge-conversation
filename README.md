# dsh-qq-bridge · conversation 改造版

**这是 [`Yuuyuko-uu/dsh-qq-bridge`](https://github.com/Yuuyuko-uu/dsh-qq-bridge) 的改造版，不是原作品。** 全部代码、文档、工具的著作权归原作者。本仓库只做了一处适配改造，详见下文。

版本号 `1.0.7-conversation.1`，表示「基于上游 1.0.7，改造第 1 版」。

## 为什么有这个改造

配套的 [`dsh-qq-conversation`](https://github.com/Bird-ux11/dsh-qq-conversation) 和 [`dsh-identity-vault`](https://github.com/Bird-ux11/dsh-identity-vault) 需要知道：**这条消息是谁说的、在哪个群、什么时候看到的、正文从第几个字开始是本人原话**。

上游把这轮对话交给 agent 时只有 `source: { kind: 'qq' }` —— 身份库拿不到发起人，也就没法把「这条消息属于谁」记进人物档案。所以这里把身份信息一起递过去。

## 改了什么

只有一个位置，`lib/host.js` 里那次 `agent.followup(...)`，包在 `/* dsh-identity-vault:qq-source:v2:begin */` 和 `:end */` 之间：

```js
await agent.followup({
  id: 新id(), role: 'user', content,
  source: { kind: 'qq', ...(身份库来源 ? { identityVault: 身份库来源 } : {}) }
})
```

`身份库来源` 由 `buildQqIdentitySource(row, sender, firstMessageId, observedAt, userTextSpan, replyOnlyMarked)` 造出来，字段：

| 字段 | 说明 |
| --- | --- |
| `version` | 固定 `1`，给身份库判结构版本用 |
| `transport` | 固定 `'onebot11'` |
| `qqId` / `chatId` | 发送者 QQ 号 / 会话（群或私聊）号，都过一遍「5~20 位、首位非 0」的十进制校验 |
| `chatType` | `'group'` 或 `'direct'` |
| `groupId` | 只在群聊里出现，等于 `chatId` |
| `displayName` | 发送者昵称，去掉控制字符，截到 200 字 |
| `upstreamFirstMessageId` | **只记合并后的第一条**，不是完整 ID 列表——桥会把同一发送者的连续消息并成一批，这里刻意不假装是全量 |
| `observedAt` | 桥看到这条消息的时间戳 |
| `userTextSpan` | `{ start, length }`：正文里哪一段是本人原话（前面可能有 `昵称：` 署名） |
| `replyOnlyMarked` | 这一轮是否处于「只发标记内」模式 |

任何一项校验不过（`row` / `sender` 缺失、类型不是群或私聊、私聊里发送者和会话不是一个号、时间戳不合法）就返回 `null`，`source` 退回上游的 `{ kind: 'qq' }` —— **改造失败时行为与上游完全一致**，不会把消息弄丢。

相对上游：`+44 / -1` 行，全在 `lib/host.js`。

## 怎么跟着上游升级

`lib/` 下两个跟着 host.js 走的文件是这个改造的账本：

- `lib/host.js.identity-vault.original` —— 上游 1.0.7 的原始 `host.js`，改造前的样子
- `lib/host.js.identity-vault.adapter.json` —— 记账：`adapterVersion: 2`、`bridgeVersion: "1.0.7"`、标记名 `dsh-identity-vault:qq-source:v2`，以及原版和改后版的 sha256

升级步骤：

1. 拿新版本上游的 `host.js`
2. 比对它的 sha256 和 `originalHash`。**不一样说明上游改过这个文件，不能直接覆盖**
3. 一样的话，把 `begin` / `end` 之间那段连同下面那行 `followup` 整个搬过去，再把 `patchedHash` 更新掉
4. 不一样的话，在新版里找到 `agent.followup(`，重新套一遍同样的逻辑，然后更新 `bridgeVersion` 与两个 hash

`host.js` 自己**不读**这两个文件——它们是给人和脚本看的，删掉不影响运行。

## 装

```bash
dsh plugin --profile <你的 profile> add file:<本仓库路径或 tgz>
```

需要配套：

- [`dsh-qq-conversation`](https://github.com/Bird-ux11/dsh-qq-conversation)（它 `inject` 了 `qqConversationBridge`）
- [`dsh-identity-vault`](https://github.com/Bird-ux11/dsh-identity-vault) `0.9.4-conversation.1` 或更高

**不要把这个版本和上游版本混装**：上游没有 `qqConversationBridge` 接口，`dsh-qq-conversation` 装上会因为依赖缺失起不来。反过来只装本仓库不装 conversation 插件也能用，行为和上游一样。

## 许可

MIT，著作权归原作者 Yuuyuko-uu，见 [LICENSE](LICENSE)。本仓库的改造部分同样以 MIT 提供。

上游的完整说明见 [README.upstream.md](README.upstream.md)，上游更新记录见 [CHANGELOG.md](CHANGELOG.md)（本仓库的改造记录单独放在 [CHANGELOG.conversation.md](CHANGELOG.conversation.md)）。
