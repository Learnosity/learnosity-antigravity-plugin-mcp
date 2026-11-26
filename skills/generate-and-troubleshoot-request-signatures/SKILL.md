---
name: generate-and-troubleshoot-request-signatures
description: Decide between SDK signing and a hand-rolled HMAC, and resolve a rejected Learnosity signature by mapping the reported error code to the one signed input that differs — domain, timestamp, key/secret pairing, or request bytes.
---

# Generate and Troubleshoot Request Signatures

**Goal:** Decide between SDK signing and a hand-rolled HMAC, then resolve a
rejected signature by identifying which signed input differs between the signer
and Learnosity.

**APIs:** `author-api`, `items-api`, `reports-api`, `data-api`

## When to use this skill

- Adding the server-side signing step to a new integration.
- An integration that signs correctly on a development host is rejected once
  deployed.
- A `4100x` error code appears in the browser console or in a Data API
  response body.
- Credentials are being rotated, or content is moving between consumers and
  each side needs its own signature.
- Reviewing signing code for secret exposure.

## Core principle

Learnosity recomputes the signature from the request it received and its own
copy of the secret. A rejection means one input differed between the two
computations; it rarely means the algorithm is wrong.

| Signed input | Set by | Differs when |
| --- | --- | --- |
| `consumer_key` | the provisioned key set | a key from another environment or another consumer is used |
| `domain` | the signer, explicitly or by SDK default | a default such as `localhost` survives onto a deployed host, or the request reaches Learnosity from a different origin |
| `timestamp` | the signing host's clock | the clock is not UTC, or a signed init object is replayed after the window closes |
| `user_id` | the application | the value is edited after signing |
| `expires`, when present | the signer | it is added to the security packet after signing |
| the `request` object | the serializer | the JSON is re-encoded between signing and sending |

Two consequences carry most of the debugging:

- `consumer_secret` is the HMAC key. It is never an element of the pre-hash
  string and never leaves the server, so a wrong or mispaired secret reports
  the same "signatures do not match" as a mistyped domain.
- No input is validated for sense before hashing. A well-formed but wrong
  value fails at verification, not at signing, which is why signing code that
  "looks right" is the normal case in a signature bug.

## Preconditions

- A matched `consumer_key` / `consumer_secret` from the same credential set. A
  consumer may hold several sets.
- The host the request originates from is allowlisted against that consumer in
  Learnosity Console. Allowlist and key changes can take up to 30 minutes to
  take effect, so a correct configuration can be rejected inside that window.
- The signing host's clock is UTC-accurate.
- The secret is readable server-side only.
- An official SDK for the server language, or Learnosity Support engaged if
  none exists — see **install-and-set-up-the-sdk**.

## Choosing an approach

| Approach | Use when | Cost |
| --- | --- | --- |
| SDK `Init(service, security, secret, request[, action])`, then `generate()` | the server language is one of the six with an official SDK | none worth avoiding: the SDK owns element order, timestamp generation and serialization |
| Hand-rolled HMAC-SHA256 | no SDK exists for the language | you own element order, JSON serialization and future signature versions permanently. Contact Learnosity Support before committing to it |

Prefer the SDK, including for the Data API, where the same `Init` call takes
the `action` as its last argument.

## Worked example

One credential set, two environments, two services. Only `domain` and the
regenerated `timestamp` change between environments, and the Data API packet
carries no `user_id`.

```yaml
# Browser-facing service (items-api), staging
security:
  consumer_key: YOUR_CONSUMER_KEY
  domain: staging.your-domain.com   # the origin the browser loads from
  timestamp: "20260923-1030"        # UTC, YYYYMMDD-HHMM, SDK-generated
  user_id: 11111111-2222-3333-4444-aaaaaaaaaaaa
  expires: "20260923-1230"          # optional, and signed: set before signing
# signature added by Init('items', security, CONSUMER_SECRET, request)

# Same credential set, production: only the domain differs
security:
  consumer_key: YOUR_CONSUMER_KEY
  domain: www.your-domain.com
  timestamp: "20260923-1030"
  user_id: 11111111-2222-3333-4444-aaaaaaaaaaaa

# Server-to-server (data-api): no user_id in the packet
security:
  consumer_key: YOUR_CONSUMER_KEY
  domain: www.your-domain.com
  timestamp: "20260923-1030"
# signature added by Init('data', security, CONSUMER_SECRET, request, 'get')
```

A signature is not portable across those packets. Re-sign per environment, per
service and per request.

## Implementation

- Set `domain` explicitly in every environment rather than relying on an SDK
  default. The PHP reference documents a `localhost` default, and a default is
  what carries a development value into production unnoticed.
- Let the SDK generate `timestamp`. It must be UTC in `YYYYMMDD-HHMM`; a
  local-time clock yields a well-formed value outside the accepted window.
- The timestamp window and session validity are separate limits. The timestamp
  must sit within three hours of current UTC, while a signed session's default
  validity is 168 hours, so a cached init object fails on its timestamp long
  before `expires` is reached.
- `expires` is hashed with the rest of the packet. Add it before calling the
  SDK; adding it to the returned object invalidates the signature.
- Serialize the `request` object once and send those exact bytes. A framework
  that re-encodes JSON on the way out — different key order, different
  escaping — changes the hash without changing any value.
- Derive `security.user_id` and the request's learner id from one variable.
  Only the security copy is signed, so a divergence between them is not caught
  by verification.
- Keep the secret out of the pre-hash string, out of the served payload, and
  out of version control.

## Failure modes

### Reported as an error code

| Condition | Result |
| --- | --- |
| `41000` | Missing security parameters: a field this service requires is absent from the packet |
| `41001` | Invalid parameter length: a key or secret altered in length by environment or config handling |
| `41002` | Not a valid consumer: the key is unrecognized — another environment's key, or a newly created set still propagating |
| `41003` | Signatures do not match: a signed input differs from what was sent, or the secret belongs to a different key |
| `41004` | Timestamp outside the accepted window relative to current UTC |
| `41005` | Invalid domain: the signed domain is not allowlisted, or does not match the origin the request arrived from |
| `41006` | `expires` is not a GMT `YYYYMMDD-HHMM` value |
| `41007` | The request arrived after its `expires` |
| `40002` / `40003` | Consumer or organization access, not signing: the signature verified and authorization failed |

The Data API returns these in the response body alongside an HTTP status;
browser-facing APIs log them to the JavaScript console.

### Reported weakly or not at all

| Condition | Result |
| --- | --- |
| Secret read with surrounding whitespace or quotes from an env file or shell | `41003` while every visible value is correct |
| Request re-encoded after signing by framework middleware | `41003` that survives every credential check, because the credentials are not the fault |
| One signature generated and reused across users or requests | works until the timestamp window closes, then fails for everyone at once |
| Allowlist or key change not yet propagated | the same configuration starts working later with no code change; an immediate retry looks like a hard failure |
| Secret serialized into the init object sent to the browser | no error at all — the integration works, and only inspecting what is served detects it |
| `user_id` carrying PII or exceeding 50 characters | verification is unaffected: this is a data constraint, so it surfaces elsewhere or not at all |

## Diagnostics

Confirm the cause before changing configuration — several of these report the
same `41003`.

1. **Correct on the development host, rejected once deployed** → the signed
   `domain`. Compare the `domain` in the served init object against the
   browser's own host, then confirm that host is allowlisted for the consumer.
2. **`41003` with credentials known to be correct** → compare the pre-hash
   string. Log it server-side at signing time and compare its request segment
   with the JSON the page actually received. A difference there is a
   serialization fault; no difference points back at the secret.
3. **`41002` immediately after generating a key set** → propagation. Re-test
   after 30 minutes before changing anything else.
4. **`41004` on a page that worked minutes earlier** → a cached or reused init
   object, or a clock that is not UTC. Compare the signed timestamp against
   `date -u +%Y%m%d-%H%M` on the signing host.
5. **Data API rejected while browser APIs pass on the same credentials** →
   packet shape and origin. The Data API packet takes no `user_id`, and a
   request sent from a browser must be signed with the browser's domain.
6. **Rejected for some learners only** → the `user_id` those sessions carry;
   an empty or oversized value is the input that varies between them.
7. **Everything verifies and the concern is exposure** → search the served
   payload for the secret. A leaked secret raises no error.

## Runtime checks

| Question | Check |
| --- | --- |
| Which error did the API actually raise? | `LearnosityItems.errors` in the browser console; the response body for a Data API call |
| Is the signature well formed and on the current version? | it begins `$02$` and is 68 characters long |
| Is the signed domain the browser's domain? | compare `security.domain` in the served init object with `window.location.hostname` |
| Is the signing clock UTC-accurate? | `date -u +%Y%m%d-%H%M` on the signing host against the signed `timestamp` |
| Did the request bytes change after signing? | hash the serialized request at signing time and again in the rendered response, then compare |
| Is the secret reaching the client? | search the served HTML and JSON for `consumer_secret` and for the secret's leading characters |
| Which fields does this service's packet require? | the SDK's `REFERENCE.md` for the language in use |

## Reference lookups

| Topic | `source` | `article_id` |
| --- | --- | --- |
| Signature components, pre-hash string construction, hash generation, `$02$` prefix | `help_site` | `360000754738` |
| Full security error table, including `41005`–`41007` and the 40xxx access codes | `help_site` | `16458061457565` |
| Items API error codes and the browser-side error list | `help_site` | `16458090530845` |
| Data API error codes mapped to HTTP response codes, and the browser-domain rule | `help_site` | `16458089770653` |
| Domain allowlisting, key sets per consumer, propagation delay | `help_site` | `360001767178` |
| `expires` semantics and default session validity | `help_site` | `360000755378` |
| `user_id` constraints | `help_site` | `16560472198045` |

## Further reading

| Topic | `source` | `article_id` |
| --- | --- | --- |
| `Init()` arguments and security defaults, PHP | `github` | `8814326499369822536` |
| `Init()` arguments and security defaults, Node.js | `github` | `10790999250645865205` |
| `Init()` arguments and security defaults, Python | `github` | `8178786445494155368` |
| `Init()` arguments and security defaults, Java | `github` | `10051180954462946798` |
| `Init()` arguments and security defaults, Ruby | `github` | `9144409672829329245` |
| `Init()` arguments and security defaults, ASP.NET | `github` | `16317273184219915862` |
| Signing a Data API request sent from the browser | `help_site` | `360000755758` |
| PII definition applying to `user_id` | `help_site` | `360002309578` |

## Related skills

- **install-and-set-up-the-sdk** — installing and configuring the SDK that
  signs.
- **content-migration** — signing `get` against a source bank and `set`
  against a destination with two separate credential sets.
- **golden-path-author-items-reports** — where the signed init object is
  consumed by Author, Items and Reports.
