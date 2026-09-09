# Artifacts

An **artifact** is any command, template, script, or hook Spec Kit exposes in a project, regardless of which layer contributes it — built-in assets, an installed preset, an installed extension, or a project-local override in `.specify/templates/overrides/`.

The `specify artifact` command group is the read-only introspection surface for that inventory. `specify preset resolve <name>` answers "which file wins for this preset-managed name?"; `specify artifact` answers "what exists at all, and what is the full composition stack behind it?" — including built-in artifacts that no preset touches.

Both subcommands currently require `--json`. Omitting it exits with code `2` and prints a usage message on stderr; no stdout is produced. Text rendering is deliberately deferred so the JSON shapes below are the only contract, and adding a default text renderer later stays a non-breaking, additive change.

## List Artifacts

```bash
specify artifact list --json
```

| Option   | Description                                              |
| -------- | -------------------------------------------------------- |
| `--json` | Required. Emit the inventory as a JSON array on stdout.  |

Prints the full inventory of every visible artifact — one row per `(kind, name)` pair, including its composition `stack`. Command, template, and script rows are sorted by kind and then by name. Hook rows follow those three kinds and use the hook-specific ordering described in [Sort order](#sort-order).

```json
[
  {
    "id": "command:speckit.specify",
    "name": "speckit.specify",
    "kind": "command",
    "description": "Create or update the feature specification.",
    "stack": [
      {
        "id": "command:speckit.specify",
        "layer": null,
        "sourceId": null,
        "presetId": null,
        "presetName": null,
        "strategy": "replace",
        "active": true,
        "hidden": false,
        "manifestPath": null,
        "lookupId": null,
        "sourcePath": null
      }
    ]
  },
  {
    "id": "script:create-new-feature",
    "name": "create-new-feature",
    "kind": "script",
    "description": "Create a new feature branch and spec directory.",
    "stack": [
      {
        "id": "script:create-new-feature",
        "layer": null,
        "sourceId": null,
        "presetId": null,
        "presetName": null,
        "strategy": "replace",
        "active": true,
        "hidden": false,
        "manifestPath": null,
        "lookupId": null,
        "sourcePath": null
      }
    ]
  }
]
```

| Field         | Description                                                               |
| ------------- | ------------------------------------------------------------------------- |
| `id`          | `{kind}:{name}` — the shorthand `artifact info` accepts as its argument (for hooks, `hook:{eventName}:{targetCommand}`) |
| `name`        | Logical artifact name (commands use the `speckit.<stem>` namespace; hooks use `{eventName}:{targetCommand}`) |
| `kind`        | One of `command`, `template`, `script`, `hook`                             |
| `description` | Description from the highest-precedence layer that declares one, else `""` |
| `stack`       | Composition stack for this artifact, using the same row shape as `artifact info` |

Hook rows carry additional fields describing their event, command, and runtime registration state — see [Hook artifacts](#hook-artifacts) below.

Built-in artifacts always appear, even when nothing overrides them. Descriptions come from the highest-priority layer that has one — a preset or project override that hides a built-in command reports its own description, not the hidden built-in text. Skills (`.github/skills/**/SKILL.md`) are excluded: they are integration-specific output, not a shipped asset family.

## Artifact Info

```bash
specify artifact info <name> --json
```

| Option           | Description                                                        |
| ---------------- | ------------------------------------------------------------------- |
| `--json`         | Required. Emit the composition stack as a JSON object on stdout.    |
| `--kind <kind>`  | Narrow the lookup to `command`, `template`, `script`, or `hook`     |

`<name>` accepts either a bare name (`speckit.specify`) or the `kind:name` shorthand (`command:speckit.specify`). For hooks, the shorthand is `hook:{eventName}:{targetCommand}` — the bare hook name (`{eventName}:{targetCommand}`) is also accepted. When both the shorthand and `--kind` are supplied they must agree.

```json
{
  "id": "command:speckit.specify",
  "name": "speckit.specify",
  "kind": "command",
  "description": "Create or update the feature specification.",
  "stack": [
    {
      "id": "command:speckit.specify",
      "layer": "preset",
      "sourceId": "compliance",
      "presetId": "compliance",
      "presetName": "Compliance Preset",
      "strategy": "replace",
      "active": true,
      "hidden": false,
      "manifestPath": ".specify/presets/compliance/preset.yml",
      "lookupId": "preset:compliance:command:speckit.specify",
      "sourcePath": ".github/skills/speckit-specify/SKILL.md"
    },
    {
      "id": "command:speckit.specify",
      "layer": null,
      "sourceId": null,
      "presetId": null,
      "presetName": null,
      "strategy": "replace",
      "active": false,
      "hidden": true,
      "manifestPath": null,
      "lookupId": null,
      "sourcePath": null
    }
  ]
}
```

The top-level `id`, `name`, `kind`, `description`, and `stack` fields match the corresponding row on `artifact list --json`.

### Stack semantics

`stack` is ordered by resolution precedence: index `0` is the layer that wins. Each row describes one contributing layer:

| Field          | Description                                                                     |
| -------------- | -------------------------------------------------------------------------------- |
| `id`           | `{kind}:{name}` — the source-agnostic round-trip key, identical on every row of the same artifact's stack |
| `layer`        | `project`, `preset`, or `extension`; `null` for built-in layers                    |
| `sourceId`     | Source component of `lookupId`, or `null` when the layer has no provenance         |
| `presetId`     | Preset pack directory id; `null` on built-in, `project`, and `extension` rows       |
| `presetName`   | Preset display name when its manifest declares one, else the pack id; `null` when `presetId` is `null` |
| `strategy`     | `replace`, `wrap`, `prepend`, or `append`                                         |
| `active`       | `true` only for index `0` — the layer whose content is served                     |
| `hidden`       | `true` when a lower-index `replace` layer cuts this layer out of the composition |
| `manifestPath` | Project-relative path to the declaring manifest, or `null` when none applies      |
| `lookupId`     | Deterministic `{layer}:{sourceId}:{kind}:{name}` identifier, or `null` for built-in layers |
| `sourcePath`   | Project-relative POSIX path to the concrete file backing the layer, or `null` for built-in/synthetic layers |

`active` and `hidden` are independent labels, not opposites. `active` identifies the highest-precedence layer selected by the existing Spec Kit layer-resolution order; it does not validate that the layer content can be read or composed. This preserves the diagnostic behavior of `specify preset resolve`, which reports the discovered layer chain even when content composition later produces a warning. Composing strategies (`wrap`, `prepend`, `append`) keep lower layers in the composed output, so an inactive layer is not necessarily hidden: only layers below the first `replace` layer are marked `hidden`. Built-in rows have no provenance: `layer`, `sourceId`, and `lookupId` are `null` — but `id` is always populated, even on built-in rows. `id` is the round-trip key: `specify artifact info` accepts it as input (for example, `specify artifact info command:speckit.specify --json`), and it resolves the same artifact whether the caller passes the bare name or the `id`.

Lookup IDs use the same grammar as [preset contribution identifiers](presets.md#contribution-identifiers) and [extension contribution identifiers](../../extensions/EXTENSION-API-REFERENCE.md#contribution-identifiers), so a `lookupId` from this command joins directly to `PresetManifest.iter_contributions()` / `ExtensionManifest.iter_contributions()` for manifest-declared layers. Project-local overrides carry a synthetic `project:_:{kind}:{name}` ID that intentionally matches no manifest contribution. `lookupId` is manifest-backed layer provenance, not the round-trip key — use `id` for that. `sourcePath` is populated only when the layer maps to a concrete installed preset/extension file or a tracked agent materialization; core, project-override, and other synthetic rows report `null`.

## Hook artifacts

Hook rows extend the shape above with a few fields that only apply to hooks. A hook row's public identifier is `hook:{eventName}:{targetCommand}` — that string is the round-trip key that `artifact info` accepts, and `name` is the same value with the `hook:` prefix stripped.

```json
{
  "id": "hook:before_specify:speckit.compliance.pre-check",
  "name": "before_specify:speckit.compliance.pre-check",
  "kind": "hook",
  "description": "Compliance pre-check guard",
  "eventName": "before_specify",
  "targetCommand": "speckit.compliance.pre-check",
  "registered": true,
  "stack": [
    {
      "id": "hook:before_specify:speckit.compliance.pre-check",
      "layer": "extension",
      "sourceId": "compliance-fast",
      "presetId": null,
      "presetName": null,
      "strategy": "additive",
      "active": true,
      "hidden": false,
      "manifestPath": ".specify/extensions/compliance-fast/extension.yml",
      "lookupId": "extension:compliance-fast:hook:before_specify:speckit.compliance.pre-check",
      "priority": 5,
      "optional": false
    },
    {
      "id": "hook:before_specify:speckit.compliance.pre-check",
      "layer": "extension",
      "sourceId": "compliance-audit",
      "presetId": null,
      "presetName": null,
      "strategy": "additive",
      "active": true,
      "hidden": false,
      "manifestPath": ".specify/extensions/compliance-audit/extension.yml",
      "lookupId": "extension:compliance-audit:hook:before_specify:speckit.compliance.pre-check",
      "priority": 10,
      "optional": true
    }
  ]
}
```

### Top-level fields

| Field           | Description                                                                                     |
| --------------- | ----------------------------------------------------------------------------------------------- |
| `eventName`     | The event whose occurrence triggers this hook (`before_specify`, `after_plan`, …)               |
| `targetCommand` | The command the hook proposes to run when the event fires                                        |
| `registered`    | `true` when at least one matching `.specify/extensions.yml` binding is enabled                   |

There are no row-level `optional` or `priority` fields because hooks do not have a single winner. Those values remain on each stack entry, where they describe that contributor's manifest declaration.

### Hook stack entries

Hook stack entries retain the common stack fields and add per-contributor `priority` and `optional`. Hooks execute additively across extensions, so priority never suppresses another declaration and `hidden` is always `false`. Extension entries use `null` for `presetId` and `presetName`; a future preset hook would populate them consistently with other preset contributions. `manifestPath` points to the manifest that declares the hook.

| Field       | Description                                                                                 |
| ----------- | -------------------------------------------------------------------------------------------- |
| `id`        | The row-shorthand `hook:{eventName}:{targetCommand}`, identical on every entry               |
| `layer`     | Always `preset` or `extension` (never `null`, never `project`, never the built-in tier)      |
| `sourceId`  | The contributing pack's manifest id                                                          |
| `presetId`  | Installed preset id for a preset declaration, otherwise `null`                              |
| `presetName` | Preset display name for a preset declaration, otherwise `null`                             |
| `strategy`  | Always `"additive"` because enabled contributors all execute                                |
| `active`    | `true` exactly when this declaration matches an enabled runtime registration returned by `HookExecutor.get_hooks_for_event()`; multiple entries may be `true` |
| `hidden`    | Always `false`; additive hook declarations do not hide one another                         |
| `manifestPath` | Project-relative path to the manifest declaring the hook                                  |
| `lookupId`  | The manifest identifier: `{layer}:{sourceId}:hook:{eventName}:{targetCommand}`               |
| `priority`  | Priority declared by the contributing manifest (ascending values are registered to run earlier by default) |
| `optional`  | Optional flag declared by the contributing manifest                                          |

### `registered` semantics

`registered` reflects the project's runtime binding state under `.specify/extensions.yml` and MUST match the runtime's own execution decision. Each stack entry is independently `active` when an entry in the event's binding array (a) names that contributor via `extension`, (b) matches the command or omits it, and (c) is not explicitly `enabled: false`. Top-level `registered` is `true` when any stack entry is active. This mirrors the runtime: `HookExecutor.get_hooks_for_event` returns every enabled entry, sorted by priority.

`active` describes registration state only. It does not identify a priority winner or evaluate the hook's optional event-time `condition`; condition filtering happens later in `HookExecutor.check_hooks_for_event()`.

This division is consistent with Spec Kit's existing hook model: the installed extension manifest declares the hook and its defaults, while `.specify/extensions.yml` records the project's registered and enabled runtime state. Registration normally copies `priority` and `optional` from the manifest, but this artifact view does not attempt to reconcile later manual drift between those files; that broader registration concern is outside this command.

A declared hook whose contributors have **no** matching binding entry still appears in the inventory with `registered: false`. This is intentional: `artifact list --json` describes what an extension declares, and `registered` tells you whether the runtime will actually invoke it. Consistent with Spec Kit's existing hook runtime, an invalid or unreadable `.specify/extensions.yml` is normalized to an empty bindings map — every declared hook then reports `registered: false` and no error is raised to callers.

### Layer invariant

Hooks only appear on `preset` or `extension` stack entries. There is no built-in ("core") hook tier — the identifier grammar itself refuses to build hook IDs on any other layer, and `derive_hook_id` will raise `IdentifierComponentError` for a `layer` outside `{preset, extension}`. In practice, no built-in preset today emits hooks; extensions are the only source. Even so, the grammar reserves the preset layer for forward compatibility.

Runtime bindings under `.specify/extensions.yml` that name an extension or command no installed extension has declared do **not** synthesize a phantom row — the inventory is manifest-driven, and orphan bindings only influence the `registered` flag on rows that already exist.

### Sort order

Hooks appear after all `command` / `template` / `script` rows in `artifact list --json`. Within the hook block, rows are sorted primarily by `eventName` (alphabetical) and secondarily by the first stack entry's manifest-declared priority (ascending). Stack entries use declared priority ascending, with original insertion order preserved for ties.

## JSON Errors

On failure, nothing is written to stdout. A single-key JSON envelope is written to stderr and the process exits with code `1`:

```json
{ "error": "unknown artifact hook:before_specify:absent.cmd" }
```

| Message                                             | Cause                                                            |
| --------------------------------------------------- | ---------------------------------------------------------------- |
| `not a Spec Kit project: no .specify/ directory found` | Run outside an initialized project                             |
| `unknown artifact <name>`                           | No artifact matches the requested name (and kind, when given) — same envelope for unknown hooks (`hook:{event}:{command}`) |
| `ambiguous artifact <name>: matches kinds [...]`    | The bare name matches more than one kind — re-run with `--kind`   |
| `artifact resolution failed`                        | The extension registry could not be read, or an error prevented manifest contributions or artifact layers from being collected. Runtime hook configuration uses the tolerant behavior described above. |

Exit code `2` is reserved for usage errors — a missing `--json` flag or an invalid `--kind` value (accepted: `command`, `template`, `script`, `hook`) — and emits a plain-text message on stderr rather than a JSON envelope.
