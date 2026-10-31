# Omarchy configuration tool

Omarchy is a tool in the existing Configuration Blueprint. Declare it under
`configurations` with `tool: omarchy`, then run `rwr run configuration` or
`rwr all`. If the init file defines an explicit order, include `configuration`
after the files and packages the desktop setup needs.

This tool personalizes an installed Omarchy desktop after login. It never
installs or updates Omarchy itself, runs as the desktop user without sudo, and
requires working Hyprland and Omarchy shell IPC for apply.

```yaml
configurations:
  - name: desktop
    tool: omarchy
    action: set
    plugins:
      - id: expose.window-overview
        source:
          git: https://github.com/kristofferR/omarchy-expose.git
        enabled: true
        settings:
          hotCornerEnabled: false
      - id: io.github.woogy7.workspaces
        source:
          git: https://github.com/Woogy7/omarchy-workspace-switcher.git
        enabled: true
        settings:
          minWorkspaces: 5
          maxWorkspaces: 5
        widget:
          visible: false
      - id: omarchy.workspaces
        enabled: true
        widget:
          visible: true
          section: left
          after: omarchy.menu
    shell:
      idle: {screensaver: 300, lock: 360}
      bar: {position: top, transparent: false}
```

Omarchy configuration entries support `profiles` and `import`. Import paths and
asset sources resolve relative to the file declaring them, including nested
cross-format imports.

```yaml
configurations:
  - tool: omarchy
    import: ../shared/omarchy.json
    profiles: [desktop]
```

Selected entries are combined before applying. Conflicting values name both
source files. CUE, YAML, TOML and JSON have the same schema. Static validation
and dry-run work without a running desktop and perform no installation,
network mutation, readiness check, reload, backup or journal write.

## Plugins and configuration

`source` accepts exactly one of `git`, `path`, or `clone`. Git sources use HTTPS or SSH URLs. An optional `ref` pins a revision.

Already-satisfied floating checkouts are not fetched. Use `update: true` to refresh an unpinned Git source explicitly. A changed pin is a desired-state change. RWR stages all changed sources and checks their manifests before changing installed resources.

Plugin IDs come from manifests. A repository name is not an ID. Sources with a
different manifest identity are rejected. Local source directories must contain
regular files rather than symlinks. Omarchy discovers installed plugins under
`~/.config/omarchy/plugins/<id>`. An arbitrary installation directory is not a
supported option.

`source: {clone: omarchy.clock}` explicitly invokes Omarchy's native clone operation. The declared ID must match the user-derived `<username>.clock` ID. The native clone initially replaces its source. To turn it off, declare `enabled: false`.

Settings and integration operations never implicitly clone a built-in. Packaged plugins have no source and cannot use `state: absent`. To disable them, use `enabled: false`.

Omitting `enabled` preserves existing activation. `widget.visible` independently
controls bar presence, so an overlay can stay enabled without its optional
widget. Placement accepts `section: left|center|right` and at most one of
`before`, `after`, or zero-based `index`. Missing anchors and conflicting layouts
fail preflight. Built-in replacement operations use the native registry APIs.

`settings` merges declared keys and nested maps into the plugin's existing
entry. Unknown options survive. Empty arrays/maps, false, zero and null are
values. `unset: [nested.option]` explicitly removes a setting. Use nested maps,
not literal dots in option keys. Structural identity fields are reserved.

Shell options cover idle `screensaver` and `lock` durations, plus bar `position`, `transparent`, and `centerAnchor`. These updates target specific fields instead of replacing `shell.json`. Existing invalid JSON is an error. RWR seeds missing user configuration from installed defaults.

Writes are atomic, retain protected prior versions, and reject detected concurrent changes. RWR cannot lock out an external writer racing the final filesystem rename. Readback and later status checks expose changes that remain after application.

## Themes, defaults and hooks

```yaml
configurations:
  - name: appearance
    tool: omarchy
    action: set
    theme:
      name: my-theme
      source: {path: ../assets/my-theme}
      active: true
      background: ../assets/background.png
    defaults: {browser: firefox, terminal: ghostty, editor: nvim}
    hooks:
      - event: theme-set
        name: 20-personal
        source: ../assets/theme-hook.sh
```

A theme can select stock assets, install from Git, or copy a local directory.
An overlay with a stock theme's slug inherits its packaged files. Git themes
retain their `.git` metadata so native Omarchy theme activation applies its
executable-content restrictions. Theme assets and hooks precede activation. Active name/background are read back afterward.

Defaults require installed applications. The inspected Omarchy setters support these names:

- Browser: `chromium`, `chrome`, `brave`, `brave-origin`, `edge`, `firefox`, `zen`.
- Terminal: `alacritty`, `foot`, `ghostty`, `kitty`.
- Editor: `code`, `cursor`, `zed`, `sublime_text`, `helix`, `vim`, `emacs`, `nvim`.

Install applications through package declarations. The native terminal setter replaces `~/.config/xdg-terminals.list`. The browser setter updates XDG web handlers. Editor selection updates Omarchy's default editor. RWR does not invoke other defaults, including the agent installer. A notification failure does not undo a default whose saved value was verified.

Hooks support `theme-set`, `font-set`, `post-boot`, `post-update`, `battery-low`,
and `pre-refresh-pacman`. Each named hook gets an owned wrapper and a protected
copy of its source. Arguments are preserved. Unrelated hooks are untouched.
Declared theme-hook execution outcomes are checked because the native runner
can hide failures. Undeclared hooks retain Omarchy's native reporting behavior.

## External screensavers

```yaml
configurations:
  - name: external-screensaver
    tool: omarchy
    action: set
    integrations:
      screensaver:
        mode: external
        windowClass: org.omarchy.screensaver
        launch: {exec: /absolute/path/to/engine, args: [start]}
        check: {exec: /absolute/path/to/engine, args: [check]}
```

The stock `omarchy.idle` plugin must be enabled. If a custom idle clone is active,
explicitly restore stock idle in the same blueprint before selecting this route.
The engine is supplied separately. Its check must be nonvisual and return zero
only when ready. Launch/check executables are absolute paths and arguments stay
separate, including quotes and whitespace. Apply never starts a visible
screensaver or locks the desktop.

The stock idle service invokes `omarchy-launch-screensaver` through `bash -lc`. RWR installs a dispatcher under `~/.config/rwr/omarchy/bin`. It adds an identifiable PATH block at the end of the active Bash login profile: `.bash_profile`, `.bash_login`, or `.profile`. Before reporting activation, RWR checks launcher resolution in a fresh login shell.

Descendants of those login shells also receive the route. Existing processes retain their environment. If the route check fails, RWR restores the prior profile. RWR does not edit or clone the packaged launcher or idle service. `RWR_OMARCHY_STOCK_SCREENSAVER=1` selects the absolute packaged fallback.

The engine must use Hyprland window class `org.omarchy.screensaver`. It must create windows on the intended monitors, dismiss them on input, and stop when Omarchy locks. The stock idle service uses those window events to maintain or cancel its lock timer.

RWR checks the installed idle adapter's required source contract and the engine's readiness command. An exit code cannot prove an arbitrary engine's visual or locking behavior. Before relying on a new engine, test it in a disposable logged-in session. Check idle triggering, dismissal, multi-monitor behavior, subsequent locking, and stock restoration.

`mode: stock` explicitly removes RWR's login routing block and verifies that a
fresh login shell resolves the packaged launcher (including an equivalent symlink). Omission leaves routing alone.
Modified or unowned dispatchers/profile blocks are not overwritten. The owned,
dormant dispatcher is retained for later reactivation.

## Keybindings

Keep Lua and helper scripts in ordinary file blueprints. Example assets are in
`examples/omarchy/assets/`. Deploy `rwr-bindings.lua` and executable
`workspace-cycle` into the target user's `~/.config/hypr/`, then include this line
in the `bindings.lua` your blueprint already manages:

```lua
dofile(os.getenv("HOME") .. "/.config/hypr/rwr-bindings.lua")
```

This changes only Ctrl+Alt+Arrow bindings and leaves window-moving shortcuts unchanged. The helper reads Workspace Switcher's `maxWorkspaces`, including when its optional widget is in the bar. Missing settings default to 10. Invalid limits fail without dispatching. There is no second workspace-count setting.

Respect OmaSettings' later override load order. Check Lua with `luac -p`. After deployment, run `hyprctl reload`. Then inspect `hyprctl configerrors`. These are ordinary file and script declarations, not a second keybinding language.

## Ownership, status and recovery

Each resource has a stable identity such as `plugin/<id>/enabled`,
`plugin/<id>/settings/<path>`, `theme/active`, or `default/browser`. Status probes
actual state even without a prior journal record. Unavailable IPC is unknown.
Dry-run lists desired resources without pretending live state was observed.

Created plugin/theme assets and hooks have private ownership/content records
under `~/.local/state/rwr/omarchy`. `state: absent` removes only owned, unchanged
plugins/hooks. Adopted resources and dirty or modified checkouts are retained
with an error. Omitting a declaration never removes it. Ordinary journal entries
contain identities and outcomes, not plugin option maps or whole configuration
files. Previous configuration contents are stored separately with mode 0600.

Generic `rwr uninstall` lists configuration changes as not automatically
reversible. Use explicit desired-state changes to restore settings, stock
activation or routing. RWR does not guess old values or delete shared
configuration. Semantic capture and diff authoring remain follow-up work.

The adapter targets the installed Omarchy plugin catalog/list/validate,
enable/disable/clone/remove, theme and default command contracts, structured
`shell.json`, and Bash-launched stock idle service. Capability failures are
reported before dependent work. Unsupported environments get no deferred login
job or second shell process. Automated validation uses disposable files,
command fixtures and the launcher boundary. Live visual desktop acceptance is
separate from those tests.
