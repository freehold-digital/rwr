# The Files Blueprint

The Files blueprint manages files through copy, move, delete, create, and modification actions. It also renders template files.

See [Fields Common to Every Blueprint](common-fields.md) for `profiles`,
`import`, `interactive`, and the rule that an unknown key is an error.

## Blueprint Structure

The Files Blueprint has the following structure:

```yaml
files:
  - name: <file_name>
    action: <action>
    source: <source_path_or_url>
    target: <target_path>
    content: <file_content>
    owner: <owner>
    group: <group>
    mode: <mode>
    elevated: <elevated>

templates:
  - name: <template_name>
    action: <action>
    source: <source_path_or_url>
    target: <target_path>
    owner: <owner>
    group: <group>
    mode: <mode>
    elevated: <elevated>
    variables:
      <variable_name>: <variable_value>
```

## Blueprint Settings

The following settings are available for the Files Blueprint:

| Setting | Required | Description |
|---------|----------|-------------|
| `name` | Yes, if `names` or `import` is not provided | The name of the file or template. |
| `names` | Yes, if `name` or `import` is not provided | A list of file names to manage (allows batch operations). |
| `import` | Yes, if `name` or `names` is not provided | Path to import file/template definitions from another file (relative to blueprint directory) |
| `profiles` | No | List of profiles this file/template belongs to. If empty, file is always processed (base item). |
| `action` | Yes | The action to perform on the file or template. Valid values are `copy`, `move`, `delete`, `create`, `chmod`, `chown`, `chgrp`, and `symlink`. |
| `source` | No | The source path or URL of the file or template. Required for `copy`, `move` and `symlink`. Can be a local path or a URL. |
| `target` | Yes | Where the file goes. See [Target paths](#target-paths). |
| `content` | No | The content of the file. Setting it forces the `create` action. |
| `owner` | No | The owner of the file. Applied by `chown`, and after a `create`. |
| `group` | No | The group of the file. Applied by `chown`/`chgrp`, and after a `create`. |
| `mode` | No | The permission mode. Applied by `create` and `chmod` only. See [File modes](#file-modes). |
| `elevated` | No | Only the `copy` action reads this. With `true`, the copy is staged through a temporary file and installed with elevation. Every other file action is performed by rwr's own process and ignores the field. Defaults to `false`. |
| `variables` | No | A map of variables and their values to be used for template rendering. Only applicable to the `templates` section. |
| `interactive` | No | Accepted by the schema but **not read** by the files processor: file operations run the same way whatever it is set to. Only the global `--interactive` flag has any effect here. |

## Blueprint Imports

Import file and template definitions from other blueprint files:

```yaml
files:
  # Import common dotfiles
  - import: ../../Common/files/dotfiles.yaml

  # Add system-specific files
  - name: local-config.ini
    action: create
    target: /etc/myapp/
    content: |
      [settings]
      local=true

directories:
  # Import shared directory structure
  - import: ../shared/directories.yaml

templates:
  # Import common templates
  - import: ../../Common/templates/configs.yaml
```

Import features work for files, directories, and templates within the Files Blueprint.

## File Processing

The `files` section manages regular files. The processor dispatches on `action`:

| Action | What it does |
|--------|--------------|
| `copy` | Copies `<blueprint_dir>/<source>/<name>` to the target |
| `move` | Renames the source to the target. A move already carried out is not an error |
| `delete` | Removes the target. A target already absent is not an error |
| `create` | Writes `content` to the target, creating parent directories as needed |
| `chmod` | Applies `mode` to the target. `mode` is required |
| `chown` | Applies `owner` and/or `group` to the target |
| `chgrp` | Applies `group` to the target |
| `symlink` | Makes the target a symlink to the source. An existing correct link is left alone. An existing wrong link is replaced. An existing regular file or directory is an error |

There is no `append` action and no `template` action. Templates are a separate
section, described below.

Setting `content` forces the action to `create`. If the entry declared something
else, RWR logs a warning and creates the file anyway.

### URL Sources

If `source` is a URL, RWR downloads it to a temporary directory first and then
performs the requested action on the downloaded file.

### Target paths

`target` is resolved to the single path the action operates on:

- A target ending in `/` names the **directory** the file goes into: the entry's
  `name` is appended to it. `target: /etc/myapp/` with `name: app.conf` writes
  `/etc/myapp/app.conf`.
- Any other target is the **file's own path**, which is how you rename on the
  way: `target: /etc/myapp/production.conf`.
- If the target is an existing directory, RWR appends the source basename even without a trailing separator. RWR does not truncate the directory.

`~` is expanded. Because expansion drops a trailing separator internally, the
separator is read from the value you wrote, so `target: "~/.config/"` still
behaves as a directory.

Writes reject a final target that is a symlink or, on Windows, a reparse point.
RWR also applies copied-file permissions through the already-open destination,
so replacing the path during a copy cannot redirect the permission change.

## File modes

`mode` sets permissions with at most four octal digits, up to `0o7777`. Only `create` and `chmod` apply it. A copy uses the source permissions. A symlink has no permission mode. `delete`, `chown`, and `chgrp` do not change the mode. `rwr validate` warns when an action ignores a declared mode.

### How to write one

Write the mode as a quoted octal string. `mode: "0644"` means the same thing in YAML, JSON, TOML, and CUE. This avoids differences in numeric parsing. The `0` and `0o` prefixes are optional: `"0644"`, `"0o644"`, and `"644"` specify the same mode.

A number is read as **the mode's own value**, which is what every parser already
produces for an octal literal. YAML `0644`, TOML `0o644` and JSON `420` are one
and the same mode, `0o644`.

A bare integer that reads like octal digits typed without quotes is **refused**:

```yaml
mode: 644     # ERROR
```

```text
ambiguous file mode 644: as a number it is 0o1204, but it reads like the octal
mode 0644 typed without quotes; write mode: "0644" for 0o644, or mode: "1204"
for 0o1204
```

Anything above `0o7777`, or a negative number, is refused the same way.

### TOML and setuid/setgid/sticky modes

TOML's decoder returns the decoded number without the original literal. Both `0o4755` and `2541` arrive as the same number, so the ambiguity check cannot distinguish them. In TOML, a mode with a setuid, setgid, or sticky bit must be a quoted string:

```toml
mode = "04755"   # correct
mode = 0o4755    # ERROR: ambiguous file mode 2541
```

Plain permission modes at or below `0o777` are unaffected: `mode = 0o644` is
fine in TOML.

### Defaults

When no `mode` is declared:

| Entry | Mode |
|-------|------|
| A rendered template | `0600` |
| A plain file (`create`) | `0644` |
| A directory | `0755` |

Templates can render credentials into files such as `.netrc` or a `gh` configuration. These files must never be world-readable, even during creation.

If a created file's mode grants no group or other permissions, RWR creates any missing parent directories with mode `0700`, not `0755`.

A `chmod` action with no `mode` is an error at validation time and at run time. It is never treated as `0000`.

## Template Processing

The `templates` section of the blueprint renders template files. The template is
read from `<blueprint_dir>/<source>/<name>`, rendered with the merged variable
set, and written to `target`.

A template entry requires `name`, `source`, and `target`. If any field is missing, RWR skips the entry with a warning. RWR always creates the rendered content at the target, regardless of `action`. An entry with `action: copy` still renders and creates the file, and RWR logs a warning.

Template files use the Go template syntax and can include variables, conditionals, and loops. The `variables` setting allows you to define a map of variables and their corresponding values, which can be used within the template files.

> [!NOTE]
> The `variables` setting is only applicable to the `templates` section and will not be used for regular files defined in the `files` section.

For more information on using variables and templating in RWR, please refer to the [Variables and Templating](../variables.md) documentation.

## Examples

Here are some examples of using the Files Blueprint for both regular files and templates in YAML, JSON, and TOML formats:

### Regular Files

#### YAML

```yaml
files:
  # Base files - always processed (no profiles field)
  - name: global-config.ini
    action: copy
    source: ./config/
    target: /etc/myapp/
    elevated: true

  - names:
      - common.sh
      - utils.sh
    action: copy
    source: ./scripts/
    target: /usr/local/bin/
    elevated: true

  # A copy takes the source's permissions; set them with a chmod entry
  - name: common.sh
    action: chmod
    target: /usr/local/bin/common.sh
    mode: "0755"

  # Development profile files
  - name: dev-config.json
    profiles:
      - dev
    action: create
    target: /home/user/.config/myapp/
    content: |
      {
        "debug": true,
        "log_level": "debug"
      }

  - names:
      - dev-script.sh
      - debug-tools.sh
    profiles:
      - dev
    action: copy
    source: ./dev-scripts/
    target: /usr/local/bin/
    elevated: true

  # Production profile files
  - name: production.conf
    profiles:
      - production
    action: copy
    source: https://config.example.com/prod.conf
    target: /etc/myapp/production.conf
    elevated: true

  - name: ssl-cert.pem
    profiles:
      - production
    action: copy
    source: ./certs/
    target: /etc/ssl/certs/
    elevated: true

  # A copy does not apply owner or group either; use a chown entry
  - name: ssl-cert.pem
    profiles:
      - production
    action: chown
    target: /etc/ssl/certs/ssl-cert.pem
    owner: root
    group: root
```

#### JSON

```json
{
  "files": [
    {
      "name": "global-config.ini",
      "action": "copy",
      "source": "./config/",
      "target": "/etc/myapp/",
      "elevated": true
    },
    {
      "names": ["common.sh", "utils.sh"],
      "action": "copy",
      "source": "./scripts/",
      "target": "/usr/local/bin/",
      "elevated": true
    },
    {
      "name": "dev-config.json",
      "profiles": ["dev"],
      "action": "create",
      "target": "/home/user/.config/myapp/",
      "content": "{\n  \"debug\": true,\n  \"log_level\": \"debug\"\n}"
    },
    {
      "names": ["dev-script.sh", "debug-tools.sh"],
      "profiles": ["dev"],
      "action": "copy",
      "source": "./dev-scripts/",
      "target": "/usr/local/bin/",
      "elevated": true
    },
    {
      "name": "production.conf",
      "profiles": ["production"],
      "action": "copy",
      "source": "https://config.example.com/prod.conf",
      "target": "/etc/myapp/production.conf",
      "elevated": true
    },
    {
      "name": "ssl-cert.pem",
      "profiles": ["production"],
      "action": "copy",
      "source": "./certs/",
      "target": "/etc/ssl/certs/",
      "elevated": true
    }
  ]
}
```

#### TOML

```toml
# Base files - always processed (no profiles field)
[[files]]
name = "global-config.ini"
action = "copy"
source = "./config/"
target = "/etc/myapp/"
elevated = true

[[files]]
names = ["common.sh", "utils.sh"]
action = "copy"
source = "./scripts/"
target = "/usr/local/bin/"
elevated = true

# Development profile files
[[files]]
name = "dev-config.json"
profiles = ["dev"]
action = "create"
target = "/home/user/.config/myapp/"
content = """
{
  "debug": true,
  "log_level": "debug"
}
"""

[[files]]
names = ["dev-script.sh", "debug-tools.sh"]
profiles = ["dev"]
action = "copy"
source = "./dev-scripts/"
target = "/usr/local/bin/"
elevated = true

# Production profile files
[[files]]
name = "production.conf"
profiles = ["production"]
action = "copy"
source = "https://config.example.com/prod.conf"
target = "/etc/myapp/production.conf"
elevated = true

[[files]]
name = "ssl-cert.pem"
profiles = ["production"]
action = "copy"
source = "./certs/"
target = "/etc/ssl/certs/"
elevated = true
```

### Templates

#### Templates YAML

```yaml
templates:
  # Base template - always processed (no profiles field)
  - name: app.conf
    action: create
    source: ./templates/
    target: /etc/myapp/app.conf
    owner: root
    group: root
    mode: "0644"
    elevated: true
    variables:
      app_name: MyApplication
      log_level: info

  # Development profile template
  - name: nginx-dev.conf
    profiles:
      - dev
    action: create
    source: ./templates/
    target: /etc/nginx/sites-available/dev.conf
    owner: root
    group: root
    mode: "0644"
    elevated: true
    variables:
      server_name: dev.example.com
      port: 3000
      debug: true

  # Production profile template
  - name: nginx-prod.conf
    profiles:
      - production
    action: create
    source: ./templates/
    target: /etc/nginx/sites-available/prod.conf
    owner: root
    group: root
    mode: "0644"
    elevated: true
    variables:
      server_name: example.com
      port: 80
      ssl_enabled: true
```

#### Templates JSON

```json
{
  "templates": [
    {
      "name": "nginx.conf",
      "action": "create",
      "source": "/path/to/templates/nginx.conf.tmpl",
      "target": "/etc/nginx/nginx.conf",
      "owner": "root",
      "group": "root",
      "mode": "0644",
      "variables": {
        "server_name": "example.com",
        "port": 80
      }
    }
  ]
}
```

#### Templates TOML

```toml
[[templates]]
name = "nginx.conf"
action = "create"
source = "/path/to/templates/nginx.conf.tmpl"
target = "/etc/nginx/nginx.conf"
owner = "root"
group = "root"
mode = "0644"
[templates.variables]
server_name = "example.com"
port = 80
```

## Notes

- The `chmod`, `chown`, and `chgrp` actions are carried out by rwr's own process, not by a helper. Changing a file you do not own therefore needs rwr itself to be running with the privilege to do it.
- The `create` action creates missing parent directories.
- Write `mode` as a quoted octal string, `mode: "0644"`. A bare `mode: 644` is an error. See [File modes](#file-modes).
- When using URL sources, RWR will download the file to a temporary location before performing the requested action.
- Every action is safe to rerun. Deleting an absent file succeeds without changes. So does repeating a completed move or creating a symlink that already points to the source.

For more information on using the Files Blueprint in your RWR configuration, please refer to the [Blueprints Overview](../blueprints-general.md) and the [Best Practices](../best-practices.md) guide.
