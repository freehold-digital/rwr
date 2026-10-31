# Packages Blueprint

The Packages Blueprint allows you to manage packages on your system using RWR. You can specify packages to be installed or removed using various package managers, and now you can also provide additional arguments for package installation.

See [Fields Common to Every Blueprint](common-fields.md) for `profiles`,
`import`, `interactive`, and the rule that an unknown key is an error.

## Blueprint Structure

The Packages Blueprint has the following structure:

```yaml
packages:
  # Single package with base installation (no profiles)
  - name: package1
    action: install
    package_manager: apt
    args:
      - "--no-install-recommends"

  # Single package removal
  - name: package2
    action: remove
    package_manager: brew

  # Multiple packages with profiles
  - names:
      - package3
      - package4
    action: install
    package_manager: chocolatey
    profiles:
      - work
    args:
      - "--params"
      - "'/NoDesktopShortcut'"
```

## Blueprint Settings

The Packages Blueprint supports the following settings:

| Setting | Required | Description |
|---------|----------|-------------|
| `name` | Yes, if `names` or `import` is not provided | The name of the package to manage. It may **not** begin with `-` |
| `names` | Yes, if `name` or `import` is not provided | A list of package names to manage. The same restriction applies to each |
| `import` | Yes, if `name` or `names` is not provided | Path to import package definitions from another file (relative to blueprint directory) |
| `action` | Yes | `install` or `remove`. See [Actions](#actions) |
| `package_manager` | No | The package manager to use (e.g., `apt`, `brew`, `chocolatey`) |
| `elevated` | No | Requests elevation in addition to the provider's requirements. An entry can add elevation, such as for a user-scoped manager targeting a system path. It cannot remove elevation required by the provider |
| `args` | No | Additional arguments to pass to the package manager (as a list of strings), appended after the package name |
| `profiles` | No | List of profiles this package belongs to. If empty, package is always installed (base item) |
| `interactive` | No | Override global interactive mode for this package (`true`/`false`). If omitted, uses the global `--interactive` flag |

Note that you must provide either `name`, `names`, or `import` for each package entry. If both `name` and `names` are given, the `names` list is processed and `name` is ignored (with a warning), matching how `files` and `fonts` behave.

### Package names may not begin with `-`

Package managers read names starting with `-` as options, such as `--allow-downgrades` or `-U <url>`. RWR rejects these names and records a failure for the entry. The rest of the run continues.

RWR runs commands as argv, not through a shell. This is not shell injection. It can still let a blueprint change the elevated package manager's behavior.

### Actions

`install` and `remove` are implemented. Any other value - including `update` -
is reported by `rwr validate` and recorded as a failure with "unknown action"
at run time. To refresh package lists, run a repositories blueprint - RWR runs
each available provider's update command after processing it.

## Blueprint Imports

You can import package definitions from other files to share common package lists across multiple configurations:

```yaml
packages:
  # Import shared base packages
  - import: ../../Common/packages/base-packages.yaml

  # Import development tools
  - import: ../shared/dev-tools.yaml

  # Add system-specific packages
  - names:
      - system-specific-tool
      - custom-package
    action: install
    package_manager: apt
```

Import features:

- Paths are resolved relative to your blueprint directory
- Prevents circular imports automatically
- Works with all package managers and formats
- Imported packages respect profile filtering
- Multiple imports can be used in a single file

For complete import examples, see [`examples/imports/`](../../examples/imports/).

## Supported Package Managers

RWR ships provider definitions for:

- Linux: `apt`, `dnf`, `zypper`, `pacman`, `apk`, `xbps`, `eopkg`, `emerge`, `slackpkg`, the AUR helpers `yay`, `paru`, `aura`, `pamac` and `trizen`, plus `flatpak`, `snap` and `gnome-extensions`
- macOS: `brew`, `macports`, `mas`
- Windows: `chocolatey`, `scoop`, `winget`
- Cross-platform: `nix`, `cargo`

There is no `yum` provider. Use `dnf`.

If `package_manager` is omitted, RWR uses the machine's detected default. Detection honors `/etc/os-release` and, on Arch, the AUR helper preference order. If the default is unavailable, RWR chooses the alphabetically first available provider. This keeps the manager choice consistent across runs.

If an entry names an unavailable package manager, RWR records that entry as failed. It continues with entries whose managers are available.

## Examples

Here are some examples of using the Packages Blueprint in different formats:

### YAML

```yaml
packages:
  # Base packages - always installed (no profiles field)
  - names:
      - git
      - curl
      - vim
    action: install
    package_manager: apt
    args:
      - "--no-install-recommends"

  # Development profile packages
  - names:
      - nodejs
      - python3
      - docker
    profiles:
      - dev
    action: install
    package_manager: apt

  # Work profile packages with multiple package managers
  - names:
      - visual-studio-code
      - google-chrome
      - brave-browser
    profiles:
      - work
    action: install
    package_manager: brew
    args:
      - "--cask"
```

### JSON

```json
{
  "packages": [
    {
      "names": [
        "git",
        "curl",
        "vim"
      ],
      "action": "install",
      "package_manager": "apt",
      "args": [
        "--no-install-recommends"
      ]
    },
    {
      "names": [
        "nodejs",
        "python3",
        "docker"
      ],
      "profiles": ["dev"],
      "action": "install",
      "package_manager": "apt"
    },
    {
      "names": [
        "visual-studio-code",
        "google-chrome"
      ],
      "profiles": ["work"],
      "action": "install",
      "package_manager": "brew",
      "args": ["--cask"]
    }
  ]
}
```

### TOML

```toml
# Base packages - always installed (no profiles field)
[[packages]]
names = ["git", "curl", "vim"]
action = "install"
package_manager = "apt"
args = ["--no-install-recommends"]

# Development profile packages
[[packages]]
names = ["nodejs", "python3", "docker"]
profiles = ["dev"]
action = "install"
package_manager = "apt"

# Work profile packages
[[packages]]
names = ["visual-studio-code", "google-chrome", "brave-browser"]
profiles = ["work"]
action = "install"
package_manager = "brew"
args = ["--cask"]
```

These examples demonstrate how to specify packages to be installed using different package managers and formats, including additional arguments for package installation.

## Additional Notes

- RWR does not check whether a package is already installed. It runs the provider's install command and lets the package manager decide. Every shipped provider's install command is idempotent for an already-installed package.
- An individual package installation or removal failure does not stop the run. RWR collects failures and reports them at the end. If an error shows that the package manager itself is unavailable, RWR stops issuing package commands. Examples include a locked pacman database or failed sudo authentication. RWR reports a bounded, redacted stderr diagnostic instead of repeating the failure for every package.
- Pacman installs use the existing synchronized package databases with `pacman -S --needed --noconfirm`. RWR does not refresh the databases for each package, and pacman treats already up-to-date targets as satisfied.
- If an entry names an unavailable package manager, RWR records that entry as failed. It continues with entries whose managers are available.
- The `args` field allows you to pass additional arguments to the package manager. This is particularly useful for package managers like Homebrew (with `--cask`), Chocolatey (with installation parameters), or apt (with `--no-install-recommends`).

For more information on using the Packages Blueprint in your RWR configuration, please refer to the [Blueprints Overview](../blueprints-general.md) and the [Commands and Flags](../cli/command-and-flags.md) pages.
