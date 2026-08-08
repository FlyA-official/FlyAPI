# FlyA Test API クライアント統合ガイド v1

[English](README.md) | [简体中文](README.zh-CN.md) | [日本語](README.ja.md)

| 項目 | 値 |
| --- | --- |
| ステータス | **パブリックベータ** |
| Base URL | `https://api.nashout.com` |
| API プレフィックス | `/beta/v1` |
| プロトコル | `flya-test-api-v1` |

機械可読契約：[OpenAPI 3.1](openapi.json) · [イベント JSON Schema](schemas/flya-mahjong-events-v2.schema.json) · [標準リクエスト／レスポンス fixture](examples/)

## 1. 認証とプラットフォーム境界

すべてのリクエストに次のヘッダーが必要です。

```http
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

Key は `flyat_` で始まります。TLS 証明書の検証を有効にしたまま使用してください。Key をブラウザ側コード、URL、ログ、ソースリポジトリに保存してはいけません。ブラウザ CORS は意図的に提供していません。

FlyAPI が受け取るのは、プラットフォーム非依存の `flya-mahjong-events-v2` observed 標準イベントストリームだけです。ゲーム、牌譜、クライアント固有のプロトコルをこの標準入力へ変換する処理は統合側の責任であり、本リポジトリの範囲外です。リクエストにプラットフォーム名を含めないでください。

クライアントに FlyTable のパッケージやソースコードは不要です。本 JSON 契約を実装し、完全な標準イベント履歴を送信してください。

局面の復元と合法手の計算はサーバーが行います。クライアントは `legal_actions`、`legal_digest`、牌山、他家の非公開手牌、tensor、mask、obs、その他のモデル内部入力を送信してはいけません。

## 2. エンドポイント

| メソッド | パス | 用途 |
| --- | --- | --- |
| `GET` | `/beta/v1/models` | 現在の Key で利用可能なモデルと呼び出しコストを取得 |
| `GET` | `/beta/v1/quota` | 新しい Key を有効化し、状態とクォータを取得 |
| `POST` | `/beta/v1/decision` | 完全な可視イベント履歴を送信して打牌判断を要求 |

Key がない、無効、失効済み、または破棄済みの場合：

```json
{
  "error": "test_api_key_invalid",
  "message": "test API key is invalid"
}
```

## 3. モデル一覧

```bash
curl -H "Authorization: Bearer $FLYAT_KEY" \
  https://api.nashout.com/beta/v1/models
```

レスポンス例：

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

| フィールド | 意味 |
| --- | --- |
| `model_id` | `/decision` で指定する安定したモデル ID |
| `display_name` | ユーザー向け表示名 |
| `rule_line` | `riichi4p` または `riichi3p` |
| `provider` | モデル提供元の識別子 |
| `multiplier` | 通常クォータ単位のコストを表す十進文字列 |
| `cost_milliunits` | 1/1000 クォータ単位の呼び出しコスト |
| `is_default` | そのルールラインのサーバーデフォルトか |
| `available` | 現在ルーティング可能なランタイムがあるか |
| `unavailable_reason` | 利用不可時の安定した理由コード。それ以外は `null` |

起動時またはモデル選択前にこのエンドポイントを取得してください。可用性、コスト、表示名、デフォルトを恒久的にハードコードしないでください。結果は現在の Key に応じてフィルタリングされ、`Cache-Control: private, no-store` と `Vary: Authorization` が返ります。異なる Key 間でキャッシュを共有してはいけません。

## 4. クォータと有効化

```bash
curl -H "Authorization: Bearer $FLYAT_KEY" \
  https://api.nashout.com/beta/v1/quota
```

`GET /quota` は唯一の有効化エンドポイントです。最初に成功した照会で有効期間が開始され、`activated_now: true` が返ります。以降は `false` です。レスポンスには `Cache-Control: no-store` が付きます。

従量制（PAYGO）の例：

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

サブスクリプションの例：

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

クォータ値は小数点以下最大 3 桁の十進文字列です。制限付き継続クレジットを使用中は `remaining` が負になる場合があります。decimal 型または整数 milliunits を使い、二進浮動小数点で残高を累積しないでください。

PAYGO を使い切るか Key の期限が切れると `grace` 状態になります。3 日間の猶予期間は `/quota` を照会でき、`destroy_at` と `destroy_in_seconds` が返ります。破棄後は `401 test_api_key_invalid` です。Key のフィンガープリントは永久保存されるため、古い Key が再発行されることはありません。

## 5. 判断リクエスト

```http
POST /beta/v1/decision HTTP/1.1
Host: api.nashout.com
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

完全で再生可能なリクエストは [examples/decision-4p-discard.request.json](examples/decision-4p-discard.request.json) にあります。

```bash
curl -X POST https://api.nashout.com/beta/v1/decision \
  -H "Authorization: Bearer $FLYAT_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @examples/decision-4p-discard.request.json
```

### トップレベルフィールド

| フィールド | 必須 | ルール |
| --- | --- | --- |
| `request_id` | はい | Key 内で一意な冪等 ID。1～200 文字、`A-Z a-z 0-9 . _ : -` のみ |
| `session_id` | いいえ | 同一対局で安定した ID。8～128 文字、同じ文字セット。連続性と制限付き継続クレジットにのみ使用 |
| `model_id` | いいえ | 省略時はルールラインのサーバーデフォルト。指定時は `/models` の利用可能な ID |
| `state` | はい | 完全な `flya-mahjong-events-v2` 自家可視状態エンベロープ |
| `match_context` | いいえ | 任意の対局長メタデータ：`{"length":"single|east|half"}`。省略時は `unknown` |
| `deadline_ms` | いいえ | 正の整数。デフォルトおよび実効上限は 30000 ms |

未知のトップレベルフィールドは拒否されます。レスポンスの `model_selection` は `server_default` または `explicit` です。

### `state` エンベロープ

| フィールド | ルール |
| --- | --- |
| `schema` | `flya-mahjong-events-v2` |
| `rule_line` | `riichi4p` または `riichi3p` |
| `rule_profile` | 任意。通常は省略し、サーバー既定のルールを使用 |
| `viewer_seat` | 四麻は `0..3`、三麻は `0..2` |
| `source.kind` | `observed` 必須。`authoritative` は拒否 |
| `from_seq` | `0` 必須。各リクエストはゲーム開始からの完全な履歴を含む |
| `to_seq` | `events.length` と一致 |
| `events` | 時系列順の、自家から見える canonical MJAI イベント |
| `state_digest` | 任意。省略時はサーバーが計算し、指定時は `events` と一致する必要あり |

サーバーは完全な履歴を検証してダイジェストを計算し、最新の `start_kyoku` から現在局を再生し、判断が必要な局面であることを確認して唯一の合法手テーブルを生成します。observed ストリームで他家の親の配牌 14 枚目が省略される場合、サーバーが内部で不明牌を補完します。クライアント側で推測しないでください。

プライバシー要件：

- 他家の配牌は `"?"` を使用します。
- 他家のツモ牌は `"?"` を使用します。
- `scores`、`deltas`、`tehais` の席数はルールラインと一致させます。
- 牌山、他家の実手牌、authoritative seed、obs、tensor、mask、モデル内部特徴を送信してはいけません。

### 標準イベントオブジェクト

機械可読の正本は[イベント JSON Schema](schemas/flya-mahjong-events-v2.schema.json)です。次の表は受理される全 fact-event を示します。未知のイベント型と未知のフィールドは拒否されます。

| `type` | 必須標準フィールド | ルールライン |
| --- | --- | --- |
| `start_game` | `names`。observed の `seed` は省略または `null`。`id` は任意 | 4P/3P |
| `start_kyoku` | `bakaze`, `dora_marker`, `kyoku`, `honba`, `kyotaku`, `oya`, `scores`, `tehais` | 4P/3P |
| `tsumo` | `actor`, `pai` | 4P/3P |
| `dealer_opening` | `actor`, `pai` | 4P/3P、v2 |
| `dahai` | `actor`, `pai`, `tsumogiri` | 4P/3P |
| `dealer_opening_dahai` | `actor`, `pai` | 4P/3P、v2 |
| `chi` | `actor`, `target`, `pai`, 2 枚の `consumed` | 4P のみ |
| `pon` | `actor`, `target`, `pai`, 2 枚の `consumed` | 4P/3P |
| `daiminkan` | `actor`, `target`, `pai`, 3 枚の `consumed` | 4P/3P |
| `kakan` | `actor`, `pai`, 3 枚の `consumed` | 4P/3P |
| `ankan` | `actor`, 4 枚の `consumed` | 4P/3P |
| `kita` | `actor`, `pai` (`N`) | 3P のみ |
| `dora` | `dora_marker` | 4P/3P |
| `reach` | `actor` | 4P/3P |
| `reach_accepted` | `actor`。`scores`, `deltas` は任意 | 4P/3P |
| `hora` | `actor`, `target`。Schema 所定の精算／詳細フィールドは任意 | 4P/3P |
| `ryukyoku` | `deltas`, `reason`, `tenpais`, `tehais`, `scores` は任意 | 4P/3P |
| `end_kyoku` | 追加フィールドなし | 4P/3P |
| `end_game` | `scores`, `rankings` は任意 | 4P/3P |

新規統合で `state_digest` を送る必要はありません。互換実装では FlyA の compact canonical イベント配列 JSON に対する FNV-1a 64 を指定できますが、送信前に [examples/state-digest-vectors.json](examples/state-digest-vectors.json) で検証してください。

実行可能 fixture は [4P 打牌](examples/decision-4p-discard.request.json)、[4P チー応答ウィンドウ](examples/decision-4p-chi.request.json)、[3P 北抜きターン](examples/decision-3p-kita.request.json)を含みます。レスポンス fixture は[成功](examples/decision-success.response.json)、[失敗](examples/decision-failure.response.json)、[タイムアウト](examples/decision-timeout.response.json)、[abstain](examples/decision-abstain.response.json)、[ランタイム前エラー](examples/pre-runtime-error.response.json)を含みます。

## 6. 公開アクション形式

合法手は成功レスポンスにのみ含まれます。各候補の `action_id` は action オブジェクトの外側にあります。

| `type` | action オブジェクト | 意味 |
| --- | --- | --- |
| `dahai` | `{"type":"dahai","pai":"5pr","tsumogiri":false}` | 通常の打牌 |
| `dealer_opening_dahai` | `{"type":"dealer_opening_dahai","pai":"5pr"}` | 親が 14 枚の配牌から行う最初の打牌 |
| `riichi_dahai` | `{"type":"riichi_dahai","pai":"5pr","tsumogiri":false}` | リーチ打牌。通常打牌とは別アクション |
| `dealer_opening_riichi_dahai` | `{"type":"dealer_opening_riichi_dahai","pai":"5pr"}` | 親の配牌直後のリーチ打牌 |
| `chi` | `{"type":"chi","pai":"3m","consumed":["1m","2m"]}` | チー。四麻のみ |
| `pon` | `{"type":"pon","pai":"E","consumed":["E","E"]}` | ポン |
| `daiminkan` | `{"type":"daiminkan","pai":"E","consumed":["E","E","E"]}` | 大明槓 |
| `ankan` | `{"type":"ankan","pai":"5p","consumed":["5p","5p","5p","5p"]}` | 暗槓 |
| `kakan` | `{"type":"kakan","pai":"E","consumed":["E","E","E"]}` | 加槓 |
| `kita` | `{"type":"kita"}` | 北抜き。三麻のみ |
| `tsumo` | `{"type":"tsumo","pai":"2s"}` | ツモ和了。親の配牌直後は `pai` が省略される場合あり |
| `ron` | `{"type":"ron","pai":"2s","target":1}` | ロン和了。`target` は放銃席。親の配牌直後は `pai` が省略される場合あり |
| `kyushukyuhai` | `{"type":"kyushukyuhai"}` | 九種九牌 |
| `pass_all` | `{"type":"pass_all"}` | 現在の応答機会をすべて見送る |

`chi.pai` は上家が捨てた鳴き対象牌、`chi.consumed` は自家の手牌から使用する 2 枚です。`dealer_opening_*` はアクションのタイミングを表し、ゲームプラットフォームを表しません。その最初の打牌より前に通常の `tsumo` イベントはありません。

牌文字列は FlyTable 形式です。数牌は `1m`～`9m`、`1p`～`9p`、`1s`～`9s`、赤五は `5mr`、`5pr`、`5sr`、字牌は `E S W N P F C` です。`"?"` はイベントストリームの非公開牌にのみ使用でき、アクションには使用できません。

## 7. 判断レスポンス

成功例：

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

`attempt.action` を実行してください。また、`attempt.actions` 内で `action_id == attempt.selected_action_id` の項目と完全に一致することを確認してください。`selected_index` は互換用フィールドであり、`selected_action_id` の代わりにはなりません。

`attempt.actions` はサーバーが返す最終候補リストです。各項目は安定した `action_id`、アクション、`probability` を含み、確率の合計は 1 です。候補数や並び順を仮定しないでください。

ランタイム実行開始後は、失敗、タイムアウト、abstain でも HTTP は `200` です。必ず `attempt.status` を確認してください。

- `success`
- `failure`
- `timeout`
- `abstain`

成功以外の `attempt.error.code` は、機械処理向けの安定したエラーコードです。代表例は `runtime_timeout`、`engine_unavailable`、`state_sync_required`、`runtime_protocol_error`、`invalid_decision`、`probability_unavailable`、`model_abstained`、各種 `*_capacity_exhausted` です。未知のコードは通常の失敗として扱ってください。

## 8. 課金、対局継続、冪等性

1. 構造、モデル、ルーティング検証後、サーバーは判断処理前に、選択モデルのコストに相当する同時実行枠を原子的に予約します。
2. サーバー側の合法手検証を通過したすべての `success` アクション応答は、合法手が 1 つだけの場合も含め、選択モデルのレートでクォータを消費します。`failure`、`timeout`、`abstain` は `quota.consumed="0.000"` を返します。
3. 認証失敗、不正形式、非対応モデル、ルート利用不可など、判断処理前に拒否されたリクエストは消費しません。
4. `request_id` は同一 API Key 内の冪等キーです。
5. 同じ ID と完全に同じ本文を再送すると、最初の終端結果を `quota.replay=true`、`quota.consumed="0.000"` で返します。
6. 同じ ID で本文が異なる場合は `409 idempotency_conflict` です。
7. 同一リクエストが実行中の場合は `409 request_in_progress` です。後で同じ ID と本文をそのまま再送してください。
8. 通常クォータ到達後は、確立済みで連続した `session_id` のみが、制限付き継続クレジットで現在の対局を完了できます。
9. session の連続性は、対局継続クレジットの根拠としてのみ使われます。ネットワーク切断、履歴の切り詰め、同期ずれにより超過利用資格を失うことはありますが、それだけでリクエストが不正になることはありません。各標準イベントストリームは単独で合法な状態へ再生できる必要があります。
10. サブスクリプションの超過分は繰り越されます。次の 5 時間または週次ウィンドウは、先に旧債を差し引いてから新しい利用可能枠を付与します。
11. FlyAPI は Key ごとの同時実行上限を設けません。全体容量と過負荷拒否は推論サービス提供側が管理します。

ゲームプラットフォーム側ですでに処理済み、または選択が不要とクライアントが判断できる場合は `/decision` を呼び出さないでください。このようなリクエストでも `success` が返れば通常どおり課金されます。

レスポンスを失った場合、またはサーバーが受信したか判断できない場合は、同じ `request_id` と本文をそのまま再送してください。ネットワーク再試行で新しい ID を生成してはいけません。

## 9. エラー

ランタイム実行前のエラーは非 2xx HTTP と次の形式で返ります。

```json
{
  "error": "test_api_model_unavailable",
  "message": "model is not enabled for the test API"
}
```

| HTTP | `error` | 意味 | 課金 |
| --- | --- | --- | --- |
| 400 | `bad_request_id` | `request_id` が不正 | なし |
| 400 | `bad_deadline` | `deadline_ms` が正の整数ではない | なし |
| 400 | `model_rule_unsupported` | モデルと三麻／四麻ルールラインが不一致 | なし |
| 400 | `authoritative_state_forbidden` | 非公開の authoritative 状態を送信 | なし |
| 400 | `state_digest_mismatch` | イベントダイジェスト不一致 | なし |
| 400 | `state_replay_invalid` | イベントストリームを再生できない | なし |
| 400 | `state_replay_incomplete` | 状態を正確に復元できない | なし |
| 401 | `test_api_key_invalid` | Key がない、無効、失効、または破棄済み | なし |
| 402 | `test_api_quota_exhausted` | PAYGO クォータを利用できない | なし |
| 402 | `test_api_five_hour_quota_exhausted` | 5 時間クォータを利用できない | なし |
| 402 | `test_api_weekly_quota_exhausted` | 週次クォータを利用できない | なし |
| 402 | `test_api_subscription_expired` | サブスクリプション期限切れ | なし |
| 403 | `test_api_key_frozen` | `/quota` で Key を有効化していない | なし |
| 404 | `test_api_disabled` | Test API が無効 | なし |
| 404 | `test_api_model_unavailable` | この Key ではモデルを利用できない | なし |
| 409 | `idempotency_conflict` | 同じ ID に異なる本文を使用 | なし |
| 409 | `request_in_progress` | 同一リクエストが実行中 | なし |
| 429 | `test_api_client_error_rate_limited` | この Key から短時間に不正リクエストが多発。`Retry-After` に従って待機 | なし |
| 422 | `decision_not_required` | 現在は自家の判断が不要 | なし |
| 422 | `decision_window_inconsistent` | 末尾の通知イベント前後で合法手が一致しない | なし |
| 503 | `model_unavailable` | ルーティング可能なモデルがない | なし |
| 503 | `model_policy_unavailable` | デプロイポリシーが Test API 利用を拒否 | なし |

JSON 型エラー、必須フィールド欠落、未知フィールドは HTTP JSON 抽出層から `400` または `422` で返る場合があります。ランタイムには到達せず、課金されません。サーバー側 `5xx` のメッセージは `internal server error` に固定して秘匿されます。

不正リクエストを繰り返すと、その Key の `/decision` が一時的に制限される場合があります。`429` の `Retry-After` に従い、原因を修正してから再開してください。この一時保護で Key が失効することはありません。

ログには HTTP ステータス、安定したエラーコード、自分の `request_id`、時刻のみを記録してください。完全な Key や非公開情報を含む本文を記録してはいけません。

## 10. 統合チェックリスト

1. Key は OS の資格情報ストアまたはサーバー側 secret にのみ保存します。
2. 新しい Key は最初に `/quota` で有効化し、その後 `/models` を取得します。
3. ゲーム開始からの完全な observed イベント履歴だけを送信し、合法手を生成・送信しません。
4. 新しい判断点ごとに安定した一意の `request_id` を生成します。
5. ネットワーク再試行では同じ ID と本文をそのまま使用します。
6. HTTP 200 でも `attempt.status` を確認します。
7. 成功時は `attempt.action` を実行し、`selected_action_id` で整合性を確認します。
8. プラットフォーム処理済み、または選択不要の局面では `/decision` を呼び出しません。成功したアクション応答はすべて課金されます。
9. `429` の `Retry-After` に従い、不正リクエストの原因を修正してから再開します。
10. `/quota` を定期的に取得し、利用不可時は新しい判断要求を停止します。
11. 未知のレスポンスフィールドを無視し、v1 の追加拡張に対応します。

## 11. バージョニング

現在の URL メジャーバージョンは `/beta/v1`、成功レスポンスのプロトコルは `flya-test-api-v1` です。

新規統合では `flya-mahjong-events-v2` を使用してください。凍結済み v1 イベントエンベロープは、v2 の dealer-opening イベントを使わない場合に限り引き続き受理されます。

v1 内の変更は追加的かつ後方互換です。フィールド削除、フィールドの意味、認証または冪等性の変更、および本書でゼロ消費と定めた結果への課金には、新しい URL メジャーバージョンが必要です。
