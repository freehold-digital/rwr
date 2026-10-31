# Terminal interactions

The dashboard owns the terminal throughout a run. All operator interaction goes
through `internal/reporting`, so processors and credential providers share the
same popup behavior, cancellation handling, and headless fallback.

| Interaction | API |
| --- | --- |
| Choices, text entry, masked credentials, confirmations | `reporting.RunForm(form)` |
| Instructions while an asynchronous task runs, such as OAuth | `reporting.RunFormWithTask(form, task)` |
| Administrator password | `reporting.RequestSecretContext(ctx, prompt)` |
| File overwrite with diff and apply-to-all choices | `reporting.RequestConfirmationAllWithDetails(prompt, details)` |
| External tool with its own prompts, such as MFA | `reporting.RunInteractiveCommand(ctx, command, secrets, private)` |

Forms render inside the dashboard's modal frame and use its colors. Esc or Ctrl-C
cancels a form. A waiting task receives a canceled context and must stop its
network requests. Form submission never quits the parent dashboard.

When dialogs overlap, masked administrator entry takes priority over confirmations
and forms for both rendering and keyboard/paste input. Closing a dialog restores
any waiting interaction underneath it. OAuth completion cannot hide a pending
administrator password prompt.

A successful sudo readiness probe does not guarantee reusable authorization for later commands. A command can use a different session or popup TTY. Under the dashboard, sudo uses managed askpass instead of disabling password entry with `-n`.

If sudo needs a password, the masked popup appears. A valid ticket or password-free policy needs no extra prompt. After startup, the bridge binds access to the actual child session for both captured and interactive execution.

Command input stays separate from askpass authentication. Each run starts with fresh readiness state.

External tools run in a private pseudoterminal displayed inside a popup. Their output, cursor movement, choices, and password echo settings stay inside that pane. Keyboard input and paste go to the tool. Resizing updates the pane and its pseudoterminal.

Ctrl-C or Ctrl-Esc cancels the tool. Esc remains available to the tool's menus. Unix cancellation kills the child process group.

Unix commands retain a TTY on stdin when initial input is supplied. RWR queues initial lines with echo disabled before starting the child or popup input. It then restores normal terminal settings for the child to control.

Initial input must be newline-terminated text, no larger than 4 KiB, and fit intact in the terminal input buffer. Use non-interactive execution for other input.

Windows ConPTY commands currently require unsupplied stdin. Commands that provide stdin must use non-interactive execution on Windows.

List known credentials in `secrets` so output is scrubbed across write boundaries.
Set `private` for authentication commands: their output is displayed but never
sent to the run log. Password echo is controlled by the tool's pseudoterminal
settings. RWR's own credential fields always use masked input.

Canceling a choice returns an error, never its default selection. A dismissed
package-manager chooser cannot approve installation of the default manager.

In headless mode these APIs use the existing terminal interaction. During a
dashboard run, `WithTerminal` refuses raw handoffs. New code must not call
standalone form `Run`, `tea.Exec`, `tea.ExecProcess`, or read `os.Stdin` directly.
The architecture regression test guards terminal handoffs and standalone forms. The approved raw-input paths are headless fallbacks and pre-dashboard CLI setup.

Tests run real Bubble Tea event loops and local tool fixtures. They cover
selection, token validation and masking, OAuth cancellation, a tool requiring a
controlling TTY, private output, redaction, resize, and child cancellation.
