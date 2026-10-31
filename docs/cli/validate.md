# Validate Command

The `validate` command in RWR helps you verify your blueprints and provider configurations before running them. This ensures that your configurations are correct and will work as expected when deployed.

## Overview

The `validate` command checks blueprint and provider configurations for errors before a run.

```bash
rwr validate [flags]
```

The validation process includes:

* Checking blueprint structure and content
* Validating provider configurations
* Resolving blueprint imports and detecting circular ones
* Checking that a referenced package manager has a provider

> [!NOTE]
> `validate` reads files. It never installs anything and never runs a blueprint's
> commands, so it cannot tell you that a package exists in a repository or that a
> script succeeds.
>
> `--profile` is accepted, because it is a global flag, but `validate` ignores
> it: every blueprint file in the tree is checked regardless of profiles.

## Command Flags

| Flag | Description |
|------|-------------|
| `--blueprints` | Force validation as blueprint files |
| `--providers` | Force validation as provider configurations |
| `--verbose` | Log a "validation completed" line when the walk finishes |

Give the path as an argument, not as a flag. The default is the current
directory:

```bash
rwr validate path/to/blueprints
```

Without `--blueprints` or `--providers`, RWR chooses validation by path. It treats a directory named `providers` or a `.toml` file as providers. It treats other paths as blueprints. Each invocation validates one type, never both.

Validating as blueprints needs an init file (`init.json`, `init.yaml`,
`init.yml` or `init.toml`) at or above the files being checked; a tree with no
init file cannot be validated.

> [!NOTE]
> `--verbose` does not change what is checked or add per-file output. Use
> `--debug` for a detailed trace.

## Validation Process

### Blueprint Validation

Blueprint validation checks your blueprint files for structural and content issues.

The blueprint validation process includes:

* RWR decodes each file strictly into its blueprint type. Unknown keys are errors rather than silently ignored fields.
* Required fields must be present. Examples include repository `url`, SSH key `name`, `type`, and `path`, and file `content` or `source`.
* RWR checks `action` values against the processor's supported actions. It also checks file modes against accepted values.
* Every `import` path must exist and parse as the expected blueprint type. Circular imports are errors.
* A blueprint's `package_manager` must name a provider available on the system.

### Provider Validation

Provider validation checks your provider configuration files for structural and compatibility issues.

The provider validation process includes:

* **File Type**: A provider configuration must be a `.toml` or `.json` file
  (an error otherwise)
* A missing `provider.name` is an error. Missing detection `binary` or `distributions` fields produce warnings. Missing `install`, `update`, or `remove` commands also produce warnings.
* Every install, remove, or repository step must name a supported action and provide its required fields. Examples include `exec` for `command` and `source`/`dest` for `download` and `copy`. Missing required fields are errors. Empty `content` on a `write` or `append` step is a warning.
* RWR warns when a declared repository sources path does not exist on the system.

## Error Reporting

Each issue is logged as it is found, with the file it came from and a suggested
fix:

```text
ERRO ERROR: No URL specified for repository 'core' [/path/to/repositories/main.yaml] - Suggestion: Add URL field to repository
WARN WARNING: Location does not exist: blueprints [/path/to/init.yaml] - Suggestion: Create the directory or update the location
```

The command ends with a summary, and exits non-zero if there was at least one
error:

```text
validation failed with 2 errors and 1 warnings
```

With no errors it prints `Validation completed with N warnings`, or
`Validation completed successfully`.

> [!NOTE]
> Issues carry the file, not a line number. Where the underlying decoder reports
> a line - a strict-decoding failure, for example - it appears inside the message
> text.

## Examples

### Validating the Current Directory

```bash
rwr validate
```

### Validating a Specific Path

```bash
rwr validate /path/to/blueprints
rwr validate providers/paru.toml
```

### Forcing a Mode

```bash
rwr validate path/to/dir --blueprints
rwr validate path/to/file --providers
```

## Common Validation Errors

### Blueprint Errors

| Message | Meaning |
|---------|---------|
| `field <name> not found in type types.<T>` | The blueprint uses a key that type does not have. Blueprints decode strictly, so this is an error rather than a silently ignored key |
| `No URL specified for repository '<name>'` | A repository entry has no `url` |
| `No content or source specified for file '<name>'` | A file entry gives neither |
| `No type specified for SSH key '<name>'` | An SSH key entry has no `type` |
| `Circular import detected: '<path>'` | An import chain returns to a file it already read |
| `Import file not found '<path>'` | An `import` path does not resolve |
| `Location does not exist: <path>` | The init file's `blueprints.location` is not there (a warning) |
| `Failed to find init file` | There is no `init.*` at or above the path being validated |

### Provider Errors

| Message | Severity | Meaning |
|---------|----------|---------|
| `Not a provider file` | Error | Provider configurations must be `.toml` or `.json` |
| `Missing required field 'provider.name'` | Error | The `[provider]` section has no `name` |
| `Missing binary in detection section` | Warning | The provider does not say which binary to look for |
| `No distributions specified in detection section` | Warning | The provider does not say which distributions it supports |
| `Missing install command` / `Missing update command` / `Missing remove command` | Warning | A command is absent from `[commands]` |
| `Unsupported action "…" in … step` | Error | A step names an action the processor does not implement |
| `Missing exec/source/dest/path/… in … step` | Error | A step lacks a field its action needs |
| `No providers available for the current system` | Warning | Nothing was detected on this machine |

Only errors make the exit code non-zero. A run that produces warnings alone
still exits `0`.

## Best Practices

* Run validation before attempting to run your configurations
* Fix all errors and warnings before proceeding with deployment
* Validate both blueprints and providers, in two runs, to cover both
* Use `--debug` when a message is not enough to locate the problem

## See Also

* [Commands and Flags](command-and-flags.md)
* [Blueprints Overview](../blueprints-general.md)
* [Providers](../providers.md)
* [Best Practices](../best-practices.md)
