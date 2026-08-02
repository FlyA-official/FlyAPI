# FlyA Test API Client Integration Guide v1

[English](README.md) | [简体中文](README.zh-CN.md) | [日本語](README.ja.md)

| Item | Value |
| --- | --- |
| Status | **Public beta** |
| Base URL | `https://api.nashout.com` |
| API prefix | `/beta/v1` |
| Protocol | `flya-test-api-v1` |

## 1. Authentication and platform boundary

Every request requires:

```http
Authorization: Bearer <FLYAT_KEY>
Content-Type: application/json
```

Keys begin with `flyat_`. Keep TLS certificate verification enabled. Never put a key in browser code, a URL, logs, or a source repository. Browser CORS is intentionally unavailable.

FlyAPI is platform-independent. It does not accept, identify, or restrict Mahjong Soul, Tenhou, Riichi City, Amatsuki, or any other game platform. The client bridge converts the platform protocol into the `flya-mahjong-events-v1` observed event stream described below. Do not send a platform name.

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

### Top-level fields

| Field | Required | Rules |
| --- | --- | --- |
| `request_id` | Yes | Idempotency ID unique within the key; 1–200 characters from `A-Z a-z 0-9 . _ : -` |
| `session_id` | No | Stable game ID; 8–128 characters from the same set. Used only for continuity and bounded continuation credit |
| `model_id` | No | Omit to use the server default for the rule line; otherwise select an available ID from `/models` |
| `state` | Yes | Complete `flya-mahjong-events-v1` visible-state envelope |
| `deadline_ms` | No | Positive integer; default and maximum effective wait are 30000 ms |

Unknown top-level fields are rejected. `model_selection` in the response is `server_default` or `explicit`.

### State envelope

| Field | Rules |
| --- | --- |
| `schema` | `flya-mahjong-events-v1` |
| `rule_line` | `riichi4p` or `riichi3p` |
| `viewer_seat` | `0..3` for four-player; `0..2` for three-player |
| `source.kind` | Must be `observed`; `authoritative` is rejected |
| `from_seq` | Must be `0`; every request contains the complete history from game start |
| `to_seq` | Must equal `events.length` |
| `events` | Canonical MJAI events visible to the requesting seat, in time order |
| `state_digest` | Stable FlyTable digest of `events` |

The server validates the complete history and digest, replays the current hand from its latest `start_kyoku`, verifies that a decision is required, and derives the only legal action table. If an observed stream omits an opponent dealer's hidden fourteenth starting tile, the server supplies an unknown tile internally; the client must not infer it.

Privacy requirements:

- Opponents' initial hands use `"?"`.
- Opponents' drawn tiles use `"?"`.
- Seat widths in `scores`, `deltas`, and `tehais` must match the rule line.
- Never send wall state, another player's real hand, authoritative seeds, observations, tensors, masks, or private model features.

Do not compute the digest from arbitrary JSON text. It is FNV-1a 64 over the compact UTF-8 JSON emitted by FlyTable after typed serialization. Rust clients should call:

```rust
use flytable_protocol::compute_state_digest;

let state_digest = compute_state_digest(&events);
```

Other languages must reproduce the same field order and canonical serialization and verify their implementation against a known-good FlyTable fixture.

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

## 8. Billing, continuation, and idempotency

1. After structural, model, and routing validation, the server atomically charges the selected model cost before runtime execution.
2. `success`, `failure`, `timeout`, and `abstain` consume the charge after runtime execution begins.
3. Authentication errors, malformed requests, unsupported models, unavailable routes, and requests rejected before runtime do not consume quota.
4. `request_id` is the idempotency key within one API key.
5. Reusing the same `request_id` with the exact same body returns the original terminal result with `quota.replay=true` and `quota.consumed="0.000"`.
6. Reusing it with a different body returns `409 idempotency_conflict`.
7. If the identical request is still executing, the server returns `409 request_in_progress`; retry later with the exact same ID and body.
8. At a normal quota boundary, only an established, continuous `session_id` may use bounded continuation credit to finish the current game.
9. A network gap makes continuity unknown, not illegal. Three consecutive explicit contradictions in the same session return `400 session_state_invalid`.
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
| 400 | `session_state_invalid` | Three consecutive explicit session contradictions | No |
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

Within v1, changes are additive and backward-compatible. Removing fields or changing field meaning, authentication, billing, or idempotency semantics requires a new URL major version.
