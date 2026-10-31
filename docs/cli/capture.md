# rwr capture

Capture creates a blueprint tree from an existing machine. It scans explicitly installed packages with manager-specific commands such as `pacman -Qe`, `brew leaves`, and `cargo install --list`. These scans exclude dependencies.

Capture also scans dotfiles, `~/.config` entries, enabled services, and Git checkouts. It excludes known cache and state files. A form for each category lets you choose what to keep.

```bash
# Interactive: one selection page per category
rwr capture ~/git/me/rwr-blueprints

# Take the defaults without the form (CI/scripting)
rwr capture --all out/

# Choose the format and add a root manifest matched to this machine
rwr capture --format cue --manifest out/
```

Defaults: explicit packages and known dotfiles pre-selected, services off
(most enabled units are distro plumbing), git checkouts on. Secret-shaped
paths (ssh, gnupg, netrc, credential dirs) are never pre-selected and warn
if chosen. Selected configs are copied under `files/src/` with entries
targeting their origin via `{{ .User.home }}`.

The generated tree is validated before capture reports success - output
that needs hand-repair is a failure, not a capture. From there:
`rwr all --init-file <dir>` provisions the next machine.
