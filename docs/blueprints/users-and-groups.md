# Users and Groups Blueprint

The Users and Groups blueprint allows you to manage user accounts and groups on your system. You can create, modify, and remove users, assign them to groups, and set their properties such as password, shell, and home directory.

## Blueprint Structure

The Users and Groups blueprint has the following structure:

```yaml
users:
  - name: john
    action: create
    password: "$6$rounds=656000$saltsalt$hashhashhash"
    uid: "1500"
    comment: "John Doe"
    groups:
      - users
      - developers
    shell: /bin/bash
    home: /home/john

  - name: jane
    action: modify
    new_name: jane_smith
    new_shell: /bin/zsh
    new_home: /home/jane_smith
    add_groups:
      - designers
    remove_groups:
      - interns
    unlock: true

  - name: bob
    action: remove
    remove_home: true

groups:
  - name: developers
    action: create
    gid: "3000"

  - name: designers
    action: modify
    new_name: design_team
```

See [Fields Common to Every Blueprint](common-fields.md) for `profiles`,
`import`, `interactive`, and the rule that an unknown key is an error.

Ordinary user-management commands such as macOS `dscl` do not take over the terminal because the run uses the interactive TUI. RWR collects required sudo authentication through a masked dashboard prompt and keeps command output inside the dashboard. Headless runs fall back to RWR's masked terminal prompt. Set `interactive: true` on an entry only when its account-management command needs direct terminal interaction.

## Actions

`create`, `modify` and `remove` for both users and groups. `delete` is an
accepted alias for `remove`.

`create` is idempotent (safe to run more than once). If an account or group exists, RWR updates it to match the declared attributes instead of failing. It uses `usermod` on Linux and `dscl -create` on macOS. If a password is supplied, RWR sets it.

For an existing account, `home` declares the intended home without relocating the current one. To relocate it, use `modify` with `new_home`.

`remove` on an account or group that does not exist succeeds and does nothing.

## `users` settings

| Setting | Description |
|---------|-------------|
| `name` | The username (required if `import` is not provided) |
| `action` | `create`, `modify`, `remove` (or its alias `delete`) |
| `uid` | User ID to assign. It is a string: write `uid: "1500"` |
| `password` | See [Passwords](#passwords) |
| `groups` | Supplementary groups to put the user in (`create`) |
| `add_groups` | Groups to add the user to (`modify`) |
| `remove_groups` | Groups to remove the user from (`modify`) |
| `shell` | Login shell (`create`) |
| `new_shell` | New login shell (`modify`) |
| `home` | Home directory (`create`) |
| `new_home` | New home directory (`modify`) |
| `comment` | The GECOS/real-name field |
| `system` | Create as a system account |
| `expire` | Account expiration date, `YYYY-MM-DD` |
| `lock` | Lock the account (`modify`) |
| `unlock` | Unlock the account (`modify`) |
| `remove_home` | Remove the home directory too (`remove`) |
| `new_name` | Rename the account (`modify`) |
| `profiles`, `import`, `interactive` | See [common fields](common-fields.md) |

## `groups` settings

| Setting | Description |
|---------|-------------|
| `name` | The group name (required if `import` is not provided) |
| `action` | `create`, `modify`, `remove` (or its alias `delete`) |
| `gid` | Group ID to assign. It is a string: write `gid: "3000"` |
| `system` | Create as a system group |
| `new_name` | Rename the group (`modify`) |
| `profiles`, `import` | See [common fields](common-fields.md) |

Groups have no `interactive` field.

## Passwords

On Linux, RWR passes the password to `chpasswd` through standard input, never as a command argument. The password does not appear in `ps` or RWR's debug output. The value can be either:

- **Cleartext**, which `chpasswd` hashes itself, or
- **A crypt(3) hash** - `$6$…`, `$y$…`, a 13-character DES hash, or one of the
  locked markers `!`, `!!`, `*`, `*LK*` - which is passed through with
  `chpasswd -e`.

RWR detects which of the two you wrote. There is no field to declare it.

> [!WARNING]
> **On macOS**, the password is passed to `dscl . -passwd` as a command-line
> argument, because macOS computes its own salted blob and there is no hash to
> pre-compute. It is therefore briefly visible in `ps` to every local user on
> the machine, and lands in sudo's syslog record. RWR logs a warning each time
> it does this. Prefer setting macOS passwords out of band.

A `password` is applied by both `create` and `modify`.

## Supported Platforms

| Platform | Support |
|----------|---------|
| Linux | Full, via shadow-utils: `useradd`, `usermod`, `userdel`, `groupadd`, `groupmod`, `groupdel`, `gpasswd`, `chpasswd` |
| macOS | Full, via Open Directory: `sysadminctl` when present, otherwise `dscl`, plus `dseditgroup`, `createhomedir` and `pwpolicy` |
| Windows | Not implemented. Each entry logs a warning and is skipped |

### What differs on macOS

macOS is genuinely supported, but Open Directory has no equivalent for some of
the fields, and RWR warns rather than pretending it applied them:

| Field | On macOS |
|-------|----------|
| `system` | **Ignored, with a warning.** UIDs and GIDs below 501 are reserved by Apple and RWR does not allocate them |
| `expire` | **Ignored, with a warning.** A local Open Directory account has no expiration field |
| `new_home` | The `NFSHomeDirectory` record is rewritten, but the directory's **contents are not moved**. RWR warns when it does this |
| `remove_home` | Honoured through `sysadminctl -deleteUser` (which is also why RWR passes `-keepHome` when the flag is false). If `sysadminctl` is not available, the `dscl` fallback deletes the record only and warns that the home directory was left in place |
| `lock` / `unlock` | Applied with `pwpolicy -disableuser` / `-enableuser` |
| primary group | New accounts land in `staff` (GID 20). There is no field to choose another |
| `uid` / `gid` | Allocated from 501 upwards when not declared |

Other fields map directly to macOS account attributes:

- `shell` maps to `UserShell`.
- `comment` maps to `RealName`.
- `home` maps to `NFSHomeDirectory`.
- `groups`, `add_groups`, and `remove_groups` use `dseditgroup`.
- `new_name` changes `RecordName` after every other edit.

## Examples

Here are a few examples of using the Users and Groups blueprint in different formats:

### YAML

```yaml
users:
  - name: alice
    action: create
    password: "$6$secretpassword"
    groups:
      - users
      - admin
    shell: /bin/bash
    home: /home/alice

  - name: bob
    action: modify
    new_name: robert
    new_shell: /bin/zsh
    new_home: /home/robert
    add_groups:
      - developers

  - name: charlie
    action: remove
    remove_home: true

groups:
  - name: admin
    action: create

  - name: developers
    action: modify
    new_name: dev_team
```

### JSON

```json
{
  "users": [
    {
      "name": "david",
      "action": "create",
      "password": "$6$othersecretpassword",
      "groups": [
        "users",
        "staff"
      ],
      "shell": "/bin/zsh",
      "home": "/home/david"
    },
    {
      "name": "eve",
      "action": "modify",
      "new_name": "evelyn",
      "add_groups": [
        "managers"
      ]
    },
    {
      "name": "frank",
      "action": "remove"
    }
  ],
  "groups": [
    {
      "name": "staff",
      "action": "create"
    },
    {
      "name": "managers",
      "action": "modify",
      "new_name": "management"
    }
  ]
}
```

### TOML

```toml
[[users]]
name = "carol"
action = "create"
password = "$6$passwordhash"
groups = ["users", "managers"]
shell = "/bin/fish"
home = "/home/carol"

[[users]]
name = "carol"
action = "modify"
new_name = "carolyn"
new_shell = "/bin/zsh"
new_home = "/home/carolyn"
add_groups = ["designers"]

[[users]]
name = "grace"
action = "remove"
remove_home = true

[[groups]]
name = "managers"
action = "create"

[[groups]]
name = "designers"
action = "modify"
new_name = "design_team"
```

These YAML, JSON, and TOML examples define, modify, and remove users and groups.

## Blueprint Imports

Import user and group definitions from other files:

```yaml
users:
  # Import shared user accounts
  - import: ../../Common/users/base-users.yaml

  # Add environment-specific users
  - name: dev_user
    action: create
    password: "$6$secrethash"
    groups:
      - developers
    shell: /bin/bash
    profiles:
      - dev

groups:
  # Import shared groups
  - import: ../../Common/users/base-groups.yaml

  # Add local groups
  - name: local_admins
    action: create
```

This allows you to maintain common user and group configurations separately from environment-specific ones.

For more information on managing users and groups in RWR, please refer to the [Blueprints Overview](../blueprints-general.md) and the [Best Practices](../best-practices.md) sections of the documentation.
