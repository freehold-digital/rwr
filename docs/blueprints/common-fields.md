# Fields Common to Every Blueprint

This page describes the settings that behave the same way in every blueprint
type, and the one decoding rule that applies to all of them.

## Unknown keys are an error

> [!IMPORTANT]
> RWR decodes YAML, JSON, TOML, and CUE blueprints strictly.
> An unknown key stops the run.
> Examples include the misspelling `pacakges:` and the singular `profile:` instead of `profiles:`.
> The error names the file and, for YAML, the line.

An unknown key used to be dropped in silence. A misspelled section then produced
an empty list, every processor found nothing to do, and the run reported success
having changed nothing. A misspelled `profiles` was worse: the entry lost its
scoping and ran on every machine. Both are now reported.

`rwr validate` decodes the same way, so it reports the same error before
anything is applied.

## `profiles`

Every blueprint type supports `profiles`: a list of profile names the entry
belongs to.

An entry with no `profiles` is a base item and is always processed. An entry
with one or more profiles is processed only when a named profile covers it
(`rwr all --profile dev`). With no `--profile` flag, RWR applies only the base
items. See [Profiles](../profiles.md).

```yaml
packages:
  - name: git         # base item, always installed
    action: install

  - name: docker
    action: install
    profiles:
      - dev
      - work
```

## `import`

`import` pulls entries from another blueprint file. RWR resolves the path relative to the file that declares it. Imports can be nested. RWR detects and rejects circular imports.

An import entry cannot also declare resources, but it can carry `profiles`. An Omarchy configuration import also needs `tool: omarchy` so the configuration processor can route it.

Supported by: `packages`, `repositories`, `files`, `templates`, `directories`,
`git`, `scripts`, `services`, `ssh_keys`, `users`, `groups`, `credential_setup`,
and configuration entries that use `tool: omarchy`.

**Not** supported by `fonts` or the other configuration tools. An `import` on a
`dconf`, `gsettings`, `macos_defaults`, or `windows_registry` entry is an error.

## `interactive`

`interactive: true` or `interactive: false` overrides the global `--interactive`
flag for a single entry. Omit it to follow the flag.

Read per entry by `directories`, `packages`, `repositories`, `scripts`,
`services`, `ssh_keys` and `users`. `files`, `templates` and `git` accept the
key but do **not** read it - those processors follow only the global flag. Not
supported at all by `fonts`, `groups` or `configurations`.

## `schema_version`

Every blueprint file can declare a `schema_version` at the top level, next to
its entry list:

```yaml
schema_version: 1
packages:
  - name: git
    action: install
```

A file's `schema_version` overrides the tree-wide version in the init file. This lets one blueprint change schema versions without changing the rest of the tree. Every blueprint type currently supports only version `1`. RWR rejects a version the type does not support.

## `name` and `names`

Every type has `name`. A `names` list, which repeats the rest of the entry once
per name, is read by `files`, `templates`, `packages` and `fonts`.

`directories` and `configurations` accept a `names` key and do **not** read it -
write one entry per item there. Every other type rejects `names` outright.

## Credential dependencies

Scripts, files, templates, and directories accept
`requiresCredentials: [name]` and `onCredentialUnavailable: skip|fail`. These
are checked after profile selection. Exposing a credential alone does not
acquire it. See [credentials](../credentials.md).
