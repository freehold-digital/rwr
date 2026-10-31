# Best Practices

Use these recommendations to keep an RWR setup understandable as it grows. Organize blueprints so the next reader can find a resource, and keep secrets inside the credential model.

## Organize blueprints

1. Use a consistent naming convention for your blueprint files, such as `<blueprint-type>.yaml` (for example `packages.yaml`, `repositories.yaml`).

2. Group related blueprints into subdirectories:

    ```text
    blueprints/
    ├── packages/
    │   ├── core.yaml
    │   └── development.yaml
    ├── repositories/
    │   ├── common.yaml
    │   └── personal.yaml
    └── services/
        └── web.yaml
    ```

3. Split large blueprints into smaller, focused files. RWR routes by top-level keys, so a file containing both `packages:` and `files:` feeds two processors.

4. Use `import` to share base entries across files instead of copying them. See [Common fields](blueprints/common-fields.md#import).

5. Use variables for values that change per machine, such as file paths, URLs, or personal preferences. See [Variables and templates](variables.md).

## Keep a machine reproducible

- Track the tree in Git. RWR re-applies the declared state from any checkout, so a reinstall or a new machine starts from the same description.
- Check a tree with `rwr validate`, preview with `rwr all --dry-run`, then apply. The same sequence works for later changes.
- Set `schema_version` in the init file when you want a tree to hold its schema. An unpinned version follows the latest version after an RWR upgrade. See [Schema versioning](schema-versioning.md).

## Use profiles deliberately

- Name profiles for what they mean on the machine, such as `work`, `gaming`, or `laptop`. Avoid `p1`, `setup`, or other names that only make sense today.
- Keep base items (no `profiles` field) small and shared. Move machine-specific work into profiles.
- Preview a profile combination before applying it: `rwr all --dry-run --profile work,gaming`.
- See [Best practices for profiles](profile-best-practices.md) for organization patterns and the guard-script pattern.

## Keep credentials out of the tree

- RWR withholds your GitHub token and SSH private key from blueprints. Give a credential to a blueprint only when the blueprint needs it, with the `exposeCredentials` key. Read [Credentials](credentials.md) before you use that key.
- Keep vault item references in the init file, and reference them by name from blueprints. Blueprints select names but cannot redirect endpoints.
- RWR removes credential values from logs. `rwr config view --show-secrets` reveals them. Use it only to confirm a value, and do not keep the output.

## What belongs where

| Content | Home |
|---|---|
| Package managers, providers, repository steps | [providers.md](providers.md) |
| Credential sources, scopes, native tasks | [credentials.md](credentials.md) |
| Run records, drift, reversal | [state.md](state.md) |
| Profile organization | [profile-best-practices.md](profile-best-practices.md) |