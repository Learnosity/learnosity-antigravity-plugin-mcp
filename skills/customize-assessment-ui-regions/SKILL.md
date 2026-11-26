---
name: customize-assessment-ui-regions
description: "Guide a developer to design, review, debug or repair an Items API UI Regions configuration — choosing between layout presets, region_overrides and a full regions object, and handling what a catalog cannot state: the order in which the configuration layers are applied, regions inside sections, which behaviors depend on the build in use, and the failure modes that report nothing."
---

# Customize the Assessment Player UI

**Goal:** Design, review, debug or repair an Items API UI Regions
configuration, establishing which configuration layer takes precedence, how
the responsive engine transforms the result, and which errors are reported
silently.

**APIs:** `items-api`

## When to use this skill

For any question about the assessment player's layout or controls: relocating
the table of contents, showing or hiding buttons, building a custom player
shell, or invoking a dialog from application code.

- **Building.** Selecting the least invasive configuration that produces the
  required layout.
- **Debugging.** An element is absent, in an unexpected region, present only
  at some widths or in some sections, or an event does not fire. Most failures
  in this area are silent; use the diagnostics below, symptom to cause.
- **Reviewing.** A configuration exists and its rendered result needs to be
  established, which frequently differs from what the JSON appears to specify.

The region, element, button and dialog catalogs are in the reference
articles listed at the end; read them there for lookups. This skill covers the
operational layer above them: how the parts combine, what varies by build and
activity, and how to confirm it.

## Core principle: four layers produce the layout

The rendered layout is not the object submitted. It is the result of applying,
in order:

| Order | Layer | Effect |
| --- | --- | --- |
| 1 | `config.regions` — preset string or object | The base layout. |
| 2 | `config.region_overrides` | Edits the base: adds, removes or re-options individual elements, or replaces a whole region. |
| 3 | A section's `config.regions` / `config.region_overrides` | `regions` replaces the global result for that section; `region_overrides` is applied on top of it. |
| 4 | The responsive engine, at render time | Transforms the result for the container width. |

Three consequences follow:

- **Layers 1–3 specify the large breakpoint only.** Any narrower rendering is
  the output of layer 4, so an element that is absent on a tablet is not
  necessarily a configuration defect.
- **`navigation.show_*` is overridden.** Any element defined in a regions
  object, or belonging to the selected preset, takes precedence over the
  matching `navigation.show_*` flag. Once regions are in use, elements are the
  only control surface.
- **Invalid placement is a no-op, not an error.** An element in a region that
  does not allow it, an element name the build does not implement, or the
  `items` region in the wrong layer are all discarded with no `errorListener`
  call and no console output. The absence of an error is therefore not
  evidence of a valid configuration; compare the region's rendered contents
  against the element's allowed-regions entry in `360000758337`.

## Preconditions

- **`rendering_type` must be `"assess"`.** Inline rendering produces no
  regions, `assessApp()` returns nothing, and `dialogs()` raises `50007`.
  `typeof itemsApp.dialogs` is still `function` under inline rendering, so
  branch on `request.rendering_type` rather than on the method.
- **`config` belongs inside `request`.** The init object is
  `{ security, request }`. Examples commonly show the `config` block in
  isolation for brevity; in a real initialization it sits under `request`. If
  `itemsApp.getActivity().config` returns without the expected keys, check
  this first.
- **Runtime methods exist only once the app is ready.** Immediately after
  `init()` returns, `itemsApp.dialogs` and `itemsApp.on` may still be
  `undefined` rather than inert, so a listener registered at that point is
  lost without an error. Register listeners and call methods inside
  `readyListener`; `typeof itemsApp.on` distinguishes the two states.
- **An intro screen suppresses the layout.** Where an activity opens on a
  Start screen, the regions are present in the DOM but hidden. A configuration
  that appears to render nothing may be behind that screen.

## Choosing an approach

Select the least invasive option that satisfies the requirement.

1. **A layout preset** (`regions` as a string). A single line, inheriting the
   maintained layout including elements added in later releases. Applies in
   full or not at all.
2. **A preset plus `region_overrides`.** The appropriate default for most
   requirements: the maintained base is retained and only the difference is
   declared. Note the distinction between the dotted and array forms below.
3. **A full `regions` object.** Only when defining a layout from scratch. All
   elements then become the integrator's responsibility: any element omitted
   is not rendered, including elements added by later releases.

`title_element` renders nothing unless `config.title` is set, which can be
mistaken for a preset failure.

## Worked example: move the horizontal TOC to the top

The `horizontal` preset places the previous button, `horizontaltoc_element`
and next button in `bottom`. To relocate only the TOC to `top`:

```yaml
config:
  regions: horizontal
  region_overrides:
    bottom.horizontaltoc_element: false
    top:
      - type: horizontaltoc_element
```

`bottom: false` would disable the entire region, removing the previous and
next buttons with it. Target the element, not the region.

The two TOC types have disjoint allowed regions — `horizontaltoc_element` in
the top and bottom regions, `verticaltoc_element` in `right` only — so
relocating a TOC between the dock and a horizontal region requires changing
the element type, not only the region name.

## Regions inside sections

This area has the greatest behavioural variation between releases; confirm the
result against the build in use.

| Section config | Result |
| --- | --- |
| `regions` object | Replaces the global layout for that section. No part of the global configuration is retained, and the replacement does not carry forward: a later section without a regions config renders the global layout again. |
| `region_overrides` | Applied on top of the global layout; regions not named retain their global contents. |
| A preset string | Applied, replacing the global layout. Presets are documented as a top-level option, so treat a preset inside a section as build-specific and verify what that section renders before depending on it. |
| `regions` and `region_overrides` together | The two can compose, but the combination is not a documented contract. Use one or the other per section. |
| `items` inside a section's `regions` | Ignored, as documented. |

Where regions carry over from one section into the next, the documented
v2021.2/v2021.3 behavior — the first section merging into the global config
and carrying forward — is the first hypothesis. Establish which build is
serving the page before concluding: read `LearnosityItems.version` in the
console, or the `items.learnosity.com/v…` URL in the network panel. A version
in the script tag does not guarantee that build is the one loaded, so use the
version reported at runtime.

## Custom buttons and dialogs

Behavior beyond the payload shapes given in the reference:

- **`label` is a label bundle reference, not display text.** A reference
  present in the bundle is translated; one that is absent is rendered
  verbatim, so the symptom is a raw identifier as the button text and
  aria-label.
- **Either app object receives button events.** `assessApp().on()` is the form
  used in the examples, and `itemsApp.on()` receives the same event, so bind
  to whichever object is already held.
- **A wildcard listener is not a diagnostic.** `on("*")` may deliver no events
  at all, so silence from it establishes nothing. Subscribe to the named
  event, and validate the listener itself with a known event such as
  `item:changed`.
- **Both dialog call forms are valid** and return the same set:
  `itemsApp.dialogs()` and `itemsApp.assessApp().dialogs()`. Leave existing
  code on whichever form it uses.
- **The dialog set follows the activity.** The reference lists the full
  catalog; a session exposes only the dialogs it has configured — a plain
  assess session has no `configuration` or `idleTimeout`, for example. Verify
  the key exists before calling `show()`.

## Failure modes

### Regions configuration

| Configuration error | Result |
| --- | --- |
| `region: false` used to remove one element | Disables the region and all its contents. Use the dotted form, `region.element: false`. |
| Array form where the dotted form was intended | An array replaces the entire region; the dotted form edits a single element. |
| Both forms for one element | A key occurs once per mapping: `right.calculator_button` is either `false` or an options object. |
| `horizontaltoc_element` added to `right` | The allowed-regions entry for that element lists the top and bottom regions only, and the player follows it: the override renders nothing. Check an element's allowed regions before placing it. |
| `items` outside the global `regions` object | Ignored, so `scrollable_option`, `vertical_element` and `progress_element` must be set globally. |
| A button hidden with `navigation.show_*` | Overridden by the regions config or preset that defines the element. Remove the element instead. |
| An element name the build does not implement | Discarded silently — `highlight_mode_button`, for example, renders nothing on a build predating its release. |

### Responsive rendering

| Condition | Result |
| --- | --- |
| Verification at desktop width only | Below the large breakpoint the `right` region moves into the menu and the top and bottom regions collapse to navigation only. |
| Breakpoint derived by arithmetic | Margins around the player shift the effective boundary. In a plain container the boundaries fall at 801 for large and 481 for medium, with 480 small, measured on the element passed to Items API; confirm the integration's own container with the resize check below. |
| `configuration.responsive_regions: false` | Removes the adaptation on every device, including phones. A deliberate trade-off, not a defect fix. |
| More than seven buttons in `right` | The dock switches to a scrolling menu at approximately the eighth button, and the expand toggle is always added and cannot be positioned. Where the exact count matters, add one button at a time and observe the `menu-with-scroll` class on `.right-wrapper`. |

### Runtime and signing

| Condition | Result |
| --- | --- |
| Handlers registered immediately after `init()` | The methods do not yet exist, and the lost registration is not reported. |
| A dialog key assumed to exist | The set follows the activity's configuration. |
| `config` assembled after signing | The signature covers the entire `request`, so the player reports `41003 Signatures do not match`, whose message references `security.domain` and directs investigation to the wrong cause. |

## Diagnostics

Work symptom to cause, and confirm the cause before changing configuration.

1. **Previous and next buttons removed along with the TOC.** A region was
   disabled where an element override was intended.
2. **A configured element never renders.** Check the allowed-regions entry for
   that element type, then whether it sits in the `items` region outside the
   global config, then whether the name exists in the build in use.
3. **A button persists despite `navigation.show_next: false`.** The regions
   config or preset defines it. Remove the element.
4. **Correct at desktop width, incorrect on a tablet.** The responsive engine,
   not the configuration. Establish the container width, then choose between
   accepting the reflow, moving the element to a region that survives it, and
   disabling `responsive_regions` for every device.
5. **A later section renders unexpected controls.** Read that section's own
   config first: `regions` replaces, `region_overrides` is applied on top. If
   regions carry over between sections, consider the older-LTS merge
   behavior and establish the build in use.
6. **A custom button renders but its event never fires.** Verify the event
   name against `options.name`, then that the handler was registered inside
   `readyListener`. Do not use a wildcard listener to investigate.
7. **A custom button displays a raw identifier.** The label reference is
   absent from the bundle.
8. **`dialogs()` raises an error or has no effect.** In order:
   `rendering_type` is not `"assess"`; the call precedes `readyListener`; the
   dialog key does not exist for this activity. The call form is not the
   cause.
9. **Nothing renders and no error is raised.** Check the intro screen, then
   `rendering_type`. Where the failure is an undetailed `50004`, examine the
   environment before the credentials or the content: Items API loads a
   cross-domain iframe from `items.learnosity.com`, and a browser, extension
   or network policy that blocks it produces exactly this result. Check
   `document.querySelector('iframe[src*="learnosity"]')` and the network
   panel; running the same init in an unrestricted browser confirms it
   fastest.

## Runtime checks

Build-specific and activity-specific questions are answered from the console
of a running player. Prefer these to constructing a test harness.

| Question | Check |
| --- | --- |
| Which build is serving the page? | `LearnosityItems.version` |
| Did the config reach the activity? | `itemsApp.getActivity().config`, compared with the signed request |
| What rendered in a region? | `document.querySelectorAll('.lrn-right-region .lrn_btn').length`, or list the classes: elements carry stable ones such as `test-save`, `lrn-calc-toggle`, `toc lrn-vertical-toc`, `item-next` |
| Which dialogs does this activity expose? | `Object.keys(itemsApp.dialogs())` |
| Is an event firing? | `itemsApp.on('item:changed', console.log)` — a named event, registered inside `readyListener` |
| Where is the responsive boundary for this layout? | Resize the element passed to Items API, not the window, and observe when the `right` region empties |

Run these inside `readyListener` or after the player has started; before that,
the app object does not expose the methods.

## Reference lookups

Call `read_document` with the `source` and `article_id` below. Fetch only what
the current question requires.

| Topic | `source` | `article_id` |
| --- | --- | --- |
| Regions overview: every region, element and button, allowed regions, options, preset contents, responsive breakpoints, regions in sections | `help_site` | `360000758337` |
| `config.regions` initialization option | `help_site` | `16560445464989` |
| Standard dialogs and the `show()`/`hide()` contract | `help_site` | `16461106026013` |
| Custom dialogs: `header` / `body` / `buttons` and button events | `help_site` | `360000755558` |
| A complete signed initialization using `regions` | `demos` | `3811035298584101061` |

## Further reading

| Topic | `source` | `article_id` |
| --- | --- | --- |
| `rendering_type`, the precondition for regions and dialogs | `help_site` | `16560451464605` |
| `config.navigation`, the `show_*` flags regions override | `help_site` | `16560458801821` |
| `itemsApp` methods index (`dialogs()`, `assessApp()`, `on()`) | `help_site` | `16458082061213` |
| Initialization and callbacks, including `readyListener` | `help_site` | `16582668672669` |
| Items API events index | `help_site` | `16458090489117` |
| Label bundles, for `custom_button` labels | `help_site` | `360002918818` |
| Breaking assessments into sections | `help_site` | `360000758297` |
| Signature construction and what it covers | `help_site` | `360000754738` |

## Related skills

- `install-and-set-up-the-sdk` — installing the SDK that signs the
  initialization this configuration travels in.
- `golden-path-author-items-reports` — the Author, Items and Reports flow this
  player configuration sits inside.
