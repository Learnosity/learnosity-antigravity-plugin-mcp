---
name: golden-path-author-items-reports
description: Guide a developer from zero to a working platform with content authoring, assessment delivery, and score reporting using Author API, Items API, and Reports API.
---

# Build a Complete Assessment Platform

**Goal:** Guide a developer from zero to a working platform with content authoring, assessment delivery, and score reporting using Author API, Items API, and Reports API.

**APIs:** `author-api`, `items-api`, `reports-api`

## Sequence

### Step 1: Set up content authoring (`author-api`)

Initialize Author API to embed the Learnosity authoring interface in your
application. This lets content creators build items (questions) and
organize them into activities (tests). The key output of this step is
one or more activity references that you will use in step 2 to deliver
assessments to learners.

### Step 2: Deliver an assessment to a learner (`items-api`)

Initialize Items API to render an assessment session for a learner.
You pass in the `activity_id` created in step 1, a `user_id` for the
learner, and a unique `session_id` for this attempt. Items API handles
all question rendering, response capture, and submission. The `session_id`
produced here is what you use in step 3 to retrieve results.

### Step 3: Display results and reporting (`reports-api`)

Initialize Reports API to show score breakdowns, session summaries,
or progress dashboards. You reference the same `session_id` and `user_id`
from step 2 to pull results for a specific attempt, or use `activity_id`
to aggregate across multiple learners.

## Shared identifiers

| Name | Produced by step | Consumed by step(s) | Description |
|---|---|---|---|
| `activity_id` | 1 | 2, 3 | A unique reference for an activity (test) created during authoring. |
| `session_id` | 2 | 3 | A unique identifier for one learner's attempt at an activity. Generated server-side before initializing Items API. |
| `user_id` | 2 | 3 | Your application's identifier for the learner. Must be consistent across Items and Reports so results can be correlated. |

## Signing

All three APIs require a server-side security object containing a
HMAC-SHA256 signature. You never expose your `consumer_secret` to the
browser. Use the official Learnosity SDK for your language to generate
the signature.

### SDKs

- **PHP** — `learnosity/learnosity-sdk-php`
- **Python** — `learnosity_sdk`
- **Node.js** — `learnosity-sdk-nodejs`
- **Java** — `learnosity-sdk-java`
- **C#/.NET** — `LearnositySDK`
- **Ruby** — `learnosity-sdk-ruby`

### Security fields per API

| API | Required security fields |
|---|---|
| `author-api` | `consumer_key`, `domain`, `timestamp`, `user_id` |
| `items-api` | `consumer_key`, `domain`, `timestamp`, `user_id` |
| `reports-api` | `consumer_key`, `domain`, `timestamp`, `user_id` |

### Signing flow

1. On your server, assemble the security object (`consumer_key`, `domain`, `timestamp`, `user_id`).
2. Pass the security object and your request object to the SDK's `init()` method along with your `consumer_secret`.
3. The SDK returns a signed init object (security + request + signature).
4. Send that signed object to the browser.
5. In the browser, call `LearnosityAuthor.init()` / `LearnosityItems.init()` / `LearnosityReports.init()` with the signed object.

## Init blueprints

### `author-api`

```yaml
security:
  consumer_key: YOUR_CONSUMER_KEY
  domain: your-domain.com
  timestamp: "20260610-1200"
  user_id: admin-user-1
  signature: GENERATED_BY_SDK
request:
  mode: item_list
  config:
    global:
      save: true
  user:
    id: admin-user-1
    firstname: Admin
    lastname: User
```

### `items-api`

```yaml
security:
  consumer_key: YOUR_CONSUMER_KEY
  domain: your-domain.com
  timestamp: "20260610-1200"
  user_id: learner-user-1
  signature: GENERATED_BY_SDK
request:
  activity_id: activity_created_in_step_1
  session_id: unique-session-uuid
  user_id: learner-user-1
  rendering_type: assess
  type: submit_practice
  name: My Assessment
```

### `reports-api`

```yaml
security:
  consumer_key: YOUR_CONSUMER_KEY
  domain: your-domain.com
  timestamp: "20260610-1200"
  user_id: learner-user-1
  signature: GENERATED_BY_SDK
request:
  reports:
    - id: session-detail
      type: session-detail-by-item
      user_id: learner-user-1
      session_id: unique-session-uuid
```
