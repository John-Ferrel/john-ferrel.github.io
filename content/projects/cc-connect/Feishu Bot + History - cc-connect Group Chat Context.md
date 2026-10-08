---
title: Feishu Bot + History - cc-connect Group Chat Context
draft: false
created: 2026-09-03 00:00
modified: 2026-10-08 22:00
tags:
  - cc-connect
  - feishu
  - agent
  - context
---

本文记录 [PR #1698](https://github.com/chenhg5/cc-connect/pull/1698) 合入时的实现，以下示例以 `group_reply_all = false` 的普通群聊 text/post 消息为主。

之前 cc-connect 在这个配置下的行为是这样：

```
普通群消息
    ↓
忽略

@Bot 消息
    ↓
发送给 Agent
    ↓
回复
```

这个模型下，普通群聊讨论会被 mention filter 过滤，Agent 无法通过这些消息获得讨论上下文；已有的引用消息、thread bootstrap 等上下文路径另行处理。

例如：

```
Alice: 这个接口最近经常超时
Bob: 我看日志像是 upstream 的问题
Alice: 昨天晚上 11 点之后开始明显变多

Alice: @Bot 帮我总结一下现在的问题
```

Agent 实际收到的可能只有：

```
帮我总结一下现在的问题
```

前三条消息并不会进入上下文。

这次给 cc-connect 增加了一个 opt-in 配置：

```
group_chat_history_share = true
```

默认值仍然是：

```
false
```

## 行为

启用以后，上述 text/post 消息仍然需要满足已有触发条件，例如显式 `@Bot`。`group_chat_history_share` 本身不会让普通消息触发 Agent。

区别在于，普通群消息不再直接丢弃，而是先保存到当前 group scope 的 pending history。

```
Alice: message A
Bob:   message B
Carol: message C
          ↓
      pending history

Alice: @Bot question
          ↓
message A
message B
message C
question
          ↓
        Agent
```

因此：

```
“是否触发 Agent”
```

和：

```
“是否允许消息成为 Agent context”
```

被拆成了两个行为。

普通消息：

```
observe = yes
invoke  = no
```

`@Bot`：

```
observe = yes
invoke  = yes
```

## Pending history

pending history 保存当前进程观察到的、尚未被已接纳 Agent turn 消费的消息。连续两次正常请求都被接纳时，可以用下面的区间理解：

```
上一次有效 @
        ↓
本次有效 @
```

之间积累的消息。

例如：

```
A
B
C
@Bot Q1
D
E
@Bot Q2
```

Q1 获得：

```
A
B
C
Q1
```

Q2 获得：

```
D
E
Q2
```

Q1 消费过的：

```
A
B
C
```

不会再次注入 Q2。

这条缓存路径使用已有 message event，不额外调用 Feishu history API，也不会补齐启动前或断连期间未收到的讨论。缓存只保存在内存中；已有的引用消息和 thread bootstrap 路径仍可能调用飞书 API。

当前限制为每个 scope：

```
50 messages
```

超过以后丢弃最旧记录，进程重启后缓存清空。合入版本没有为 pending history 设置 TTL，50 条上限按 scope 计算；正文长度和 scope 总数也不受这个条数上限约束。

## 消息注入

历史消息按接收顺序注入，并保留 sender。

例如：

```
Alice: API 返回 502
Bob: staging 也出现了
Alice: prod 大概一分钟一次
```

不能压成：

```
API 返回 502
staging 也出现了
prod 大概一分钟一次
```

否则 Agent 无法判断是谁提供了哪部分信息。

最终通过 `ExtraContent` 把 pending history 附加到本次 Agent turn。下面仅展示结构，实际格式使用 sender 名称、换行和上下文分隔标记：

概念上类似：

```
ExtraContent:
  [Alice] API 返回 502
  [Bob] staging 也出现了
  [Alice] prod 大概一分钟一次

User:
  帮我分析一下
```

当前只处理能够稳定转换为上下文的：

```
text
post
```

`post` 会提取为纯文本，提取结果为空的消息不进入缓存。图片、文件和音频等消息沿用已有处理路径。

## `/status` 不消费 history

假设当前状态：

```
A
B
C
```

然后用户发送：

```
@Bot /status
```

`/status` 由 cc-connect 直接处理，不进入正常 Agent turn。adapter 可以为它附带 history 快照和回调，但 core 的命令分支不会调用 `OnAccepted`，因此不会消费缓存。

如果它把 pending history 一起消费掉，那么后面：

```
@Bot 帮我看一下刚才讨论的问题
```

就无法获得：

```
A
B
C
```

因此规则是：

```
/status
    ↓
读取 Bot 状态
    ↓
pending history 保留
```

即：

```
A
B
C
@Bot /status
@Bot Q
```

Q 仍然获得：

```
A
B
C
Q
```

## `/new` 清空 history

`/new` 表示开启一个新的 session。对于通过触发和权限检查的 text 命令，adapter 会清空当前 history scope，再继续交给 core 处理。

```
A
B
C
@Bot /new
D
E
@Bot Q
```

Q 只获得：

```
D
E
Q
```

应当排除的旧上下文如下：

```
A
B
C
D
E
Q
```

否则 Agent session 已经重置，但群聊上下文仍然跨 session 泄漏。

## Recall

如果群消息已经进入 pending history，随后被撤回，需要同时从缓存中删除。

例如：

```
Alice: password is ...
Alice: 撤回
Alice: @Bot 总结一下
```

收到 recall event 后，adapter 按 message ID 从 pending history 删除对应消息，格式化快照时也会跳过已标记撤回的记录。

这个保证适用于待注入的缓存。已经格式化并交给 core、进入队列或发送给 Agent 的 `ExtraContent`，无法仅靠删除 pending history 收回；撤回也不能保证清除 Agent 已处理的内容。

## Thread isolation

cc-connect 已经支持：

```
thread_isolation = true
```

启用后，群主频道和各个 thread 使用不同 scope。

例如：

```
Main:
  A
  B

Thread #1:
  C
  D
```

如果在 Thread #1 中：

```
@Bot Q
```

Agent 获得：

```
C
D
Q
```

不会混入：

```
A
B
```

如果：

```
thread_isolation = false
```

则按照现有 session 语义，main channel 与 thread 被视为同一个 scope，消息按到达顺序进入同一份 pending history。

history scope 按 chat ID 和 thread root ID 隔离，具体键为 `chat:<chat_id>` 或 `thread:<chat_id>:<root_id>`。它与 Agent session key 分别计算：开启 thread isolation 后，主频道的一次 mention 可以创建 root-scoped session，后续主频道普通消息仍保存在 chat-level history 中，不会跟随之前分出的 session。

## 权限边界

权限检查分别回答下面两个问题：

```
Bot 能不能观察一条群消息
```

和：

```
这个发送者能不能触发 Agent
```

两个问题分别对应 observation 和 invocation 权限。

实现中继续区分：

```
allow_chat
```

和：

```
allow_from
```

`allow_chat` 决定这个群是否属于 Bot 可以处理的范围。

`allow_from` 决定某个用户是否可以真正触发 Agent。允许访问的群中，未获触发权限的用户所发普通消息仍可能进入 history，随后由有权限的用户触发共享。

飞书侧还需要申请并发布 `im:message.group_msg` 权限，才能向 Bot 推送未提及它的群消息。仅打开本地配置无法获得飞书未推送的消息。

合入版本也保留了 fail-closed 检查：bot ID 无法正常解析、mention filter 处于 degraded 状态时，普通群消息不会被捕获为 history。

否则为了获取群聊上下文而放宽 message observation，很容易顺便放宽 Agent invocation 权限。

因此：

```
普通消息
    ↓
允许进入 history
```

不代表：

```
该发送者可以 @Bot 并执行任意 Agent 请求
```

## `group_reply_all` 不受影响

以下配置控制普通消息的触发行为：

```
group_reply_all = false
```

保持这个设置时，普通 text/post 消息不会因为打开 history share 而触发回复。

即使：

```
group_chat_history_share = true
```

Bot 对这些普通消息仍然只缓存上下文。

例如：

```
Alice: A
Bob: B
Carol: C
```

结果仍然是：

```
Bot: <nothing>
```

直到：

```
Alice: @Bot Q
```

才触发 Agent。

若已有配置是 `group_reply_all = true`，普通消息仍会立即进入原有处理路径。开启 history share 不会将它们改为等待下一次 mention。

所以两个配置控制的是不同维度：

```
group_reply_all
    → 什么消息触发回复

group_chat_history_share
    → 未触发回复的消息能否成为后续 context
```

## 状态模型

下面保留概念流程，适用于前述 mention 模式。实际实现会先检查 chat 权限和 mention filter；只有未触发 Bot 的合格 text/post 消息才进入 pending history。recall 由独立事件处理：

```
receive message
      ↓
resolve scope
      ↓
message recalled?
 ├─ yes → remove from pending history
 └─ no
      ↓
normal group text/post?
 ├─ yes → append pending history
 └─ no
      ↓
explicit Bot trigger?
 ├─ no  → stop
 └─ yes
      ↓
command?
 ├─ /status → keep pending history
 ├─ /new    → clear pending history
 └─ normal turn
      ↓
inject pending history
      ↓
Agent
      ↓
accepted turn
      ↓
consume pending history
```

这里的 accepted 指 core 接纳消息处理，或将消息加入忙碌 session 的队列。此时触发 `OnAccepted` 消费快照，Agent 的异步处理和最终回复可能尚未开始。

接纳前未触发回调时，pending history 会保留；接纳后发生模型错误或回复失败，不会自动恢复缓存。消费以快照中的最大记录 ID 为边界，快照之后新到达的普通消息会留给后续请求。

## 测试

合入版本的 `platform/feishu/group_history_test.go` 包含 10 个测试。下面按行为归纳，sender、顺序和 consume once 由同一个测试覆盖；recall 单独注明核实范围：

```
default-off
```

没有打开 `group_chat_history_share` 时，保持原有行为。

```
ordered history
```

多条消息按原顺序注入。

```
sender attribution
```

不同发送者不能在格式化时丢失身份。

```
consume once
```

一批历史只能被一次正常 Agent turn 消费。

```
/status
```

不消费 pending history。

```
/new
```

清空当前 scope。

```
thread isolation
```

main / thread 的 scope 与现有 `thread_isolation` 规则一致。

```
recall
```

源码已核实撤回消息会从 pending history 删除；上述测试文件没有专门针对 pending history recall 的测试，不能把这一项计为新增测试覆盖。

```
permissions
```

观察群聊消息不能绕过原有 invocation 权限。

```
group_reply_all
```

专门的测试验证 `group_reply_all = true` 时普通消息仍立即派发；其他 history 测试验证 mention 模式下普通消息只缓存、不派发。

```
text / post formatting
```

两种支持的 Feishu message type 在进入 history 后保持可读格式。

此外，`TestFeishuGroupHistory_DegradedMentionFilterDoesNotCapture` 验证 bot ID 解析降级时不捕获 history。PR 的提交记录还列出了全量测试、race detector、构建和实群验证；这些结果对应提交时的版本。

## 配置

下面保留配置意图示意。`[platforms.feishu]` 不是合入版本的实际 TOML 层级，不能直接照抄：

```
[platforms.feishu]
group_chat_history_share = true
group_reply_all = false
thread_isolation = true
```

实际使用时，在已有 `[[projects.platforms]]` 且 `type = "feishu"` 的配置项下，将这三个选项写入对应的 `[projects.platforms.options]`，与 `app_id`、`app_secret` 同级。层级可对照[合入版本的配置示例](https://github.com/chenhg5/cc-connect/blob/4000b2338aa6e850c99df54f8b0ed6ed7460b401/config.example.toml)。

启用前还需要完成飞书群消息权限的申请和发布。本文的显式 mention 示例沿用默认 mention 设置；`@all`、已激活 thread 的附件等触发规则仍由原有配置和处理路径决定。

对应的群聊行为：

```
普通消息
  → 缓存
  → 不回复

@Bot
  → 注入当前 scope 的 pending history
  → 调用 Agent
  → 回复

/status
  → 不消费 pending history

/new
  → 清空当前 scope

recall
  → 从 pending history 删除

restart
  → pending history 清空
```

PR #1698 在 rebase 和 CI 通过后，于 2026-09-03 合入 cc-connect main。

## 参考

- [合入 commit：4000b23](https://github.com/chenhg5/cc-connect/commit/4000b2338aa6e850c99df54f8b0ed6ed7460b401)
- [Feishu adapter 实现](https://github.com/chenhg5/cc-connect/blob/4000b2338aa6e850c99df54f8b0ed6ed7460b401/platform/feishu/feishu.go)
- [Group history 测试](https://github.com/chenhg5/cc-connect/blob/4000b2338aa6e850c99df54f8b0ed6ed7460b401/platform/feishu/group_history_test.go)
- [PR #1698：feat(feishu): share group chat context on mention](https://github.com/chenhg5/cc-connect/pull/1698)
