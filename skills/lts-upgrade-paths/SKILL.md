---
name: lts-upgrade-paths
description: Decide a target LTS version, sequence the upgrade across the core APIs, and find the breaking changes for every hop, so an integration moves off an ageing or unpinned version without silent regressions.
---

# Upgrade Between Learnosity LTS Versions

An LTS upgrade changes the version token the APIs load. The reference material
for it is spread across a lifecycle article, a version-dates list, a CSS-diff
tool, and one migration guide per release. This skill sits above them: it says
what order to do things in, which changes break silently, and how to confirm the
version actually in use.

**APIs:** `author-api`, `items-api`, `reports-api`, `data-api` (the version
token applies identically to every core API, including `questions-api` and
`assess-api`).

## When to use this skill

- **Migrating** an integration from one LTS version to a newer one.
- **Planning** a move off `developer` or `latest-lts` onto a pinned version
  before going to production.
- **Debugging** behavior or styling that changed after a version token changed.
- **Reviewing** an upgrade for the changes that report no error.

## Core principle

The version is a single token in the API load, not a package to install. Each
core API resolves the same token independently, so an upgrade is not atomic
unless every load is changed together.

| Token | Resolves to | Use for |
| --- | --- | --- |
| `vYYYY.X.LTS` | one fixed LTS release | production; the only safe choice there |
| `latest-lts` | whatever LTS is newest, moving | development and testing only |
| `developer` | newest biweekly build, unpinned | proof of concept only |

Consequences:

- A pinned integration does not move until the token changes. Upgrades are
  therefore deliberate, and skipping releases is normal — the breaking changes
  of every skipped release still apply, cumulatively.
- Backwards compatibility holds across releases, but a deprecated option is
  removed on a later LTS, announced in the migration guide for the release that
  deprecates it. Code relying on it keeps working until the removing release,
  then stops taking effect — often by reverting to a default rather than
  erroring — with no change on your side.
- `latest-lts` and `developer` move under a running integration. A regression
  that appears without a deploy points at an unpinned token.

## Preconditions

- The current version token is known for **every** API load, server and client
  side. An integration can load different tokens for different APIs.
- The target LTS version is chosen against the lifecycle, not the newest by
  default: confirm the target's dates and that the current version's End of Life
  has not already forced the hand (`360001268538`, `10501902647581`).
- Custom CSS or DOM-dependent automation is inventoried before starting, because
  UI changes are the most common source of silent breakage (see Failure modes).

## Sequencing the upgrade

Least invasive first.

1. **Locate every version token.** Every `<script>` load of a core API and every
   Data API endpoint URL. Missing one leaves a mixed-version integration.
2. **Enumerate the hops.** List each LTS release strictly between the current and
   target versions, inclusive of the target. Breaking changes are documented per
   release, so a multi-release jump requires reading every intervening guide, not
   only the target's.
3. **Read each hop's migration guide.** For each release in the list, read its
   migration guide for product changes, deprecations and removals; reach every
   release's guide from the version list (`10501902647581`).
4. **Diff the CSS/UI.** For a customized UI, compare the current and target
   versions with the CSS-diff tool before touching code (`6563040320029`).
5. **Change the token on a non-production version first.** Point loads at the
   target `vYYYY.X.LTS`. Do not validate against `latest-lts`: it will drift away
   from the version you intend to ship.
6. **Verify, then roll out together.** Run the Runtime checks below, then change
   all loads to the target token in one deploy so no API is left behind.

## Failure modes

Grouped by what changed. Every row is a change that returns no error.

### Deprecation removed on a later LTS

| Condition | Result |
| --- | --- |
| Deprecated config flag still passed after the LTS that removes it | Flag silently ignored; feature reverts to default |
| Sunset feature (e.g. `formulainput`) referenced after its sunset LTS | Feature absent; no load error |
| Renamed config flag, old name still passed before removal | Works, until the removal LTS turns it into the row above |

### Removed Question type or report

| Condition | Result |
| --- | --- |
| Activity references a Question type removed in a hop (e.g. `sortlist`) | Item fails to author or render; existing responses unaffected |
| Report configured with a type removed in a hop | Report does not render |

### UI / CSS change

| Condition | Result |
| --- | --- |
| Custom CSS targets a class whose prefix changed | Override no longer applies; layout regresses, no error |
| DOM automation asserts on a structure that changed (e.g. caption `<track>` attributes) | Automation fails; API behaves normally |
| Value read from the DOM changed (e.g. video `srcType` `youtube` → `x-youtube`) | Comparison silently fails |

### Mixed version

| Condition | Result |
| --- | --- |
| One API load left on the old token after the others move | APIs disagree at runtime; symptoms scattered and version-dependent |
| Non-prod validated on `latest-lts` | Passed against a version you will not ship |

## Diagnostics

Confirm the cause before changing configuration.

1. **Regression with no deploy** → token is `latest-lts` or `developer` and
   moved. Confirm the token in use (Runtime checks); pin it.
2. **A feature or option stopped taking effect after an upgrade** → it was
   deprecated and removed on one of the hops. Confirm by searching the migration
   guides for each hop for the option name, not only the target's guide.
3. **Styling regressed, no console error** → a CSS class or structure changed on
   a hop. Confirm against the CSS diff for the full range before editing
   overrides.
4. **Behavior differs across two APIs in the same integration** → mixed
   version. Confirm every token matches before treating it as a product bug.
5. **A change described in a guide does not match the running build** → treat as
   build-specific; confirm the token actually loaded and raise it to the user
   rather than coding against the discrepancy.

## Runtime checks

Question and check, against the running integration.

| Question | Check |
| --- | --- |
| Which version is each API actually loading? | Inspect every core-API `<script>` `src` and every Data API endpoint URL for the version token; they must all match the target |
| Is any load unpinned? | Search loads for `latest-lts` and `developer`; neither belongs in production |
| Is a deprecated option still in use? | Grep the integration's config for the old flag names listed in the hops' guides |
| Did a customized class change? | Open the CSS-diff comparison for the current..target range and search for the overridden class names |
| Is the Data API on the intended version? | Confirm the endpoint path segment is `vYYYY.X.LTS` (no `?`), not `latest-lts` |

## Reference lookups

| Topic | `source` | `article_id` |
| --- | --- | --- |
| Lifecycle, cadence, EOL table, and how to set the version token | `help_site` | `360001268538` |
| LTS version list with release, deprecation and EOL dates | `help_site` | `10501902647581` |
| CSS-diff tool for UI changes between two LTS versions | `help_site` | `6563040320029` |

## Further reading

| Topic | `source` | `article_id` |
| --- | --- | --- |
| What LTS releases are, and the version index | `help_site` | `360000758857` |
| Which APIs form the core set an upgrade must cover (Golden Path) | `help_site` | `360000754578` |
| Migration guide, example of the per-release format (v2025.3) | `help_site` | `30602837724061` |

## Related skills

- `golden-path-author-items-reports` — the core API set an upgrade must move
  together, and how they combine.
- `content-migration` — moving content between item banks; a bank-to-bank move
  is version-independent but often runs alongside an upgrade.
