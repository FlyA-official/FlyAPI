# FlyA Test API Client Integration Guide v1

[English](README.md) | [简体中文](README.zh-CN.md) | [日本語](README.ja.md)

| Item | Value |
| --- | --- |
| Status | **Public beta** |
| Base URL | `https://api.nashout.com` |
| API prefix | `/beta/v1` |
| Protocol | `flya-test-api-v1` |

Machine-readable contracts: [OpenAPI 3.1](openapi.json) · [event JSON Schema](schemas/flya-mahjong-events-v2.schema.json) · [standard request and response fixtures](examples/)

## 1. Authentication and platform boundary

Every request requires:

```http
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

Keys begin with `flyat_`. Keep TLS certificate verification enabled. Never put a key in browser code, a URL, logs, or a source repository. Browser CORS is intentionally unavailable.

FlyAPI accepts only the platform-independent `flya-mahjong-events-v2` observed event stream. Converting any game, log, or client protocol into this standard input is the integrator's responsibility and is outside this repository. Do not send a platform name.

No FlyTable package or source code is required on the client. Implement the JSON contract and send a complete standard event history.

The server reconstructs the game state and computes all legal actions. Clients must not send `legal_actions`, `legal_digest`, wall data, hidden hands, tensors, masks, observations, or other model-private inputs.

## 2. Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/beta/v1/models` | List models available to the current key and their per-call cost |
| `GET` | `/beta/v1/quota` | Activate a new key and read its status and quota |
| `POST` | `/beta/v1/decision` | Submit the complete visible event history and request a decision |

Invalid, revoked, destroyed, or missing keys return:

```json
{
  "error": "test_api_key_invalid",
  "message": "test API key is invalid"
}
```

## 3. Models

```bash
curl -H "Authorization: Bearer $FLYAT_KEY" \
  https://api.nashout.com/beta/v1/models
```

Example response:

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

| Field | Meaning |
| --- | --- |
| `model_id` | Stable ID accepted by `/decision` |
| `display_name` | User-facing name |
| `rule_line` | `riichi4p` or `riichi3p` |
| `provider` | Model provider identifier |
| `multiplier` | Cost in normal quota units, encoded as a decimal string |
| `cost_milliunits` | Cost in thousandths of a quota unit |
| `is_default` | Whether this is the server default for its rule line |
| `available` | Whether a runtime is currently routable |
| `unavailable_reason` | Stable reason code when unavailable, otherwise `null` |

Read this endpoint at startup or before model selection. Do not permanently hard-code availability, cost, display names, or defaults. Results are filtered by the current key and return `Cache-Control: private, no-store` and `Vary: Authorization`; never cache them across keys.

## 4. Quota and activation

```bash
curl -H "Authorization: Bearer $FLYAT_KEY" \
  https://api.nashout.com/beta/v1/quota
```

`GET /quota` is the only activation endpoint. The first successful query starts the validity period and returns `activated_now: true`. Later queries return `false`. The response uses `Cache-Control: no-store`.

PAYGO example:

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

Subscription example:

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

Quota values are decimal strings with at most three fractional digits. `remaining` may be negative while bounded continuation credit is in use. Use decimal arithmetic or integer milliunits; do not accumulate balances with binary floating point.

An exhausted PAYGO key or an expired key enters `grace`. During the three-day grace period, `/quota` remains available and returns `destroy_at` and `destroy_in_seconds`. After destruction, the key returns `401 test_api_key_invalid`. Key fingerprints are retained permanently, so an old key is never reissued.

## 5. Decision request

```http
POST /beta/v1/decision HTTP/1.1
Host: api.nashout.com
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

A complete, replayable request is provided in [examples/decision-4p-discard.request.json](examples/decision-4p-discard.request.json):

```bash
curl -X POST https://api.nashout.com/beta/v1/decision \
  -H "Authorization: Bearer $FLYAT_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @examples/decision-4p-discard.request.json
```

### Top-level fields

| Field | Required | Rules |
| --- | --- | --- |
| `request_id` | Yes | Idempotency ID unique within the key; 1–200 characters from `A-Z a-z 0-9 . _ : -` |
| `session_id` | No | Stable game ID; 8–128 characters from the same set. Used only for continuity and bounded continuation credit |
| `model_id` | No | Omit to use the server default for the rule line; otherwise select an available ID from `/models` |
| `state` | Yes | Complete `flya-mahjong-events-v2` visible-state envelope |
| `match_context` | No | Optional match-length metadata: `{"length":"single|east|half"}`; omitted means `unknown` |
| `deadline_ms` | No | Positive integer; default and maximum effective wait are 30000 ms |

Unknown top-level fields are rejected. `model_selection` in the response is `server_default` or `explicit`.

### State envelope

| Field | Rules |
| --- | --- |
| `schema` | `flya-mahjong-events-v2` |
| `rule_line` | `riichi4p` or `riichi3p` |
| `rule_profile` | Optional; normally omit it to use the server's default rules |
| `viewer_seat` | `0..3` for four-player; `0..2` for three-player |
| `source.kind` | Must be `observed`; `authoritative` is rejected |
| `from_seq` | Must be `0`; every request contains the complete history from game start |
| `to_seq` | Must equal `events.length` |
| `events` | Canonical MJAI events visible to the requesting seat, in time order |
| `state_digest` | Optional. Omit it and the server computes it; if supplied, it must match `events` |

The server validates the complete history, computes its digest, replays the current hand from its latest `start_kyoku`, verifies that a decision is required, and derives the only legal action table. If an observed stream omits an opponent dealer's hidden fourteenth starting tile, the server supplies an unknown tile internally; the client must not infer it.

Privacy requirements:

- Opponents' initial hands use `"?"`.
- Opponents' drawn tiles use `"?"`.
- Seat widths in `scores`, `deltas`, and `tehais` must match the rule line.
- Never send wall state, another player's real hand, authoritative seeds, observations, tensors, masks, or private model features.

### Standard event objects

The machine-readable authority is the [event JSON Schema](schemas/flya-mahjong-events-v2.schema.json). The following table shows every accepted fact-event type; unknown fields and event types are rejected.

| `type` | Required standard fields | Rule line |
| --- | --- | --- |
| `start_game` | `names`; observed `seed` is omitted or `null`; optional `id` | 4P/3P |
| `start_kyoku` | `bakaze`, `dora_marker`, `kyoku`, `honba`, `kyotaku`, `oya`, `scores`, `tehais` | 4P/3P |
| `tsumo` | `actor`, `pai` | 4P/3P |
| `dealer_opening` | `actor`, `pai` | 4P/3P, v2 |
| `dahai` | `actor`, `pai`, `tsumogiri` | 4P/3P |
| `dealer_opening_dahai` | `actor`, `pai` | 4P/3P, v2 |
| `chi` | `actor`, `target`, `pai`, two-tile `consumed` | 4P only |
| `pon` | `actor`, `target`, `pai`, two-tile `consumed` | 4P/3P |
| `daiminkan` | `actor`, `target`, `pai`, three-tile `consumed` | 4P/3P |
| `kakan` | `actor`, `pai`, three-tile `consumed` | 4P/3P |
| `ankan` | `actor`, four-tile `consumed` | 4P/3P |
| `kita` | `actor`, `pai` (`N`) | 3P only |
| `dora` | `dora_marker` | 4P/3P |
| `reach` | `actor` | 4P/3P |
| `reach_accepted` | `actor`; optional `scores`, `deltas` | 4P/3P |
| `hora` | `actor`, `target`; optional settlement/detail fields defined by the schema | 4P/3P |
| `ryukyoku` | optional `deltas`, `reason`, `tenpais`, `tehais`, `scores` | 4P/3P |
| `end_kyoku` | no additional fields | 4P/3P |
| `end_game` | optional `scores`, `rankings` | 4P/3P |

`state_digest` is not required for new integrations. Compatibility implementations may provide it as FNV-1a 64 over FlyA's compact canonical event-array JSON; verify against [examples/state-digest-vectors.json](examples/state-digest-vectors.json) before sending it.

Runnable fixtures cover a [4P discard](examples/decision-4p-discard.request.json), [4P chi response window](examples/decision-4p-chi.request.json), and [3P kita turn](examples/decision-3p-kita.request.json). Response fixtures cover [success](examples/decision-success.response.json), [failure](examples/decision-failure.response.json), [timeout](examples/decision-timeout.response.json), [abstain](examples/decision-abstain.response.json), and a [pre-runtime error](examples/pre-runtime-error.response.json).

## 6. Public action forms

Legal actions appear only in a successful response. Each candidate also has an `action_id` outside the action object.

| `type` | Action object | Meaning |
| --- | --- | --- |
| `dahai` | `{"type":"dahai","pai":"5pr","tsumogiri":false}` | Discard |
| `dealer_opening_dahai` | `{"type":"dealer_opening_dahai","pai":"5pr"}` | Dealer's first discard from the 14-tile opening hand |
| `riichi_dahai` | `{"type":"riichi_dahai","pai":"5pr","tsumogiri":false}` | Riichi discard; distinct from a normal discard |
| `dealer_opening_riichi_dahai` | `{"type":"dealer_opening_riichi_dahai","pai":"5pr"}` | Dealer's opening riichi discard |
| `chi` | `{"type":"chi","pai":"3m","consumed":["1m","2m"]}` | Chi; four-player only |
| `pon` | `{"type":"pon","pai":"E","consumed":["E","E"]}` | Pon |
| `daiminkan` | `{"type":"daiminkan","pai":"E","consumed":["E","E","E"]}` | Open kan |
| `ankan` | `{"type":"ankan","pai":"5p","consumed":["5p","5p","5p","5p"]}` | Closed kan |
| `kakan` | `{"type":"kakan","pai":"E","consumed":["E","E","E"]}` | Added kan |
| `kita` | `{"type":"kita"}` | North extraction; three-player only |
| `tsumo` | `{"type":"tsumo","pai":"2s"}` | Self-draw win; `pai` may be omitted in a dealer-opening state |
| `ron` | `{"type":"ron","pai":"2s","target":1}` | Ron; `target` is the discarding seat and `pai` may be omitted in a dealer-opening state |
| `kyushukyuhai` | `{"type":"kyushukyuhai"}` | Abortive draw for nine terminals and honors |
| `pass_all` | `{"type":"pass_all"}` | Decline the current response window |

For `chi`, `pai` is the tile discarded by the player on the left, and `consumed` contains the two tiles taken from the client's hand. `dealer_opening_*` describes timing, not a game platform; no normal `tsumo` event precedes that first discard.

Tile strings follow FlyTable: `1m`–`9m`, `1p`–`9p`, `1s`–`9s`; red fives are `5mr`, `5pr`, and `5sr`; honors are `E S W N P F C`. `"?"` is valid only for hidden tiles in the event stream, never in an action.

## 7. Decision response

Success example:

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

Execute `attempt.action`. Verify that it is identical to the entry in `attempt.actions` whose `action_id` equals `attempt.selected_action_id`. `selected_index` is a compatibility field and must not replace `selected_action_id`.

`attempt.actions` is the final candidate list returned by the server. Every entry contains a stable `action_id`, an action, and `probability`; probabilities sum to 1. Do not assume a candidate count or ordering.

After runtime execution begins, HTTP status remains `200` even if the attempt fails, times out, or abstains. Always inspect `attempt.status`, which is one of:

- `success`
- `failure`
- `timeout`
- `abstain`

For a non-success attempt, `attempt.error.code` is a stable machine-readable code. Common values include `runtime_timeout`, `engine_unavailable`, `state_sync_required`, `runtime_protocol_error`, `invalid_decision`, `probability_unavailable`, `model_abstained`, and the `*_capacity_exhausted` family. Treat unknown codes as ordinary failures for forward compatibility.

## 8. Billing, continuation, and idempotency

1. After structural, model, and routing validation, the server atomically reserves capacity for the selected model cost before runtime execution.
2. Only a `success` result that passes server-side legal-action validation consumes quota. `failure`, `timeout`, and `abstain` return `quota.consumed="0.000"`.
3. Authentication errors, malformed requests, unsupported models, unavailable routes, and requests rejected before runtime do not consume quota.
4. `request_id` is the idempotency key within one API key.
5. Reusing the same `request_id` with the exact same body returns the original terminal result with `quota.replay=true` and `quota.consumed="0.000"`.
6. Reusing it with a different body returns `409 idempotency_conflict`.
7. If the identical request is still executing, the server returns `409 request_in_progress`; retry later with the exact same ID and body.
8. At a normal quota boundary, only an established, continuous `session_id` may use bounded continuation credit to finish the current game.
9. Session continuity is used only as evidence for continuation credit. A network gap, truncated history, or mismatch may remove overdraft eligibility, but does not make a request illegal by itself. Each submitted standard event stream must still replay to a legal state on its own.
10. Subscription overdraft carries forward: the next five-hour or weekly window pays old debt before providing new usable quota.
11. FlyAPI does not impose a per-key concurrency limit; total capacity and overload rejection belong to the inference provider.

If a response is lost or the client cannot tell whether the server received a request, resend the exact same `request_id` and body. Never generate a new ID for a network retry.

## 9. Errors

Pre-runtime errors use a non-2xx HTTP status and this shape:

```json
{
  "error": "test_api_model_unavailable",
  "message": "model is not enabled for the test API"
}
```

| HTTP | `error` | Meaning | Charged |
| --- | --- | --- | --- |
| 400 | `bad_request_id` | Invalid `request_id` | No |
| 400 | `bad_deadline` | `deadline_ms` is not a positive integer | No |
| 400 | `model_rule_unsupported` | Model and 3P/4P rule line do not match | No |
| 400 | `authoritative_state_forbidden` | Hidden authoritative state was submitted | No |
| 400 | `state_digest_mismatch` | Event digest mismatch | No |
| 400 | `state_replay_invalid` | Event stream cannot be replayed | No |
| 400 | `state_replay_incomplete` | State cannot be reconstructed exactly | No |
| 401 | `test_api_key_invalid` | Missing, invalid, revoked, or destroyed key | No |
| 402 | `test_api_quota_exhausted` | PAYGO quota unavailable | No |
| 402 | `test_api_five_hour_quota_exhausted` | Five-hour quota unavailable | No |
| 402 | `test_api_weekly_quota_exhausted` | Weekly quota unavailable | No |
| 402 | `test_api_subscription_expired` | Subscription expired | No |
| 403 | `test_api_key_frozen` | Key has not been activated through `/quota` | No |
| 404 | `test_api_disabled` | Test API is disabled | No |
| 404 | `test_api_model_unavailable` | Model is unavailable to this key | No |
| 409 | `idempotency_conflict` | Same request ID, different body | No |
| 409 | `request_in_progress` | Identical request is still executing | No |
| 422 | `decision_not_required` | No decision is required for this seat | No |
| 422 | `decision_window_inconsistent` | Legal actions differ across trailing announcement events | No |
| 503 | `model_unavailable` | No model runtime is routable | No |
| 503 | `model_policy_unavailable` | Deployment policy rejects Test API use | No |

JSON type errors, missing required fields, and unknown fields may return `400` or `422` from the HTTP JSON extractor. They do not reach runtime and are not charged. Server-side `5xx` messages are redacted to `internal server error`.

Log only the HTTP status, stable error code, your request ID, and timestamp. Never log a full key or a request body containing hidden information.

## 10. Integration checklist

1. Store the key only in a system credential store or server-side secret.
2. Call `/quota` once to activate a new key, then call `/models`.
3. Submit only the complete observed event history from game start; never generate or upload legal actions.
4. Generate one stable, unique `request_id` for each new decision point.
5. Reuse the exact ID and body for network retries.
6. Check `attempt.status` even when HTTP status is `200`.
7. On success, execute `attempt.action` and verify it through `selected_action_id`.
8. Poll `/quota` and stop starting new decisions when quota is unavailable.
9. Ignore unknown response fields for forward-compatible v1 extensions.

## 11. Versioning

The current URL major version is `/beta/v1`, and successful responses identify `flya-test-api-v1`.

Use `flya-mahjong-events-v2` for new integrations. Frozen v1 event envelopes remain accepted only when they do not use v2 dealer-opening events.

Within v1, changes are additive and backward-compatible. Removing fields, changing field meaning, authentication, or idempotency, or charging a result documented here as zero-cost requires a new URL major version.
