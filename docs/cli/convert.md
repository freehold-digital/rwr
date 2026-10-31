# rwr convert

Convert a blueprint tree between formats, or migrate deprecated constructs to
their current equivalents. Dry-run by default - nothing is written without
`--write`.

## Format conversion

```bash
rwr convert --to toml path/to/tree
rwr convert --to cue --write path/to/tree
```

Every blueprint, init, bootstrap, and manifest file is decoded and re-encoded
in the target format. Originals are replaced only under `--write`.

Limits, stated plainly:

- **Comments are not preserved.** Conversion is a decode/encode cycle and no
  format's comment model maps onto another's. The command warns per file that
  carries comments so you can port them by hand.
- **Template placeholders survive as quoted strings** (`"{{ .User.home }}"`).
  A file whose templates make it unparseable raw is reported and skipped,
  never mangled.
- **CUE output is JSON-form CUE** - valid and lossless. Idiomatic CUE
  (schemas, constraints) is authoring work the converter does not guess at.

## Migration

```bash
rwr convert --migrate path/to/tree
rwr convert --migrate --write path/to/tree
```

Rewrites deprecated constructs. Current rules:

- Inline resource sections such as `packages:` and `repositories:` move from `init.yaml` into blueprint files such as `packages/from-init.yaml`. RWR previously decoded these sections without applying them. They no longer belong to the init schema. An init file containing them is invalid until migrated.

`--to` and `--migrate` compose in one invocation.
