# batchPlay

For anything the DOM doesn't expose (filters, layer styles, fills, adjustments, advanced selections, precise text). Runs **action descriptors** — Photoshop's own internal command objects. If the UI can do it, batchPlay can.

```js
const result = await action.batchPlay(descriptors, options);
```
- `descriptors` — array of commands, run in order.
- `options` — usually `{}`; keys: `synchronousExecution`, `continueOnError`, `immediateRedraw`.
- Returns an array: `result[i]` ↔ `descriptors[i]`. This is your read channel.

Call it directly (a `.psjs` is already modal). Use `bp([...], "label")` from the harness — it runs with `continueOnError` and records the error shape into `errors[]`.

## Descriptor anatomy

```js
{ _obj: "make",                          // the command: make, set, hide, gaussianBlur, ...
  _target: [ /* reference */ ],
  /* command params */
  _options: { dialogOptions: "dontDisplay" } }   // keep dialogs silent
```

**References** address an object, most-specific-first:

| Form | Example |
|---|---|
| id | `{ _ref: "document", _id: 123 }` |
| index (1-based) | `{ _ref: "layer", _index: 2 }` |
| name | `{ _ref: "document", _name: "Untitled-1" }` |
| the active one | `{ _ref: "layer", _enum: "ordinal", _value: "targetEnum" }` |
| property | `{ _ref: "property", _property: "title" }` |

`_enum: "ordinal", _value: "targetEnum"` = the currently active layer/document (the common target).

## Examples (correct, from Adobe docs)

```js
// hide the active layer
await bp([{ _obj: "hide", _target: [
  { _ref: "layer", _enum: "ordinal", _value: "targetEnum" },
  { _ref: "document", _enum: "ordinal", _value: "targetEnum" } ] }], "hide");

// sample a pixel and read it back
const [res] = await action.batchPlay([{ _obj: "colorSampler",
  _target: { _ref: "document", _enum: "ordinal", _value: "targetEnum" },
  samplePoint: { horizontal: 100, vertical: 100 } }], {});
// res.colorSampler -> { _obj: "RGBColorClass", red, green, blue }
```

Read state by sending a `get` descriptor and parsing the return — don't guess. Error shape: `{ _obj: "error", message, result }` (`result` 0 = ok, -128 = user cancelled).

## Don't hand-write descriptors blind — capture them

actionJSON is verbose and easy to get subtly wrong. Reliable path: do the op once in the UI with a listener recording, copy the actionJSON, parameterize.
- **Alchemist** (UXP plugin) — shows the exact descriptor for what you just did. The standard tool.
- **`ps-es-to-uxp`** (Adobe) — converts ExtendScript `executeAction` to batchPlay.

Heuristic: DOM first; batchPlay for gaps; from memory → keep it minimal, run it, check the return before continuing. Don't stack ten unverified descriptors and hope.
