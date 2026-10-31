# rwr diff

Compare what is actually on the machine against the blueprint tree, and get
the delta as blueprint material - never an apply.

The machine side uses read-only scans of explicitly installed packages, operator-enabled services, Git checkouts, and configurations. Package scans use provider-specific commands such as `pacman -Qe` and `apt-mark showmanual`. The tree side uses the resolved plan.

Diff excludes resources that the [run journal](../state.md) records as applied, even if the tree later changed. These resources were not added by hand. Packages and services match by name. Configurations and checkouts match by the recorded path, because entry names do not determine file locations.

```bash
# The readable list: + additions (hand-done), - removals (declared, gone)
rwr diff

# One category
rwr diff --packages

# Paste-ready blueprint blocks
rwr diff --packages --format cue

# Route additions into a tree interactively: per group, pick the machine
# file, a Common file the tree imports, or skip
rwr diff --into ~/git/thefynx/rwr-blueprints/Archcraft
```

`--into` requires a terminal. Use `--format` for non-interactive operation. RWR writes routed edits in the destination file's format. Run `rwr validate` before reviewing the changes. Commit them after review. RWR reports removals but never deletes blueprint entries automatically.
