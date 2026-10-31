# Bootstrap Process

The Bootstrap Process in Rinse, Wash, Repeat (RWR) is responsible for setting up the initial system configuration. It ensures that the necessary prerequisites and dependencies are in place before executing the main blueprints. This page explains how the Bootstrap Process works and how to define the bootstrap file.

## Overview

The Bootstrap Process is executed before any other blueprints are processed. It typically includes tasks such as:

- Installing essential packages and tools
- Setting up package managers
- Creating required directories and files
- Configuring system settings and permissions
- Setting up SSH keys

The Bootstrap Process is defined in a separate blueprint file named `bootstrap.yaml` (or `bootstrap.json` or `bootstrap.toml`, depending on the chosen format).

Profile validation happens before bootstrap or package-manager installation.
Bootstrap runs before normal credential resolution, so it can install the tools
needed by credential sources. Preparation scripts must use OS tools and cannot
depend on credentials that have not been resolved yet. If bootstrap itself needs
a declared credential, explicitly give it `scope: [bootstrap]`. RWR resolves
those credentials before bootstrap begins. An empty package list
requires no package manager. Profile-aware bootstrap entries honor the selected profiles. Package-manager declarations apply regardless of profile.

## Bootstrap File Structure

The structure of the bootstrap file is similar to other blueprint files in RWR. It can include the following sections:

- `scripts`: Preparation scripts, run first using OS tools, before package managers or vault credentials.
- `packageManagers`: Installs package managers needed by the bootstrap packages.
- `packages`: Defines the packages to be installed during the bootstrap process.
- `files`: Specifies the files to be created or modified during the bootstrap process.
- `directories`: Defines the directories to be created during the bootstrap process.
- `git`: Specifies the Git repositories to be cloned during the bootstrap process.
- `services`: Defines the services to be managed (started, stopped, enabled, disabled) during the bootstrap process.
- `users`: Specifies the user accounts to be created during the bootstrap process.
- `groups`: Defines the groups to be created during the bootstrap process.
- `ssh_keys`: Specifies the SSH keys to be generated during the bootstrap process.

Example 1: Define bootstrap tasks in `bootstrap.yaml`.

```yaml
packages:
  - name: git
    action: install
  - name: curl
    action: install
  - names:
      - vim
      - tmux
    action: install
    package_manager: apt
    args: ["--no-install-recommends"]

files:
  - name: config.ini
    action: create
    content: |
      [settings]
      debug = true
  - name: remote-config.txt
    action: copy
    source: https://example.com/remote-config.txt
    target: /etc/app/config.txt

directories:
  - name: data
    action: create
    mode: 0755

git:
  - name: my-repo
    action: clone
    url: https://github.com/example/my-repo.git
    path: /opt/my-repo

services:
  - name: nginx
    action: enable

users:
  - name: johndoe
    action: create
    password: "$6$mysecretpassword"
    groups:
      - sudo
      - docker

groups:
  - name: developers
    action: create

ssh_keys:
  - name: id_rsa
    type: rsa
    path: ~/.ssh
    comment: johndoe@example.com
    no_passphrase: true
    copy_to_github: true
    set_as_rwr_ssh_key: true
```

## Execution Order

The Bootstrap Process executes the sections in the following order:

1. `scripts`
2. `packageManagers`
3. `packages`
4. `directories`
5. `files`
6. `ssh_keys`
7. `git`
8. `services`
9. `users`
10. `groups`

This order ensures that the necessary dependencies and prerequisites are in place before proceeding with other tasks.

## Conditional Execution

By default, bootstrap runs once during initial setup. Later runs skip it unless you invoke bootstrap explicitly or select a profile missing from its recorded coverage. For an uncovered profile, RWR reruns bootstrap so the newly selected entries apply.

To force the execution of the Bootstrap Process on every run, you can use the `--force-bootstrap` flag:

```bash
rwr all --force-bootstrap
```

This flag will ensure that the Bootstrap Process is executed even if it has been run previously.

To re-run only the bootstrap - after editing `bootstrap.yaml`, say - use the
standalone command instead of re-running everything:

```bash
rwr bootstrap
```

An explicit invocation ignores the run-once marker and refreshes it on success. The marker keeps `rwr all` idempotent. It does not block a direct request.

RWR stores bootstrap coverage in a marker file in its configuration directory. Later runs repeat bootstrap when the marker is absent or does not cover a selected profile.

A marker that cannot be parsed counts as covering base entries only. This includes empty markers from older RWR versions. A run without a profile still skips bootstrap. Any `--profile` request reruns it.

`--profile all` records full coverage. Runs naming specific profiles add them to the recorded coverage. Switching between covered profiles does not repeat bootstrap. `--dry-run` does not write the marker, so a real run afterward still performs bootstrap.

A bootstrap file can declare `schema_version` to override the tree-wide version in the init file.

## New Features and Enhancements

### Package Management

- The `packages` section now supports the `names` field for installing multiple packages with the same configuration.
- Additional arguments can be specified for package installation using the `args` field.

### File Management

- The `files` section now supports URL sources. RWR will download files from the specified URL before processing them.
- If the target path does not end with `/`, RWR treats the file operation as a rename.

### SSH Key Management

- The `ssh_keys` section has been added to generate and manage SSH keys during the bootstrap process.
- The `set_as_rwr_ssh_key` option allows setting a generated key as the default RWR SSH key for operations.
- With `copy_to_github: true`, the optional `github_title` field names the key on GitHub. Without it, the machine's hostname is used.

## Best Practices

When defining your bootstrap file, consider the following best practices:

- Keep the bootstrap file minimal and only include the essential tasks required for the initial setup.
- Use variables and templating to make the bootstrap file more flexible and reusable across different environments.
- Use URL sources and target-path renaming for file operations.
- Use the `ssh_keys` section to set up necessary SSH keys, including the default RWR SSH key.
- Test the bootstrap file thoroughly to ensure it works as expected on the target systems.
- Document any manual steps or prerequisites that are not covered by the bootstrap file.

By following these best practices, you can create a reliable and maintainable bootstrap process for your RWR-managed systems.

For more information on specific blueprint types and their options, please refer to the respective documentation pages.

Successful preparation scripts are recorded individually under RWR's
`bootstrap-steps` configuration directory. Retries (including forced bootstrap)
skip an unchanged successful script, even when a later step failed. Changes to
a script declaration or its source content cause it to run again. Scripts must
converge safely after a failed or interrupted attempt, including interruption
before its completion marker is saved.
