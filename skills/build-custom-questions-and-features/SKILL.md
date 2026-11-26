---
name: build-custom-questions-and-features
description: Guide a developer from the custom-questions-skeleton repo to a custom Question or Feature that is authored in Author API and rendered in Items API — choosing between a Question and a Feature, sequencing the module, scorer, authoring schema, hosting and delivery steps, and catching the failures that report nothing.
---

# Build Custom Questions and Features

**Goal:** Take a developer from the `custom-questions-skeleton` repo to a custom Question or Feature that authors create
in Author API and learners see in Items API. The skill sets the order of the work, the decision between a Question and
a Feature, and the failure modes between those steps.

**APIs:** `author-api`, `items-api`, `questions-api` (the module contract and the skeleton's local `assessment.php`
harness are Questions API; production delivery goes through Items API, and authoring goes through Author API's
Question Editor dependency).

## When to use this skill

- **Building** a new custom Question type or custom Feature type from the skeleton.
- **Debugging** a custom type that does not render, does not save, scores locally but not on the server, or does not
  appear as an authoring tile.
- **Reviewing** a custom type before its asset URLs are saved into an item bank.
- **Publishing** a working local build to a host that Learnosity and learners can reach.

When a requirement falls outside what the articles, demos and skeleton below describe, say that no documented approach
was found, point at the closest documented option in the reference tables, and do not invent an API, event or init
option to fill the gap.

Building a custom type is a lower-level use case by nature. When `search_docs_sources` withholds Questions API or
Question Editor API hits, repeat the search with `off_golden_path_reason` set to the custom type being built.

## Preconditions

- Node.js and Yarn are on the path, at versions satisfying the `engines` field in the repo root `package.json`; a
  version below that range makes `yarn install` fail outright rather than warn. The custom type itself is JavaScript,
  CSS and HTML served as static files; none of it runs on the integrator's backend.
- The backend language is the integrator's choice. Its only role is signing the Author API and Items API requests in
  Steps 8 and 9, which any Learnosity SDK language, or a hand-rolled signature, can do. The PHP in the skeleton and
  demos is local harness scaffolding: port what it shows into the integration's own language instead of adding PHP.
- PHP CLI is needed only to run that harness. Each demo's `yarn dev` starts `php -S localhost:12345` alongside
  `webpack watch`, and the harness signs with the PHP SDK committed under `demos/vendor`. On recent PHP the committed
  SDK prints `Deprecated:` notices at the top of the harness page; they are harmless and not a build failure.
- The custom files are hosted by the integrator. Learnosity loads them from the URLs in the Question or Feature JSON on
  every render, and fetches the scorer for server-side scoring, so those URLs must be reachable from the internet.
- The target delivery is online. Custom Questions and Features are not available in offline app mode or Local Device
  Assessment.
- Signed Author API and Items API requests already work in the integration, or will be added with
  `generate-and-troubleshoot-request-signatures`. The skeleton's `config.php` signs with a shared demo consumer that
  is only for the local harness.

## Sequence

### Step 1: Choose a Question or a Feature

| Build a | When the component | Skeleton folder | JSON `type` | Id attribute | Scorer |
| --- | --- | --- | --- | --- | --- |
| Custom Question | captures a learner response that is saved, and optionally scored | `demos/custom-question-skeleton` | `custom` | `response_id` | optional |
| Custom Feature | is interactive or decorative but captures no response (a tool, a viewer, a passage) | `demos/custom-feature-skeleton` | `customfeature` | `feature_id` | none |

A Feature has no `changed`, `validate`, scoring or validation UI. If any learner input must reach a report or a score,
build a Question. Steps 4 and 5 mark the parts that apply to Questions only.

Produces: the skeleton folder used in every later step.

### Step 2: Start from the skeleton

Clone or fork `https://github.com/Learnosity/custom-questions-skeleton`, copy the chosen skeleton folder and rename it
after the custom type. Set a unique `name` in
the copy's `package.json` as well: the repo root declares `demos/*` as Yarn workspaces, and the `yarn install` that
starts both `yarn dev` and `yarn prod` fails with `There are more than one workspace with name` while two workspaces
share one. Run `yarn dev` from inside that folder, not from the repo root: the root `yarn dev` runs every demo through
`turbo`, and every demo binds port `12345`.

The local pages are `http://localhost:12345/assessment.php` (Questions API harness) and
`http://localhost:12345/authoring.php` (Author API in `item_edit` mode). Both read the JSON files in the folder, so
neither needs editing. The Question harness shows Save, then Resume and Review once the save succeeds; the Feature
harness hides Save and Resume and offers Review directly.

Consider running the harness in a container (an image with PHP CLI, Node.js and Yarn, the folder mounted, port `12345`
published) so the harness's PHP and Node versions stay off the host and are the same for every developer. The
skeleton ships no container definition. `yarn dev` binds `php -S` to `localhost`, which inside a container is the
container's own loopback: override the bind address to `0.0.0.0` in the container, and keep browsing
`http://localhost:12345` on the host so the pages load from the same origin as outside a container.

Produces: a running local harness that rebuilds `dist/` on save.

### Step 3: Define the JSON contract (`questions-api`)

Edit `question.json` or `feature.json`. This is the object Questions API receives and, for a Question, the object the
scorer receives on the server.

- Every value the module or scorer needs must be in this JSON at initialization. Attributes added to the object after
  initialization are ignored server-side.
- Reuse native attributes (`stimulus`, `validation`, `instant_feedback`, `metadata` and others) with their native
  names and shapes instead of inventing equivalents. Their handling by the platform differs per attribute; the
  Creating Custom Questions article lists which are managed and which the module must render.
- Use snake_case for custom attributes and avoid `type`, `custom_type`, `js`, `css` and `version`.
- Server-side scoring depends on a valid response being present in the initialization JSON. The Creating Custom
  Questions attribute table names a top-level `valid_response`; the skeleton's `question.json` carries it as
  `validation.valid_response`. Confirm which shape the build in use scores with the Data API scoring check (Runtime
  checks) before authoring content around either.

Produces: the attribute set that Steps 4, 5 and 6 read and expose.

### Step 4: Render the module and signal ready (`questions-api`)

Work in `src/question/index.js` or `src/feature/index.js`. The entry files (`src/question.js`, `src/scorer.js`,
`src/feature.js`) wrap the classes in `LearnosityAmd.define` and return `{ Question }`, `{ Scorer }` or `{ Feature }`;
keep that shape.

Both kinds:

- Render into `init.$el`, then trigger `ready`. Without `ready` within the allowed time, Questions API raises an
  error for the component.
- Render only the module's own attributes into `init.$el`. The platform renders `stimulus` and `instructor_stimulus`
  itself, outside the response area, so rendering `stimulus` in the module too produces a duplicate prompt. Confirm
  which attributes the surface in use renders with the attribute-rendering check (Runtime checks) before rendering any
  native attribute.
- Implement `facade.disable()`. The skeletons call it in `review` state.
- Prefix every CSS class (the skeleton's `PREFIX` constant) and style in `scss/`.
- For external libraries or services, retry a failed load up to three times with a short delay, then show a message
  in the component or stop the assessment, per the error-handling article.

Questions only:

- Trigger `changed` with the response value whenever the learner changes it. The value may be any serializable type;
  it becomes `response` in the scorer.
- Register the `validate` handler before triggering `ready`. It fires in every state, so it is the place to show
  validation UI and, when `options.showCorrectAnswers` is set, correct answers.
- In `resume` and `review` states, restore the UI from `init.response`. Review state is what Reports API's item-level
  reports display.
- Use the `lrnUtils` components and Learnosity's validation classes rather than rebuilding Check Answer, suggested
  answers or correct/incorrect styling; the Creating Custom Questions article and the skeleton README list them.

Produces: a component that renders, saves and reloads in all three states of the local harness.

### Step 5: Implement and verify the scorer (`questions-api`)

Questions only. Implement the scorer interface stubbed in `src/scorer/index.js`; the Creating Custom Questions article
defines each method. The scorer runs on Learnosity's servers as well as in the browser, so it must not touch the DOM or
import UI libraries.

- Omitting `js.scorer` disables autoscoring.
- Code defensively. Learnosity logs only uncaught server-side scorer exceptions; an error the scorer catches is
  invisible unless the integrator logs it to their own service. Do not write error details into the response value.
- Verify locally with `yarn debug-server-scorer` after editing the question and response in `debugServerScorer.js`. It
  loads `./dist/scorer.js`, so a build must exist.
- Verify server-side parity with the Data API scoring check before Step 8.

Produces: a scorer whose server-side result matches the browser's.

### Step 6: Define the authoring schema and layout (`author-api`)

Edit `question_editor_init_options.json` and `authoring_custom_layout.html`; check the result at `authoring.php`.

- Register the type in `custom_question_types` or `custom_feature_types`, with a `custom_type`, `editor_schema`,
  `editor_layout`, `js`, `css` and `version`.
- Add a `question_type_templates` entry whose key equals `custom_type`. It is the tile authors pick, and its
  `defaults` become the JSON of every new instance, including `js` and `css`.
- Define authorable attributes in `editor_schema.attributes`. Schema types, validation, conditional attributes and
  limits are in the authoring article; layout hooks and utility components are in the editor layout articles.
- Author API passes these options to Question Editor as `config.dependencies.question_editor_api.init_options`, which
  is exactly what `authoring.php` does.

Produces: `custom_type`, and a tile that creates a valid instance.

### Step 7: Build and host the assets

Run `yarn prod`. Host the contents of `dist/` (`question.js`, `scorer.js`, `question.css` or `feature.js`,
`feature.css`, plus `authoring_custom_layout.html`) on a server Learnosity and learners can reach, with CORS headers if
the host requires them.

Decide how the URLs are addressed before any content is authored, because Step 8 saves them into every instance:

| Option | Cost |
| --- | --- |
| Console Custom Questions proxy | One more Console object to manage; the hosted files can move by updating the proxy, without editing saved content |
| Direct URL to the host | Nothing to set up; moving or renaming the files later means updating every saved Question or Feature |

Replace every `/dist/...` path in `custom_question_types` / `custom_feature_types` and in the template `defaults`
with the chosen URLs. To test the production build locally first, point `question.json` or `feature.json` at the same
URLs and reload `assessment.php`.

Produces: `asset_urls`.

### Step 8: Author items in the integration (`author-api`)

In the integration's own server-side Author API request, in the backend's language, pass the Step 6 init options with
the Step 7 URLs under `config.dependencies.question_editor_api.init_options`, and sign the request per
`generate-and-troubleshoot-request-signatures`. The harness's `authoring.php` shows the request shape to port. Create an item from the custom tile, save it, and add it to an activity
if delivery uses activities.

Produces: `item_reference`, and an `activity_id` where used.

### Step 9: Deliver in Items API (`items-api`)

Initialize Items API with the item references or the activity, signed per
`generate-and-troubleshoot-request-signatures`. Items carrying custom types need no other Items API configuration;
they render, save and score through the same session flow as native types, and Reports and analytics read them
without extra work.

- Values the component needs from the host page at runtime, such as an auth token, go in
  `config.questions_api_init_options.custom_widget_options` and are read in the module with
  `init.getCustomWidgetOptions(key)`. Agree the key with whoever initializes Items API.

Produces: a learner session containing the custom Question or Feature.

## Shared identifiers

| Name | Produced by step | Consumed by step(s) | Description |
| --- | --- | --- | --- |
| `custom_type` | 6 | 8 | Unique id of the custom type; the `question_type_templates` key must equal it |
| `asset_urls` | 7 | 8, 9 | Externally reachable `js`, `css` and `editor_layout` URLs, saved into every authored instance |
| `item_reference` | 8 | 9 | An item saved with the custom Question or Feature, loaded by Items API directly or through an activity |

## Failure modes

### Local harness (Steps 2–5)

| Condition | Result |
| --- | --- |
| `yarn dev` run from the repo root | Every demo starts and contends for port `12345`; the page served may not be the one being edited |
| Harness run in a container with `php -S` still bound to `localhost` | The published port refuses connections from the host |
| PHP CLI missing | `webpack watch` still builds `dist/`, but the harness at `localhost:12345` serves nothing |
| `ready` never triggered, or triggered before an awaited render settles | Questions API raises a timeout error for the component |
| `validate` handler registered after `ready` | Outside the required order; Check Answer can produce no validation UI |
| `stimulus` rendered by the module | The platform also renders it, so the prompt appears twice; no error |
| `changed` not triggered | The UI updates but `getResponses()` holds nothing and the attempt is unscored |
| `init.response` not restored in `resume` / `review` | The saved response exists but the UI shows a blank component |
| Reload for `resume` / `review` not carrying the saved `session_id`, or deriving a different `response_id` from it | Questions API initializes with no `init.response`, so the component loads blank though the save succeeded |
| `user_id` placed in the request object instead of the security packet | `ValidationException` at init; the skeleton's `config.php` signs it into the packet, so a hand-rolled harness hits this where the skeleton does not |
| `yarn debug-server-scorer` before any build | Node cannot find `./dist/scorer.js` |
| `yarn debug-server-scorer` in a Feature project | The script is declared but `debugServerScorer.js` does not exist; Features have no scorer |

### Scoring (Steps 3, 5)

| Condition | Result |
| --- | --- |
| Valid response missing from the initialization JSON, or in a shape the build does not score | Browser scoring may work while server-side scoring is not triggered |
| Scorer reads an attribute added after initialization | Correct in the browser, wrong or `null` on the server |
| Scorer touches `window`, `document` or a UI library | Works in the browser, fails on the server |
| Scorer catches its own exception | No entry in Learnosity's server-side log; only the integrator's own logging records it |
| Scorer URL relative or on `localhost` | Not reachable from Learnosity's servers, so server-side scoring cannot run |

### Authoring and hosting (Steps 6–8)

| Condition | Result |
| --- | --- |
| Template key differs from `custom_type` | The tile is not associated with the custom type's schema and layout |
| Template `defaults` still hold `/dist/...` URLs | New instances are saved with paths that only resolve on the dev host |
| Custom attribute named `type`, `js`, `css`, `version` or `custom_type` | Collides with a reserved attribute |
| Assets moved without a Console proxy | Every saved instance still points at the old URLs and fails to load |
| Host rejects cross-origin requests | Assets fail to load in Author or Items API; visible only as network errors |

## Runtime checks

| Question | Check |
| --- | --- |
| Which content attributes does the surface render itself? | Render with the module rendering none of them, and inspect the DOM: a platform-rendered `stimulus` sits in `.lrn_stimulus` outside `.lrn_response`. A prompt that appears only when the module renders it is the module's to render |
| What response shape does the Question save? | `questionsApp.getResponses()` in the `assessment.php` console; in Items API, `itemsApp.getResponses()` |
| Does the browser scorer agree with the intended score? | `questionsApp.getScores()` locally, `itemsApp.getScores()` in delivery |
| Does the server-side scorer produce the same score? | Run the Data API scoring check from Creating Custom Questions with the Question JSON and saved response; `data[].score` and `data[].max_score` must be non-null and match the browser |
| Is the scorer loadable without a DOM? | `yarn debug-server-scorer`; the second `score:` line it prints is `maxScore()` |
| Is the component instance alive and exposing the facade? | `questionsApp.question('<response_id>')` or `questionsApp.feature('<feature_id>')`; `itemsApp.question(...)` / `itemsApp.feature(...)` in delivery |
| Did Questions API report an error for the component? | The `errorListener` output in the `assessment.php` console |
| Are the saved asset URLs the hosted ones? | Open the item's Question or Feature JSON in Author API's source view and inspect `js`, `css` |
| Are the assets reachable from outside the dev machine? | Request each `asset_urls` entry from a host other than the dev machine and check status and CORS headers |

## Reference lookups

| Topic | `source` | `article_id` |
| --- | --- | --- |
| Overview, file roles, hosting responsibility, the documented task order, offline caveat | `help_site` | `4414363148561` |
| Question JSON attributes, `init` and `lrnUtils`, events, states, scorer interface, Data API scoring check | `help_site` | `360000758817` |
| Feature module, `init` object, `ready` | `help_site` | `360000755458` |
| `editor_schema`, attribute types, limitations, `custom_type` and tile templates | `help_site` | `360000755098` |
| Retry, scorer logging and learner-facing error messages | `help_site` | `23833436280861` |
| Skeleton repo: demo list, scripts, publishing, Learnosity components and validation classes | `github` | `14454712979836724549` |
| Question skeleton key files and scripts | `github` | `16016163686799668705` |
| Feature skeleton key files and scripts | `github` | `17130591652527186733` |

## Further reading

| Topic | `source` | `article_id` |
| --- | --- | --- |
| `custom_question_types` properties | `help_site` | `16735211203997` |
| `custom_feature_types` properties | `help_site` | `16735211226141` |
| Custom Feature attributes | `help_site` | `16685276930717` |
| Custom Question attributes in the Question types reference | `help_site` | `16684586288925` |
| Editor layout hooks for a custom type | `help_site` | `360000755258` |
| Utility components (tabs, accordion) for editor layouts | `help_site` | `360000755118` |
| Author API `question_editor_api.init_options` | `help_site` | `16852477989405` |
| Items API `custom_widget_options` | `help_site` | `16560458946461` |
| Questions API `custom_widget_options` | `help_site` | `16760639142813` |
| Data API scoring endpoint | `help_site` | `18356466181917` |
| Console Custom Questions proxies | `help_site` | `360000756198` |
| CORS and domain allowlisting for externally hosted files | `help_site` | `360000757877` |
| Items API `getResponses` / `getScores` | `help_site` | `16461122550429`, `16461151401117` |
| Authoring demo for a custom type with a dynamic schema | `github` | `17292536847172029062` |
| Custom Feature authoring demo | `demos` | `128529369064418998` |
| Question type templates demo | `demos` | `5539524231046138876` |

## Related skills

- `generate-and-troubleshoot-request-signatures` — signing the Author API and Items API requests in Steps 8 and 9, and
  resolving `4100x` rejections.
- `golden-path-author-items-reports` — the Author, Items and Reports API flow that custom types slot into.
- `install-and-set-up-the-sdk` — installing the server-side SDK used for signing, in the integration's language.
- `customize-assessment-ui-regions` — the Items API player around the custom component.
