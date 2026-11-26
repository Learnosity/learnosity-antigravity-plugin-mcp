---
name: content-migration
description: Guide a developer migrating content into a Learnosity item bank via the Data API — covering bank-to-bank copies, importing from another platform via QTI, the required save order, and the SET ITEMS step that makes imported questions visible in Authoring.
---
# Migrate Content Into a Learnosity Item Bank
**Goal:** Guide a developer migrating content into a Learnosity
item bank via the Data API — covering bank-to-bank copies, importing from
another platform via QTI, the required save order, and the SET ITEMS step
that makes imported questions visible in Authoring.
**APIs:** `data-api`

## When to use this skill
Use this when a developer wants to:
- Copy content from one Learnosity item bank to another (e.g. a new
  consumer, a staging-to-production move, or splitting a bank).
- Import existing content from another platform (a data dump) into a
  Learnosity item bank.
- Understand why questions they imported via JSON are not showing up in the
  Authoring interface (answer: they have not been through SET ITEMS — see
  Step 5).

## Two migration paths
This skill covers two starting points, which differ in whether a Learnosity
**source** bank exists:

- **Bank-to-bank copy** (existing Learnosity content): you read from a source
  bank with a `get` and write to a destination bank with a `set`. This path
  uses **both** `source_credentials` (for every `get`) and
  `destination_credentials` (for every `set`).
- **Import from another platform** (new customer, e.g. a QTI export): there is
  **no Learnosity source bank**, so there are **no `source_credentials` and no
  `get` calls**. You convert the source content to Learnosity JSON off-platform
  (Step 3) and only `set` it into the destination with
  `destination_credentials`.

## Key idea: the Data API is get/set per data type
The Data API is **not** a REST API. Every request is a signed **HTTP POST**;
the operation is selected by the `action` request parameter — `get` to read,
`set` to write — not by the HTTP method. Each item bank data type has one
endpoint that serves both actions. A bank-to-bank migration is: `get` from
the source, `set` into the destination, one data type at a time, in dependency
order. An import migration skips the `get` action and only sets. Use the
official SDK, which sends the POST and applies the signature for you.

## Sequence

### Step 1: Set up credentials (`data-api`)
Gather the `consumer_key` and `consumer_secret` for the **destination** item
bank — you always need these to `set` content. If you are copying from an
existing Learnosity bank (bank-to-bank), also gather the **source** bank's
credentials for the `get` calls; if you are importing from another platform,
there is no source bank and you only need the destination credentials. Every
Data API call is signed server-side with the credentials of the bank it
targets. Never send a `consumer_secret` to the browser or across the network —
see Signing below.

### Step 1a: Target the right item bank when a key has multiple (`data-api`)
A single consumer key can have access to more than one item bank. If you do
**not** say which one, the call goes to that key's **primary** item bank — so a
migration can silently write to (or read from) the wrong bank.

Pass `organisation_id` (the target item bank's id) in the **request packet** —
not the security packet — to target a specific bank. It applies to every `get`
and `set` in this skill (tags, questions, features, items, activities, assets),
on both the source and the destination side.

- Omit `organisation_id` → the call targets the key's **primary** item bank.
- Set `organisation_id` → the call targets that specific bank. The consumer key
  must have access to the bank, or the call is rejected.

When the source and destination are different keys, set `organisation_id` per
side using each key's target bank id.

### Step 2 (optional): Pre-create tags and tag hierarchies (`data-api`)
Tags can be created on the fly when saving items and activities, but
pre-creating them first is the clearest approach and avoids surprises.
- **read:** `action: get` on `/itembank/tagging/tags`
- **write:** `action: set` on `/itembank/tagging/tags`

Make sure all tag types and tag names exist in the destination before you
save items or activities.

**Tag hierarchies have no create action.** `action: get` on
`/itembank/tagging/hierarchies` reads them, but there is no `set` action —
hierarchies must be recreated manually in the Author Site by a user with the
**Tag Hierarchy Manager** role. Flag this to the developer as a manual step;
it cannot be automated through the Data API.

### Step 3 (new-customer path only): Convert source content from QTI
If the developer is importing from another platform rather than copying
between Learnosity banks, convert the source QTI into Learnosity JSON first,
then feed that JSON into Steps 4–5.

- Use the maintained library **`@learnosity/lib-qti-converter`** (install:
  `npm install @learnosity/lib-qti-converter`; requires Node >= 22). It
  converts QTI ↔ Learnosity JSON and ships both a CLI and an importable
  library:
  - **CLI:** `lcon qti-to-learnosity --in item.xml --out item.json --pretty`
  - **Library:** `convertQtiToLearnosity(qtiXmlString, { qtiVersion: '2.2' })`
    returns `{ output, diagnostics }`; check `diagnostics` for unmapped or
    partially-supported constructs.
- Do **not** use the older `learnosity-qti` package, and do **not** use the
  Data API for QTI conversion — the `/itembank/conversion/fromqti` and
  `/itembank/conversion/toqti` endpoints have been removed and will return an
  "endpoint removed" error.

When hand-building or converting question JSON, each question type has a set
of **required properties**. Missing required properties is a common cause of
import failures. Validate the converted JSON against the target question
type's schema before saving. If unsure which properties are mandatory for a
question type, look it up in the Learnosity Questions API reference for that
type rather than guessing.

### Step 4: Migrate questions and features (`data-api`)
Save questions and features into the destination **before** the items that
use them — an item that references a question/feature that does not yet
exist will fail.
- **Questions** — `action: get` / `action: set` on `/itembank/questions`
- **Features** — `action: get` / `action: set` on `/itembank/features`

You can drive these by a list of question/feature references, or by passing
item references to the get endpoint to retrieve the questions/features
belonging to those items. The references you save here (`question_references`)
are what the items in Step 5 point at.

### Step 5: Migrate items, then activities — including SET ITEMS (`data-api`)
Save in this order: **items first, then activities** (an activity can only
contain items that already exist).
- **Items** — `action: get` / `action: set` on `/itembank/items` (on `get`,
  optionally use the `include` object — `include.items:
  ["dynamic_content_data"]` — to return dynamic content data)
- **Activities** — `action: get` / `action: set` on `/itembank/activities`

> **SET ITEMS is what makes content visible in Authoring.**
> Importing question JSON alone is **not** enough. A question only appears in
> the Authoring interface once it has been set into an item via **SET ITEMS**
> (`action: set` on `/itembank/items`). If a developer reports that imported questions are
> missing from Authoring, the cause is almost always that the questions were
> saved but never set into an item. Always include this step.

There is also a combined convenience endpoint, `/itembank/itemsandwidgets`
(`action: set`), which saves questions/features and items together in a single
request while preserving the correct internal ordering — useful when migrating
self-contained items.

You may limit what you migrate by **status**: `published`, `unpublished`,
`archived` (e.g. copy only `published` content, or exclude `archived`).

### Step 6: Handle assets (`data-api`)
- **Assets** (images, audio, video) referenced by questions/items/features
  are **not** copied inside the migrated JSON. Two options:
  - Absolute asset URLs keep working across organizations, so you may leave
    them as-is.
  - To move the files themselves, request upload URLs via `action: get` on
    `/itembank/upload/assets` and bulk-upload the assets.

## Batching
Migrate in batches (around 50 records at a time) and repeat until complete.
The SDK's Data API helper exposes a recursive request that follows the
`next` token in each response's `meta` object automatically, so you do not
have to page manually:
- single call: `request(endpoint, security, secret, requestPacket, action)`
- paginated: `requestRecursive(endpoint, security, secret, requestPacket,
  action)` — loops on `meta.next` until all matching data is returned.

## Signing
Every Data API request must carry a server-side HMAC-SHA256 signature. You
never expose your `consumer_secret` to the browser. Use the official
Learnosity SDK for your language — it builds the signature for you, so you do
not hand-roll the hashing.

For the **Data API**, the security packet needs only:

| Field | Notes |
|---|---|
| `consumer_key` | The key of the bank this call targets (source for get, destination for set). |
| `domain` | The domain the request is sent from. |

Unlike the browser-facing APIs (Items, Author, Reports), the Data API
security packet does **not** require a `user_id`. Pass the security packet,
your request packet, the `consumer_secret`, and the `action` (`get` or `set`)
to the SDK's Data API helper; it signs and sends the request. For the full
mechanics of signature construction, see
[Security and Authentication](https://help.learnosity.com/hc/en-us/articles/360000754738-Security-and-Authentication).
An official SDK is available for PHP, Python, Node.js, Java, C#/.NET, and Ruby.

## Shared identifiers
| Name | Produced by step | Consumed by step(s) | Description |
|---|---|---|---|
| `source_credentials` | 1 | 2, 4, 5, 6 | consumer_key + consumer_secret for the bank you copy FROM; signs every `get` action (including fetching assets in Step 6). Bank-to-bank path only (absent for QTI/other-platform imports). |
| `destination_credentials` | 1 | 2, 4, 5, 6 | consumer_key + consumer_secret for the bank you copy TO; signs every `set` action and the asset upload URL request. |
| `question_references` | 4 | 5 | References of saved questions/features; items must reference existing ones. |
| `item_references` | 5 | 5 | References of items created by SET ITEMS; activities contain them and only referenced items appear in Authoring. |

## Init blueprints
### `data-api` — get items from the source bank
```yaml
endpoint: https://data.learnosity.com/latest/itembank/items
action: get
security:
  consumer_key: SOURCE_CONSUMER_KEY
  domain: your-domain.com
  # signature generated by the SDK from SOURCE_CONSUMER_SECRET
request:
  # Optional. Targets a specific source bank when the key has several;
  # omit to use the key's primary item bank.
  organisation_id: SOURCE_ITEMBANK_ID
  limit: 50
  status:
    - published
  # Optional. `include.items` lists extra Item properties to return; its only
  # valid key is `items`. Use `dynamic_content_data` to pull dynamic content.
  # Omit `include` entirely if you don't need these extra properties.
  include:
    items:
      - dynamic_content_data
```

### `data-api` — set items into the destination bank (SET ITEMS)
```yaml
endpoint: https://data.learnosity.com/latest/itembank/items
action: set
security:
  consumer_key: DESTINATION_CONSUMER_KEY
  domain: your-domain.com
  # signature generated by the SDK from DESTINATION_CONSUMER_SECRET
request:
  # Optional. Targets a specific destination bank when the key has several;
  # omit to use the key's primary item bank.
  organisation_id: DESTINATION_ITEMBANK_ID
  items:
    - reference: item_reference_from_source
      status: published
      questions:
        - reference: question_reference_saved_in_step_4
      # questions/features referenced here must already be saved (Step 4)
```
