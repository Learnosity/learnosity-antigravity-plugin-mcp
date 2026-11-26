---
name: multi-tenancy
description: Guide a developer building a platform for their own customers to implement, review or repair tenant segregation on one shared Learnosity account — establishing which boundary is enforced by Learnosity and which is enforced by their own code on each of the two axes, content and response data, which failures surface as another tenant's data rather than as an error, and when a tenant that needs an enforced boundary belongs with the Learnosity account manager.
---

# Segregate Tenants on a Multi-Tenant Learnosity Platform

**Goal:** Implement, review or repair tenant segregation on one shared Learnosity account, establishing which boundaries
Learnosity enforces and which ones the integrator's own code must enforce.

**APIs:** `author-api`, `items-api`, `reports-api`, `data-api`

## When to use this skill

Whenever one Learnosity account serves several distinct groups — customers, districts, schools, publishers, resellers,
departments, product tiers — and one group's content or results must stay away from another's.

- **Designing.** Planning tenant segregation before building.
- **Reviewing.** Segregation is in place and its boundaries need to be located: which axis is enforced by Learnosity,
  which by the integration.
- **Fixing.** Content visible to the wrong tenant, content missing from its own author's list, reports returning another
  tenant's sessions, Data API reads hitting the wrong bank, tag search timing out, Item pool creation failing.
- **Migrating.** Adding tenant stamping to an integration that already has sessions, including backfilling the sessions
  delivered before it.

For a review, establish the boundary on each axis below, then work the failure modes. A developer who believes tags are
enforcing separation they are not is the case this skill exists to catch.

## Core principle: tags filter, they do not fence

Segregation happens on two independent axes. Each has a boundary Learnosity enforces and a finer one only the
integration enforces.

| Axis | Lives in | Addressed by | Enforced by |
| --- | --- | --- | --- |
| Content (Items, Activities, Questions, tags) | Item bank | `organisation_id` | Learnosity |
| Content subset within one bank | Item bank | Item bank tags + Author API restricted filters | **integration code** |
| Learner responses and sessions | API consumer's data silo | consumer credentials | Learnosity |
| Session subset within one silo | API consumer's data silo | `activity_id`, or session tags | **integration code** — no boundary is enforced inside a consumer |

**There are two unrelated tag spaces.** *Item bank tags* sit on Items and Activities and segregate content. *Session
tags* sit on a session as metadata and are the only tags that can select sessions. Conflating them produces designs that
appear to segregate results and do not.

Three consequences drive every decision below:

- **Item bank tags cannot segregate response data.** The tag-bearing reports that read them categorize results *within*
  sessions already selected by `user_id`, `session_ids` or `activities`. They aggregate; they do not select.
- **Session tags select sessions in one place only.** The Learning Outcomes Report can filter on `session_tags`, and
  only once Learnosity has provisioned a live dataset configured for those tags (`360001268518`). The Data API session
  endpoints do not take a session-tag filter: an unsupported filter parameter is ignored rather than rejected, so the
  request succeeds and returns every tenant's sessions. `activity_id` is the stamp that makes per-tenant selection
  possible; which parameter carries it is decided by the endpoint or report type.
- **A wrong filter returns another tenant's data instead of an error.** The Data API sees the whole Item bank regardless
  of tags, and a shared consumer's silo holds every tenant's sessions.

Init options are tamper-evident: the signature covers the JSON of the whole `request` object, so `request.config` and
`request.organisation_id` cannot be edited in the browser. That protects the integration only if the **server** derives
the tenant from the authenticated session. Signing a tenant id that arrived from the client signs the attacker's choice.

## The segregation model

One shared account serves every tenant: content is segregated by Item bank tags plus Author API restricted filters, and
response data by a tenant-namespaced `activity_id` plus session tags. Do not design a consumer or Item bank per tenant,
or per group of tenants; Learnosity does not provision them for that purpose.

- **Pros:** onboarding a tenant is minting a tag value — instant and unattended. One credential pair. Shared content is
  genuinely shared. Tag filters combine `all`/`either`/`none` with `created_by`, `current_user` and `status`. Item pools
  can be built per tenant from tag combinations within the bank.
- **Cons:** the boundary is the integration's authorization code and **its failure mode is silent cross-tenant
  disclosure, not an error**. One bank is one blast radius: a bad bulk Data API write reaches every tenant. Tag
  cardinality has a practical ceiling. Every failure mode in the tag section below applies.

State the failure mode explicitly when recommending this model: the integrator is accepting a correctness obligation.

### Tenants that need an enforced boundary

Competitors, regulated data, data residency or contractual separation can require a boundary Learnosity enforces rather
than one the integration enforces. That is an account-level arrangement, not an integration design: refer the customer
to their Learnosity account manager, and do not propose extra consumers or Item banks as the solution.

## Implementing the model

### Content: restrict what the author's list returns

Put a tenant tag on every Item and Activity and filter the Author API lists server-side with
`config.item_list.filter.restricted.tags` and its `activity_list` counterpart (`360000755218`, `16852548235805`). Four
operational rules decide whether that boundary holds:

- **`restricted` filters control read access; `save.restricted_tags` controls write access.** They are different options
  with similar names, and using the save option alone leaves every tenant's content listed.
- **A deny-list alone does not filter.** `none` applies only alongside `all` or `either`; on its own it is ignored and
  the list returns everything.
- **Tags must be applied at creation time** via `setItemTags()` / `setActivityTags()`, or the author's own new content
  will not match the filter and will disappear from their list.
- **`hide_tags` is cosmetic.** It keeps the tenant tag type out of the authoring UI; the `restricted` filter is what
  limits access.
- **The filter does not travel to delivery.** `restricted` filters scope the authoring lists. Items API delivers
  whatever `items`, `activity_template_id` or `item_pool_id` the signed request names, with no tag check at any level,
  so an Item the author list shows to one tenant is deliverable into any session — including as a nested Item of an
  Activity or pool. Resolve the content reference server-side against the authenticated tenant before signing, and make
  tenant consistency an invariant of how Activities and pools are assembled rather than something re-derived on every
  delivery.

These are Author API initialization options and belong in the init object the platform's server signs: that request is
what carries the tenant filter, and it is the only copy of it an embedded Author API honors. Console permission groups
apply the same option set per Item bank to **Author Site** users (`360000585238`), which covers tenant content admins
working directly in the Learnosity Author Site; they do not filter an embedded Author API that the platform signs
itself. Use both where both audiences exist, and derive the tenant for the signed init server-side either way.

### Reporting: stamp the tenant at delivery time

Do both of the following when initializing the session. Neither can be added afterwards without extra work, so this is
what makes per-tenant reporting possible later.

`activity_id` is a reporting-level grouping string the platform defines; it need not match an Item bank Activity
reference, so prefix it with the tenant. `session_metadata.session_tags` stamps the session with a tenant tag.

```yaml
# Items API, at delivery
request:
  activity_id: acme_corp::weekly_math_quiz   # reporting id, namespaced
  activity_template_id: weekly_math_quiz     # content source, from the bank
  session_id: unique-session-uuid
  user_id: learner-uuid
  session_metadata:
    session_tags:
      - type: tenant
        name: acme_corp
```

`activity_id` names the session for reporting; it loads no content. The content comes from `activity_template_id`,
`items` or `item_pool_id`, and a request carrying only a namespaced `activity_id` fails to initialize. Namespace the
reporting id and leave the content reference as the bank knows it.

Then pass only that tenant's identifiers when reporting, through whichever selector the chosen report type accepts —
`activities`, `users`, `session_ids`, or `session_tags` on a Learning Outcomes Report with a live dataset;
`16458078554653` carries the schemas. Two properties make that a tenant boundary: every value is derived server-side
from the authenticated session rather than accepted from the client, and the identifiers are unique per tenant. A
`user_id` reused for one person across two tenants returns both tenants' sessions from a shared silo, however carefully
it was derived.

Sessions delivered before tenant stamping was introduced carry neither marker. Their session tags can be backfilled for
Learning Outcomes Reports with a Data API `set` on `/jobs/sessions/metadata` (`26076304349213`), which replaces a
session's metadata rather than merging it: read the current metadata, merge locally, then write it back. The original
`activity_id` remains unchanged, so other report types need another tenant-safe selector for those sessions.

## Failure modes

### Tag-based segregation

| Condition | Result |
| --- | --- |
| `save.restricted_tags` used as a read control | It gates writes only. Read access needs `item_list.filter.restricted.tags`. This is the most common error in this area. |
| A deny-list-only filter | `none` is ignored unless `all` or `either` is also set, so the filter fails open and lists every tenant's content. |
| `allow_filtered_tags_overwrite: true` | Permits authors to save content that does not match the tenant filter, reopening the boundary. Often enabled to resolve a UX complaint. |
| New content missing from its author's own list | The tenant tag was not applied on create via `setItemTags()` / `setActivityTags()`. |
| `.` or `-` in tenant tag names | Special characters are indexed like spaces, so tag suggestion queries split and can time out on a bank with many tags (`4411124534161`). Prefer underscores or plain alphanumerics. |
| The Data API expected to respect tags | It reads the whole bank. Tags are a query filter, never an access control, so every server-side read is only as correct as the code issuing it. |
| One bank as one blast radius | A bulk Data API write is not scoped by tags and reaches every tenant. Rehearse bulk writes against a filtered read first. |
| A content reference taken from the client at delivery | Items API applies no tag filter, so any reference in the bank renders whatever tenant it belongs to, nested Items included. Validate `items` / `activity_template_id` / `item_pool_id` against the authenticated tenant before signing. |
| `created_by` used to scope a tenant by listing its authors | The array caps at 50 `user.id` values, so the approach fails once a tenant has more authors than that. Scope tenants by tag and reserve user filters for per-author views. |
| `current_user` and `created_by` set together | `current_user` overrides `created_by`, which is then ignored, leaving the author only their own content. Set one. |
| A user filter mistaken for the tenant boundary | `created_by` and `current_user` scope to authorship *within* a tenant, not to the tenant. Either one hides a colleague's content in the same tenant, so shared authoring breaks the moment two people collaborate. |

### Multiple Item banks on one account

An account can hold more than one Item bank for reasons other than tenancy — licensed or partner content, or a shared
read-only source.

| Condition | Result |
| --- | --- |
| `organisation_id` omitted on Data API calls | The call targets the key's primary bank, so it can read or write the wrong bank. |
| `organisation_id` mismatch between delivery and reporting | `sessions-summary-by-tag` needs the same `organisation_id` used at session init, and the consumer needs access to that bank. |
| Implicit and explicit Item sources mixed | Once one Item in an Items API request carries explicit source attributes, all of them must. Partial conversion fails. |
| An Item-level `organisation_id` under an Activity-level `item_pool_id` | The pool is canceled for that Item, which is then sourced from the live bank instead of the snapshot. |
| An Item pool spanning banks | A pool is a snapshot of one bank, so creation fails for an Activity whose Items span banks. |
| A bank reachable by several customer accounts | Licensed content, content partners and third-party access expose the bank to every Author Site user in all of them. 2FA can be enforced at Item bank level. |

### Response data

| Condition | Result |
| --- | --- |
| Item bank tags assumed to segregate results | They do not. Response data separates by consumer; session subsets separate by `activity_id` or session tags. |
| `session_tags` relied on as a general selector | Only the Learning Outcomes Report filters on them, and only with a provisioned live dataset. An unsupported `session_tags` filter on a Data API sessions endpoint is ignored, and the response contains every tenant's sessions. |
| Session metadata backfilled with `set` | The write replaces a session's metadata instead of merging, so other custom fields are lost. Read, merge, then write. |
| Tenant stamping added late | Sessions delivered before `activity_id` namespacing or session tagging carry neither and are invisible to per-tenant reports until backfilled. |

### Tenant identity and taxonomy

| Condition | Result |
| --- | --- |
| Tag hierarchies per tenant | The Data API hierarchies endpoint supports `get` only; hierarchies are built manually in the Author Site, so they do not scale per tenant. |
| A tenant identifier signed from client input | The signature makes init options tamper-proof in the browser, which is worthless if the signed value was the client's choice. Derive the tenant server-side. |

## Diagnostics

Work symptom to cause, and confirm the cause before changing configuration.

1. **Tenant A sees tenant B's Items.** Check whether the restriction is on `item_edit.item.save.restricted_tags` rather
   than `item_list.filter.restricted.tags`. Then check whether `none` is the only tag filter set. Then check
   `allow_filtered_tags_overwrite`. Then confirm the tenant tag is derived server-side and not read from a
   client-supplied field before signing.
2. **An author cannot see content they just created.** The tenant tag was not applied on create. Call `setItemTags()` /
   `setActivityTags()` in the save flow and confirm the new Item carries the tag the filter requires.
3. **A learner receives another tenant's content.** Inspect the content reference the server put in the signed Items API
   request. The Author API restricted filter has no effect on delivery, so this is an authorization gap in the code that
   assembles the request, not a tagging or filter problem.
4. **Tenant A's report shows tenant B's sessions.** Item bank tags neither cause nor fix this. Inspect the `activities`
   / `users` / `session_ids` arrays actually sent: one of them names another tenant. Confirm the `activity_id` prefix is
   applied at Items API init, not only at report time. If the report is not the Learning Outcomes Report, `session_tags`
   is not filtering anything.
5. **Tag suggestions time out while authoring.** The bank has many tags and the search term contains `.` or `-`. As an
   immediate workaround, scope the search with a colon: `term:` searches tag types, `:term` tag names, `type:name` both.
   Longer term, rename tenant tags to replace `.` and `-` with underscores.
6. **Data API reads or writes the wrong bank.** `organisation_id` was omitted, so the call hit the key's primary bank.
   Pass it explicitly on every `get` and `set`.
7. **An assessment fails to load Items after adding a second bank.** Check that every Item in the request carries
   explicit source attributes, then whether an Item-level `organisation_id` has canceled an Activity-level
   `item_pool_id`.
8. **Item pool creation fails.** The pool spans Item banks. The job reports which Items are not from the bank being
   snapshotted; move those Items or pool per bank.
9. **A review with no reported symptom.** Both tenant boundaries rest on integration code: the restricted tag filter for
   content, `activity_id` selection for responses. Confirm the tenant identifier is derived server-side on every signed
   init, then work each row of the runtime checks below, because these failures are silent.
10. **A tenant asks for separation the shared account cannot enforce.** Refer the customer to their Learnosity account
    manager rather than designing a boundary in the integration.

## Runtime checks

Each of these answers a segregation question directly, against the account in question. Prefer them to reasoning about
what the configuration should do.

| Question | Check |
| --- | --- |
| Is the tenant tag actually on the session? | Data API `get` on `/sessions/metadata` with `session_id: [id]` |
| Does the tenant's `activity_id` select only that tenant? | Data API `get` on `/sessions/metadata` (or `/sessions/statuses`) with `activity_id: ["tenant::activity"]`, then confirm no other tenant's session ids are returned |
| Is a filter parameter being honoured or ignored? | Re-run the same call with a deliberately impossible filter value. Identical record counts mean the parameter is not filtering |
| Does the author's restricted list hold? | Load Author API in `item_list` mode with the tenant's config and compare the rows with that tenant's known references, or with a second tenant's filtered config — an empty bank and a broken filter look alike |
| Is `none` contributing anything? | Compare the same config with and without the `all` / `either` entry alongside it |
| Did a metadata write preserve other fields? | Read `/sessions/metadata` before and after the `set` and compare the whole metadata object |

## Reference lookups

Call `read_document` with the `source` and `article_id` below. Fetch only what the current question requires.

| Topic | `source` | `article_id` |
| --- | --- | --- |
| Restricting access by tags, users and status in Item/Activity lists | `help_site` | `360000755218` |
| Console permission groups per Item bank | `author_guide` | `360000585238` |
| Multiple Item bank sources in Author API | `help_site` | `360000754938` |
| Multiple Item bank sources in Items API (per-Item `organisation_id`) | `help_site` | `360000755538` |
| Consumers, data silos, sessions and `activity_id` | `help_site` | `360000754638` |
| `session_metadata` and session tags at delivery | `help_site` | `16560427705757` |
| Live datasets, the precondition for filtering a Learning Outcomes Report on `session_tags` | `help_site` | `360001268518` |
| Learning Outcomes Report | `help_site` | `20869477363229` |
| Item and tag search behavior, including tag search caveats | `help_site` | `4411124534161` |

## Further reading

| Topic | `source` | `article_id` |
| --- | --- | --- |
| `config.item_list.filter.restricted.tags` reference | `help_site` | `16852548235805` |
| `config.activity_list.filter.restricted.tags` reference | `help_site` | `16852463872285` |
| `config.item_edit.item.save.restricted_tags` reference | `help_site` | `21176171012381` |
| `config.activity_edit.save.restricted_tags` reference | `help_site` | `21176067529245` |
| Controlling tag visibility (`show_tags` / `hide_tags`) | `help_site` | `360001939978` |
| Tag formats (TagsV1 / TagsV2) | `help_site` | `360000758597` |
| Item pools as per-tenant content snapshots | `help_site` | `360000755878` |
| Signature construction and what it covers | `help_site` | `360000754738` |
| Sessions Summary with Tags report (`organisation_id` requirement) | `help_site` | `20869477751709` |
| Data API session metadata endpoint, for backfilling session tags | `help_site` | `26076304349213` |
| Learning outcomes reporting, session selection explained | `help_site` | `360002023438` |
| Report types index and array limits | `help_site` | `16458078554653` |
| Tag hierarchies endpoint (`get` actions only) | `help_site` | `26076373806749` |
| How tag hierarchies work | `author_guide` | `360000434878` |
| Activity search behavior | `help_site` | `37486480353821` |
| Console overview, including consumer and Item bank access config | `help_site` | `360000756198` |
| Enforcing 2FA per Item bank, for banks shared across customer accounts | `author_guide` | `360005831238` |

## Related skills

- `content-migration` — moving content between banks, including the `organisation_id` targeting rules and tag
  pre-creation.
- `golden-path-author-items-reports` — the base Author, Items and Reports flow these segregation options configure.
