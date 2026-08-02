# FlyA Test API 客户端适配协议 v1

[English](README.md) | [简体中文](README.zh-CN.md) | [日本語](README.ja.md)

| 项目 | 值 |
| --- | --- |
| 状态 | **公开测试** |
| Base URL | `https://api.nashout.com` |
| API 前缀 | `/beta/v1` |
| 协议标识 | `flya-test-api-v1` |

## 1. 认证与平台边界

所有请求必须携带：

```http
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

Key 以 `flyat_` 开头。客户端应保持 TLS 证书校验开启，不得把 Key 放进网页前端、URL、日志或代码仓库。该接口故意不开放浏览器 CORS。

FlyAPI 与游戏平台无关，不接收、不识别也不限制雀魂、天凤、一番街、天月或其他平台。客户端 Bridge 只需把平台原生协议转换成本文定义的 `flya-mahjong-events-v1` observed 标准事件流；请求中不发送平台名。

服务端负责重建场况并计算全部合法动作。客户端不得发送 `legal_actions`、`legal_digest`、牌山、隐藏手牌、tensor、mask、obs 或其他模型私有输入。

## 2. 接口一览

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/beta/v1/models` | 查询当前 Key 可用的模型和单次成本 |
| `GET` | `/beta/v1/quota` | 激活新 Key，并查询状态与额度 |
| `POST` | `/beta/v1/decision` | 提交完整可见事件历史并请求决策 |

Key 缺失、无效、已吊销或已销毁时返回：

```json
{
  "error": "test_api_key_invalid",
  "message": "test API key is invalid"
}
```

## 3. 查询模型

```bash
curl -H "Authorization: Bearer $FLYAT_KEY" \
  https://api.nashout.com/beta/v1/models
```

响应示例：

```json
{
  "models": [
    {
      "model_id": "flya-manplus-1",
      "display_name": "manplus",
      "rule_line": "riichi4p",
      "provider": "flya",
      "multiplier": "1.00000000000000000000",
      "cost_milliunits": 1000,
      "is_default": true,
      "available": true,
      "unavailable_reason": null
    }
  ]
}
```

| 字段 | 含义 |
| --- | --- |
| `model_id` | `/decision` 接受的稳定模型 ID |
| `display_name` | 面向用户的显示名称 |
| `rule_line` | `riichi4p` 或 `riichi3p` |
| `provider` | 模型提供方标识 |
| `multiplier` | 以十进制字符串表示的普通额度成本 |
| `cost_milliunits` | 以千分之一额度为单位的单次成本 |
| `is_default` | 是否为该规则线的服务器默认模型 |
| `available` | 当前是否存在可路由运行实例 |
| `unavailable_reason` | 不可用时的稳定原因码，否则为 `null` |

客户端启动或准备选模时应读取本接口，不要永久硬编码可用状态、成本、显示名称或默认模型。返回结果会按当前 Key 过滤，并携带 `Cache-Control: private, no-store` 和 `Vary: Authorization`；不得跨 Key 缓存。

## 4. 额度与激活

```bash
curl -H "Authorization: Bearer $FLYAT_KEY" \
  https://api.nashout.com/beta/v1/quota
```

`GET /quota` 是唯一激活入口。第一次成功查询开始计算有效期，并返回 `activated_now: true`；以后固定为 `false`。响应携带 `Cache-Control: no-store`。

按量额度示例：

```json
{
  "key": "flyat_abcdef...1234",
  "status": "active",
  "key_kind": "paygo",
  "validity_months": 1,
  "activated_now": true,
  "activated_at": "2026-08-02T00:00:00Z",
  "expires_at": "2026-09-02T00:00:00Z",
  "expires_in_seconds": 2678400,
  "quota_total": "1000.000",
  "quota_used": "0.000",
  "quota_remaining": "1000.000"
}
```

订阅示例：

```json
{
  "key": "flyat_abcdef...1234",
  "status": "active",
  "key_kind": "subscription",
  "subscription_type": "launch",
  "validity_months": 1,
  "activated_now": false,
  "activated_at": "2026-08-01T00:00:00Z",
  "expires_at": "2026-09-01T00:00:00Z",
  "destroy_at": null,
  "destroy_in_seconds": null,
  "five_hour": {
    "limit": "1000.000",
    "used": "12.000",
    "remaining": "988.000",
    "resets_at": "2026-08-02T05:00:00Z"
  },
  "weekly": {
    "limit": "5000.000",
    "used": "12.000",
    "remaining": "4988.000",
    "resets_at": "2026-08-09T00:00:00Z"
  }
}
```

额度字段是最多三位小数的十进制字符串；使用受限续局透支时，`remaining` 可以为负数。客户端应使用 decimal 或整数 milliunits，不要用二进制浮点数累计余额。

按量额度用尽或 Key 到期后会进入 `grace`。三天宽限期内仍可查询 `/quota`，并取得 `destroy_at` 和 `destroy_in_seconds`；销毁后返回 `401 test_api_key_invalid`。服务端永久保留 Key 摘要墓碑，因此旧 Key 永不重新发行。

## 5. 决策请求

```http
POST /beta/v1/decision HTTP/1.1
Host: api.nashout.com
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

```json
{
  "request_id": "client-table-42-turn-17",
  "session_id": "client-table-42",
  "model_id": "flya-manplus-1",
  "state": {
    "schema": "flya-mahjong-events-v1",
    "rule_line": "riichi4p",
    "viewer_seat": 0,
    "source": { "kind": "observed" },
    "from_seq": 0,
    "to_seq": 3,
    "events": [],
    "state_digest": "fnv1a64:0000000000000000"
  },
  "deadline_ms": 8000
}
```

### 顶层字段

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `request_id` | 是 | 当前 Key 内唯一的幂等 ID；1–200 字符，只允许 `A-Z a-z 0-9 . _ : -` |
| `session_id` | 否 | 同一局稳定 ID；8–128 字符，字符集相同。仅用于连续性与受限续局透支 |
| `model_id` | 否 | 省略时使用该规则线的服务器默认模型；显式指定时必须来自 `/models` 且可用 |
| `state` | 是 | 完整的 `flya-mahjong-events-v1` 本席可见场况信封 |
| `deadline_ms` | 否 | 正整数；默认及最大实际等待均为 30000 ms |

未知顶层字段会被拒绝。响应中的 `model_selection` 为 `server_default` 或 `explicit`。

### `state` 信封

| 字段 | 规则 |
| --- | --- |
| `schema` | `flya-mahjong-events-v1` |
| `rule_line` | `riichi4p` 或 `riichi3p` |
| `viewer_seat` | 四麻为 `0..3`，三麻为 `0..2` |
| `source.kind` | 必须为 `observed`；`authoritative` 会被拒绝 |
| `from_seq` | 必须为 `0`；每次请求都包含从开局开始的完整历史 |
| `to_seq` | 必须等于 `events.length` |
| `events` | 按时间顺序排列的本席可见 canonical MJAI 事件 |
| `state_digest` | FlyTable 对 `events` 计算的稳定摘要 |

服务端校验完整历史与摘要，从最近的 `start_kyoku` 重放当前局，确认当前确实需要决策，并生成唯一合法动作表。若 observed 流省略了对手庄家的隐藏第 14 张起手牌，服务端会在内部补未知牌；客户端不要自行推导。

隐私要求：

- 其他玩家的起手牌使用 `"?"`。
- 其他玩家摸到的牌使用 `"?"`。
- `scores`、`deltas`、`tehais` 的席位宽度必须与规则线一致。
- 不得发送牌山、其他玩家真实手牌、authoritative seed、obs、tensor、mask 或模型私有特征。

不要根据任意 JSON 文本计算摘要。摘要是 FNV-1a 64，输入为 FlyTable 类型化序列化后的紧凑 UTF-8 JSON。Rust 客户端应直接调用：

```rust
use flytable_protocol::compute_state_digest;

let state_digest = compute_state_digest(&events);
```

其他语言必须复现相同字段顺序和 canonical 序列化，并用已知正确的 FlyTable 夹具验证实现。

## 6. 公开动作形式

合法动作只出现在成功响应中；每个候选的 `action_id` 位于动作对象外层。

| `type` | 动作对象 | 含义 |
| --- | --- | --- |
| `dahai` | `{"type":"dahai","pai":"5pr","tsumogiri":false}` | 普通切牌 |
| `dealer_opening_dahai` | `{"type":"dealer_opening_dahai","pai":"5pr"}` | 庄家从 14 张起手牌完成的第一次切牌 |
| `riichi_dahai` | `{"type":"riichi_dahai","pai":"5pr","tsumogiri":false}` | 立直切牌；与普通切牌是不同动作 |
| `dealer_opening_riichi_dahai` | `{"type":"dealer_opening_riichi_dahai","pai":"5pr"}` | 庄家起手立直切牌 |
| `chi` | `{"type":"chi","pai":"3m","consumed":["1m","2m"]}` | 吃；仅四麻 |
| `pon` | `{"type":"pon","pai":"E","consumed":["E","E"]}` | 碰 |
| `daiminkan` | `{"type":"daiminkan","pai":"E","consumed":["E","E","E"]}` | 大明杠 |
| `ankan` | `{"type":"ankan","pai":"5p","consumed":["5p","5p","5p","5p"]}` | 暗杠 |
| `kakan` | `{"type":"kakan","pai":"E","consumed":["E","E","E"]}` | 加杠 |
| `kita` | `{"type":"kita"}` | 北拔；仅三麻 |
| `tsumo` | `{"type":"tsumo","pai":"2s"}` | 自摸和了；庄家起手场况可省略 `pai` |
| `ron` | `{"type":"ron","pai":"2s","target":1}` | 荣和；`target` 是放铳席，庄家起手场况可省略 `pai` |
| `kyushukyuhai` | `{"type":"kyushukyuhai"}` | 九种九牌流局 |
| `pass_all` | `{"type":"pass_all"}` | 放弃当前响应窗口 |

`chi.pai` 是上家打出的被吃牌，`chi.consumed` 是本席手里拿出的两张牌。`dealer_opening_*` 描述动作时序，不代表游戏平台；该第一次切牌之前没有普通 `tsumo` 事件。

牌字符串沿用 FlyTable：数牌为 `1m`–`9m`、`1p`–`9p`、`1s`–`9s`；赤五为 `5mr`、`5pr`、`5sr`；字牌为 `E S W N P F C`。`"?"` 只能用于事件流中的隐藏牌，不能出现在动作中。

## 7. 决策响应

成功示例：

```json
{
  "protocol": "flya-test-api-v1",
  "request_id": "client-table-42-turn-17",
  "model_id": "flya-manplus-1",
  "model_selection": "server_default",
  "attempt": {
    "model_id": "flya-manplus-1",
    "model_selection": "server_default",
    "status": "success",
    "selected_action_id": 2,
    "selected_index": 2,
    "action": {
      "type": "dahai",
      "pai": "W",
      "tsumogiri": false
    },
    "actions": [
      {
        "action_id": 2,
        "action": { "type": "dahai", "pai": "W", "tsumogiri": false },
        "probability": 0.6
      },
      {
        "action_id": 9,
        "action": { "type": "dahai", "pai": "8p", "tsumogiri": false },
        "probability": 0.4
      }
    ],
    "latency_ms": 2362
  },
  "quota": {
    "key": "flyat_abcdef...1234",
    "status": "active",
    "quota_total": "1000.000",
    "quota_used": "1.000",
    "quota_remaining": "999.000",
    "consumed": "1.000",
    "replay": false
  }
}
```

客户端直接执行 `attempt.action`，并确认它与 `attempt.actions` 中 `action_id == attempt.selected_action_id` 的条目完全一致。`selected_index` 只是兼容字段，不能替代 `selected_action_id`。

`attempt.actions` 是服务端返回的最终候选动作列表。每项包含稳定 `action_id`、动作和 `probability`，全部概率之和为 1。客户端不得假设候选数量或排序方式。

模型进入运行时后，即使失败、超时或 abstain，HTTP 仍返回 `200`。客户端必须检查 `attempt.status`：

- `success`
- `failure`
- `timeout`
- `abstain`

## 8. 计费、续局与幂等

1. 结构、模型和路由校验通过后，服务端在进入运行时前原子扣除所选模型成本。
2. 已进入运行时的 `success`、`failure`、`timeout` 和 `abstain` 都消耗本次额度。
3. 认证失败、格式错误、模型不支持、路由不可用等在运行时前拒绝的请求不扣额度。
4. `request_id` 是同一 API Key 内的幂等键。
5. 相同 ID 与完全相同请求体返回首次终态结果，并带 `quota.replay=true`、`quota.consumed="0.000"`。
6. 相同 ID 与不同请求体返回 `409 idempotency_conflict`。
7. 相同请求仍在执行时返回 `409 request_in_progress`；稍后必须使用完全相同的 ID 和请求体重试。
8. 达到常规限额后，只有已建立且连续的 `session_id` 才能在受限额度内打完当前一局。
9. 网络中断造成的事件缺口只会使连续性未知，不会被视为非法；同一 session 连续三次明确矛盾才返回 `400 session_state_invalid`。
10. 订阅透支会结转：下一个五小时或周窗口先扣旧债，再获得本周期可用额度。
11. FlyAPI 不设置每 Key 并发上限；总容量和过载拒绝由推理服务提供方负责。

若响应丢失或客户端无法确认服务端是否收到请求，应原样重发同一 `request_id` 和请求体；网络重试绝不能更换 ID。

## 9. 错误

进入运行时前的错误使用非 2xx HTTP 状态：

```json
{
  "error": "test_api_model_unavailable",
  "message": "model is not enabled for the test API"
}
```

| HTTP | `error` | 含义 | 扣额度 |
| --- | --- | --- | --- |
| 400 | `bad_request_id` | `request_id` 无效 | 否 |
| 400 | `bad_deadline` | `deadline_ms` 不是正整数 | 否 |
| 400 | `model_rule_unsupported` | 模型与 3P/4P 规则线不匹配 | 否 |
| 400 | `authoritative_state_forbidden` | 提交了隐藏权威场况 | 否 |
| 400 | `state_digest_mismatch` | 事件摘要不匹配 | 否 |
| 400 | `state_replay_invalid` | 事件流无法按规则重放 | 否 |
| 400 | `state_replay_incomplete` | 无法精确重建场况 | 否 |
| 400 | `session_state_invalid` | 同一 session 连续三次明确矛盾 | 否 |
| 401 | `test_api_key_invalid` | Key 缺失、无效、已吊销或已销毁 | 否 |
| 402 | `test_api_quota_exhausted` | 按量额度不可用 | 否 |
| 402 | `test_api_five_hour_quota_exhausted` | 五小时额度不可用 | 否 |
| 402 | `test_api_weekly_quota_exhausted` | 周额度不可用 | 否 |
| 402 | `test_api_subscription_expired` | 订阅已到期 | 否 |
| 403 | `test_api_key_frozen` | Key 尚未通过 `/quota` 激活 | 否 |
| 404 | `test_api_disabled` | 测试 API 未启用 | 否 |
| 404 | `test_api_model_unavailable` | 当前 Key 无法使用该模型 | 否 |
| 409 | `idempotency_conflict` | 相同 ID 对应了不同请求体 | 否 |
| 409 | `request_in_progress` | 相同请求仍在执行 | 否 |
| 422 | `decision_not_required` | 当前本席不需要决策 | 否 |
| 503 | `model_unavailable` | 当前没有可路由运行实例 | 否 |
| 503 | `model_policy_unavailable` | 部署政策不允许测试 API 使用 | 否 |

JSON 类型错误、缺少必填字段或未知字段，可能由 HTTP JSON 提取层直接返回 `400` 或 `422`；它们不会进入运行时，也不扣额度。服务端 `5xx` 的消息统一脱敏为 `internal server error`。

客户端只应记录 HTTP 状态、稳定错误码、自身 `request_id` 和时间，不要记录完整 Key 或包含隐藏信息的请求体。

## 10. 接入检查表

1. Key 只保存在系统密钥库或服务端 secret 中。
2. 收到新 Key 后先调用一次 `/quota` 激活，再读取 `/models`。
3. 只提交从开局开始的完整 observed 事件历史，不生成或上传合法动作。
4. 每个新决策点生成一个稳定、唯一的 `request_id`。
5. 网络重试原样复用 ID 和完整请求体。
6. HTTP 200 后仍检查 `attempt.status`。
7. 成功时执行 `attempt.action`，并通过 `selected_action_id` 验证。
8. 定期查询 `/quota`；额度不可用时停止发起新决策。
9. 忽略不认识的响应字段，以兼容 v1 的加性扩展。

## 11. 版本规则

当前 URL 主版本为 `/beta/v1`，成功响应协议为 `flya-test-api-v1`。

v1 内只做向后兼容的加性扩展。删除字段，或改变字段语义、认证方式、计费或幂等语义时，必须使用新的 URL 主版本。
