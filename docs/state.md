# Run records

Every real run (`rwr all`, `rwr run <processor>`) leaves a record of what it
applied: one JSON file per run under `<configdir>/state/runs/`, plus a
`latest` file naming the newest one. Dry runs write nothing.

The record is evidence, not desired state. Blueprints define the desired machine state. `rwr status` uses the record to recognize files RWR wrote. `rwr uninstall` uses it to reverse recorded changes.

## Format

One append-only event log: `<configdir>/state/journal.jsonl`. Every line is
a self-contained JSON event. Nothing is ever rewritten or mutated.

```json
{"v":2,"kind":"run","id":"20260803-204243-7b0f","started":"...","location":"/path/to/blueprints"}
{"v":2,"kind":"apply","run":"20260803-204243-7b0f","processor":"files","action":"create","identity":{"name":"h.txt","dest":"/home/user/h.txt","sha256":"2d71..."},"outcome":"ok","ok":true}
{"v":2,"kind":"finish","run":"20260803-204243-7b0f","finished":"..."}
{"v":2,"kind":"reverse","run":"20260803-2050-1a2b","processor":"files","identity":{"name":"h.txt","dest":"/home/user/h.txt","sha256":"2d71..."}}
```

- Appends are O(1). A crash loses at most a partial last line, and every
  complete line before it still counts. A run without a `finish` event is
  unfinalized.
- `identity` carries what a later run needs to find the thing again:
  `provider` + `name` for packages, `dest` + `sha256` for files and
  templates, `dest` for directories, `target` for git checkouts, `dir` for
  fonts.
- `rwr uninstall` appends `reverse` events. Readers fold them over the
  applies. History is never edited.
- The journal is user-only (`0600`, directory `0700`). Legacy v1 per-run
  record files under `state/runs/` are still read and folded in.

## `rwr status`

Read-only, never elevates, exits 1 on drift. Classes per item:

| Class | Meaning |
|-------|---------|
| `in-sync` | present and (where a hash is recorded) unmodified |
| `missing` | desired but not found |
| `modified` | found, but content differs from the recorded apply |
| `unknown` | Not queryable: scripts, configuration, users, SSH keys, repositories, or a provider without a usable list operation |
| `stale` | recorded by a past run, no longer in the tree |

## `rwr uninstall`

Uninstall reads the run record, never the blueprint tree. Without a record, it refuses to run. Before confirmation, it lists non-reversible changes: scripts, configuration writes, users, uploaded SSH keys, and repositories.

Removal runs in reverse apply order. RWR checks file hashes and Git checkout cleanliness. It skips modified content and lists it. Failures do not stop removal, but they produce a non-zero exit code. Failed changes remain unreversed so a rerun can retry them.

`--yes` skips confirmation. `--dry-run` prints the plan without changing anything.
