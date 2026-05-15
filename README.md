# 更好的游戏列表 / BetterGameList

通过 RELAY-CN 自有资源，在保障 RELAY-CN 稳定运行的同时，面向大众提供反广告游戏列表。

本项目兼容原版《铁锈战争》`masterserver/1.4/interface` API 逻辑，并在内部审核环节引入 AI API 过滤能力。设计上可继续接入 IronCore 启发式扫描机制，用于对新出现的房间做更实时的补充防护。

## 原理

```text
- 每 30-45 秒拉取一次上游 List 数据
- 每次拉取后调用 AI 审核引擎进行判断
- 依据审核结果过滤违规条目
- AI 全部批次成功后，原子更新公开列表
- AI、上游或数据库任一关键步骤失败时，保留旧公开列表
```

AI 审核词库将公开并持续迭代优化。

我们立足玩家群体，期待与您携手共建。

RELAY-CN Team. 2026

- 由 `NebulaPause.Group` 群组提供设想

## 如何对接

反广告 API 与原版 MasterServer API 保持一致，可直接作为主列表源接入：

```text
https://list.rw.der.kim/gs1/masterserver/1.4/interface
```

推荐将反广告 API 作为主 API，在不可用时回退官方 API：

```text
主 API：
https://list.rw.der.kim/gs1/masterserver/1.4/interface

官方 API：
https://gs1.corrodinggames.com/masterserver/1.4/interface
https://gs4.corrodinggames.net/masterserver/1.4/interface
```

客户端请求参数不需要改，仍然使用原版参数：

```text
https://list.rw.der.kim/gs1/masterserver/1.4/interface?action=list&game_version=176&game_version_beta=false
```

推荐回退逻辑：

```text
- 优先请求反广告 API
- 如果请求超时、网络失败，或响应头不是 CORRODINGGAMES[1.0]，再请求官方 API
- 如果反广告 API 正常返回，不再请求官方 API
```

## 协议入口

主接口：

```text
/masterserver/1.4/interface.php
```

常用 action：

- `action=list`：返回公开房间列表。
- `action=add`：玩家创建房间，写入 `room_buffer`。
- `action=update`：玩家更新房间，更新 `room_buffer`，不会立即进公开表。
- `action=remove`：按 `id + private_token` 删除房间。
- `action=get`：客户端加入房间时获取真实连接 IP/端口。
- `action=self_info`：返回当前请求 IP 和端口开放状态。

客户端 `action=get` 返回格式必须保持：

```text
CORRODINGGAMES[1.0]
OK
<f.c("game_" + c)>

<room csv>
```

第三行是单次 SHA-256 后截取 14 位，第四行必须是空行。

## 上游同步

每 30-45 秒 会执行一次上游同步 (我们认为这是在防止对上游造成压力和满足下游的大概的实时)：

同步规则：

- 官方房间直接写入。
- 非官方房间交给 AI 审核。
- AI 全部批次成功后，原子更新公开表。
- AI 失败、上游失败或返回异常时，不发布本轮结果，保留旧数据。

## 上游连接字段策略

为了避免公开列表直接暴露非官方上游房间连接地址，当前策略是：

- 官方房间：保留上游 IP、端口和原 `address_version`。
- 非官方普通 IP 房间：
  - `action=list` 中 `public_ip` 返回 `0.0.0.0`
  - `address_version` 强制大于 0，让客户端走 `action=get`
  - `action=get` 返回缓存或实时获取到的真实 IP/端口
- URL 房间保持上游原样。

如果同一个上游 UUID 已经缓存过真实 IP/端口，下次同步会复用缓存，不重复请求上游 `action=get`。

## AI 过滤

AI 输入是一批房间摘要，每行包含：

```text
uuid    玩家名    地图名    当前人数    最大人数    版本号
```

AI 只需要返回允许公开的 UUID 列表，例如：

```json
["u_xxx", "u_yyy"]
```

当前 AI 客户端配置：

- 每批默认 60 条。
- 默认 2 个批次并发。
- 任意批次失败，本轮 AI 过滤失败。
- AI 失败时不更新公开表。

提示词和广告样例会缓存到 `runtime/ai-prompt-cache`，默认缓存 1 小时。缓存命中按固定文件名和文件修改时间判断。

## 输入清洗

玩家 `add/update` 输入会做协议清洗，避免破坏客户端 `split(",")` 解析：

- 英文逗号 `,` 替换为中文逗号 `，`
- `\r`、`\n`、`\t` 替换为空格
- 控制字符只保留 `\x01`，其他控制字符替换为空格

`private_token` 等鉴权字段不做协议文本清洗，避免影响认证。

调试接口不建议长期暴露在公网。
