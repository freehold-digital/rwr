# Init File - The Entrypoint

The Init file is the main entry point for your RWR blueprints. It defines the configuration settings for RWR and specifies the order of execution for the blueprints. This page describes the structure and options available in the Init file.

## File Format

The Init file is typically named `init.yaml`, but `init.yml`, `init.json`,
`init.toml` and `init.cue` work as well. RWR supports YAML, JSON, TOML, and CUE
formats for the Init file.

## File Location

RWR finds the Init file in this order:

1. The path given to `--init-file` / `-i`.
2. The `repository.init-file` key in the [configuration file](cli/configuration.md).
3. `init.yaml`, `init.yml`, `init.json`, then `init.toml` in the current
   directory.

For discovery steps 1 and 2, the path can name a directory. RWR searches that directory for the four filenames in the order above. The path can also be an `https://` URL. RWR rewrites GitHub `/blob/` URLs to their raw addresses.

RWR rejects `http://` URLs because the init file controls everything it runs.

GitHub shorthand supports these forms:

- `owner/repo` fetches the init file from the repository root on its default branch.
- `owner/repo@ref` pins a branch, tag, or commit.
- `owner/repo/path/to/init.yaml` names the file explicitly, with an optional `@ref`.

```bash
rwr all -i freehold-digital/my-blueprints
rwr all -i freehold-digital/my-blueprints@v1.2
rwr all -i freehold-digital/my-blueprints/machines/laptop.yaml
```

A local path that exists always wins over shorthand interpretation, so a
directory literally named `owner/repo` keeps working.

## Init File Structure

The Init file consists of the following main sections:

### `blueprints`

The `blueprints` section defines the configuration settings for your blueprints.

| Field | Description | Required |
|-------|-------------|----------|
| `format` | The format of the blueprint files (yaml, json, toml, cue) | Yes |
| `location` | The directory where the blueprint files are located | No (default: current directory) |
| `order` | The order of the blueprint types | No. The default is a fixed order. See [Blueprints General](blueprints-general.md) |
| `git` | Git repository settings for managing blueprints | No |
| `runOnlyListed` | Whether to run only the blueprints listed in the `order` field | No (default: false) |
| `schema_version` | The blueprint schema version for this tree. See [Schema versioning](schema-versioning.md) | No. The default is the latest version |

> [!NOTE]
> There is no `templatesEnabled` option. Templates are always processed.

> [!IMPORTANT]
> Do not put `packageManagers` in the `order` field. RWR installs the package
> managers from the `packageManagers` section before it reads the blueprint
> types. A `packageManagers` entry in `order` gives an "Unknown processor"
> warning and does nothing.

### `exposeCredentials`

The `exposeCredentials` section gives the credentials that the blueprints in this
tree can read. RWR withholds all credentials by default.

```yaml
exposeCredentials:
  - gh_api_token
```

See [Credentials](credentials.md) for the full description.

### `credentials`

The `credentials` section declares the named credentials that the blueprints in
this tree need. RWR resolves each one at the start of the run, from the first
source that has a value.

```yaml
credentials:
  - name: cachix_token
    description: "Cachix auth token for the nix cache"
    sources: [env:CACHIX_AUTH_TOKEN, keyring, prompt]
    scope: [scripts]
```

Declaring a credential does not give it to blueprints - for that, also name it
under `exposeCredentials`. See [Credentials](credentials.md) for the fields,
the source order, and where values are stored.

### `packageManagers`

The `packageManagers` section defines the configuration settings for package managers.

| Field | Description | Required |
|-------|-------------|----------|
| `name` | The name of the package manager | Yes |
| `action` | The action to perform (install, remove) | Yes |
| `asUser` | The user to run the package manager commands as | No |

### `variables`

The `variables` section holds the custom variables that your blueprints can read.

| Field | Description | Required |
|-------|-------------|----------|
| `userDefined` | A map of your own variables. Each key becomes `{{ .UserDefined.<key> }}` in a blueprint. The values may be strings, numbers, lists or nested maps | No |

```yaml
variables:
  userDefined:
    app_version: 1.0.0
    editors:
      - vim
      - neovim
```

`userDefined` is the only field you set here. Blueprints can also use the `{{ .User }}`, `{{ .System }}`, and `{{ .Flags }}` groups. RWR fills those groups from the machine and command-line flags. The init file cannot set them. A blueprint therefore cannot override the reported user or flags. See [Variables and Templating](variables.md).

## Example Init File

Example 1: Define a blueprint tree in `init.yaml`.

```yaml
blueprints:
  format: yaml
  location: blueprints
  order:
    - packages
    - repositories
    - files
  git:
    url: https://github.com/yourusername/rwr-blueprints.git
    branch: main
  runOnlyListed: true

packageManagers:
  - name: brew
    action: install

variables:
  userDefined:
    app_version: 1.0.0
    api_key: abc123
```

In this example, the Init file specifies the format and location of the blueprint
files and the order of execution. It also configures package managers,
and defines custom variables. A blueprint in this tree reads them
as `{{ .UserDefined.app_version }}` and `{{ .UserDefined.api_key }}`.

### Package Manager Installation

The Init Process includes the ability to install and configure various package managers. This is particularly useful for setting up a consistent environment across different systems. Supported package managers include:

#### Homebrew (brew)

For macOS and Linux.

#### Nix (nix)

For macOS and Linux.

#### Chocolatey (chocolatey)

For Windows.

#### Scoop (scoop)

For Windows.

#### AUR Helpers

For Arch Linux:

- Yay (yay)
- Paru (paru)
- Trizen (trizen)
- Pamac (pamac)
- Aura (aura)

#### Cargo (cargo)

Rust package manager.

#### GNOME Extensions CLI (gnome-extensions)

For managing GNOME extensions.

Example configurations for package manager installation:

```yaml
packageManagers:
  - name: brew
    action: install
  - name: nix
    action: install
  - name: cargo
    action: install
    asUser: johndoe
  - name: gnome-extensions
    action: install
```

```json
{
  "packageManagers": [
    {
      "name": "brew",
      "action": "install"
    },
    {
      "name": "nix",
      "action": "install"
    },
    {
      "name": "cargo",
      "action": "install",
      "asUser": "johndoe"
    },
    {
      "name": "gnome-extensions",
      "action": "install"
    }
  ]
}
```

```toml
[[packageManagers]]
name = "brew"
action = "install"

[[packageManagers]]
name = "nix"
action = "install"

[[packageManagers]]
name = "cargo"
action = "install"
asUser = "johndoe"

[[packageManagers]]
name = "gnome-extensions"
action = "install"
```

These configurations can be included in your init file to ensure that the necessary package managers are installed during the initial setup process.

## Best Practices

- Keep your Init file concise and organized
- Use meaningful names for your variables
- Store sensitive information (e.g., API keys) in environment variables or secure vaults
- Use the `order` field to define the execution order of your blueprints explicitly

For more information on the specific blueprint types and their configuration options, please refer to the respective blueprint type documentation pages.

## Credential policy

`blueprints.except` lists processors excluded from full runs. Named invocations override init exclusions. CLI exclusions win.

`credentialPolicy.setup` defaults to `explicit`. Use `ordered` to include provider setup deliberately in full runs. `credentialPolicy.onUnavailable` defaults to `skip`. Use `fail` for strict dependencies.

Trusted `credentialProviders`, `credentials`, and `credentialAttachments` authorize native tasks declared in `credential_setup` blueprints. See [credentials](credentials.md).
