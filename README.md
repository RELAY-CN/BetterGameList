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

## 文件说明

- `AI_PromptFilter.txt`：AI 屏蔽提示词，用于约束 AI 审核时的判定规则与拦截范围。
- `ExamplesOfAdNames.txt`：附加的 AI 参考词，提供典型广告/引流名称样例，辅助 AI 识别相似文本。
- `IP_Whitelist.ini`：IP 白名单配置文件，当前版本暂未生效。

## 如何对接

反广告 API 与原版 MasterServer API 保持一致，可直接作为主列表源接入：

```text
https://list.rw.der.kim/masterserver/1.4/interface
```

推荐将反广告 API 作为主 API，在不可用时回退官方 API：

```text
主 API：
https://list.rw.der.kim/masterserver/1.4/interface

官方 API：
https://gs1.corrodinggames.com/masterserver/1.4/interface
https://gs4.corrodinggames.net/masterserver/1.4/interface
```

客户端请求参数不需要改，仍然使用原版参数：

```text
https://list.rw.der.kim/masterserver/1.4/interface?action=list&game_version=176&game_version_beta=false
```

推荐回退逻辑：

```text
- 优先请求反广告 API
- 如果请求超时、网络失败，或响应头不是 CORRODINGGAMES[1.0]，再请求官方 API
- 如果反广告 API 正常返回，不再请求官方 API
```

如果客户端不便实现回退逻辑，也可以额外接入一个 `AD Server` 列表接口，用于获取广告房间列表：
```text
https://list.rw.der.kim/masterserver/1.4/adinterface?action=list
```

该接口当前只支持 `action=list`。

可在客户端 Hook 的列表数据处理阶段额外获取此接口数据，再与官方列表结果做异或处理，从而得到无广告列表数据。

## 服务状态

可通过状态页查看最近一小时的上游拉取、官方房间、广告过滤与正常放行趋势：

```text
https://list.rw.der.kim/masterserver/1.4/server
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

每 30-45 秒会执行一次上游同步。该间隔用于降低上游压力，同时满足下游列表的大致实时性：

同步规则：

- 官方房间直接写入。
- 非官方房间交给 AI 审核。
- AI 全部批次成功后，原子更新公开表。
- AI 失败、上游失败或返回异常时，不发布本轮结果，保留旧数据。

## 上游连接字段策略

为了避免公开列表直接暴露非官方上游房间连接地址，当前策略是：

- 官方房间：保留上游 IP、端口和原 `address_version`。
- 非官方普通 IP 房间：
  - `action=list` 中 `public_ip` 返回占位地址 `1.1.1.1`
  - `address_version` 强制大于 0，让客户端走 `action=get`
  - `action=get` 返回缓存或实时获取到的真实 IP/端口
- URL 房间保持上游原样。

如果同一个上游 UUID 已经缓存过真实 IP/端口，下次同步会复用缓存，不重复请求上游 `action=get`。

## AI 过滤

AI 输入是一批房间摘要，每行包含：

```text
uuid    game_name    created_by    game_map    game_status    current_players    max_players    version_name    password_required
```

AI 只需要返回允许公开的 UUID 列表，例如：

```json
["u_xxx", "u_yyy"]
```

当前 AI 客户端配置：

- 每批默认 60 条。
- 默认 4 个批次并发。
- 任意批次失败，本轮 AI 过滤失败。
- AI 失败时不更新公开表。

提示词和广告样例会缓存到 `runtime/ai-prompt-cache`，默认缓存 1 小时。缓存命中按固定文件名和文件修改时间判断。

## 输入清洗

玩家 `add/update` 输入会做协议清洗，避免破坏客户端 `split(",")` 解析：

- 英文逗号 `,` 替换为中文逗号 `，`
- `\r`、`\n`、`\t` 替换为空格
- `\x01` / `\u0001` 等不可见控制字符会被过滤
- 零宽字符、Bidi 控制符、BOM 等不可见 Unicode 字符会被过滤
- 普通空格会保留，因为它是可见字符

`private_token` 等鉴权字段不做协议文本清洗，避免影响认证。

公开表中的历史数据也会在上游同步时顺带清理不可见字符，避免旧房间继续携带排序前缀。

# 鸣谢

- RELAY-CN 团队  
- NebulaPause.Group 群友  
- 幻想天域  
- 隔离区  

使用API的你 我 他 与屏幕前的你  
