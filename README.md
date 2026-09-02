# Copilot Long Run

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Keeps GitHub Copilot agent sessions going.** When the agent pauses — its turn ends, or it stops to ask whether it should keep iterating — this extension automatically sends a continue message so a long-running task finishes while you're away.

---

## The Problem

You kick off a long task in Copilot agent mode, switch to something else, and expect it to be done when you come back. Instead, the agent completed a chunk of work and then just... stopped, waiting for you to tell it to keep going.

Now you have to context-switch back, type "continue", wait for the next chunk, and babysit it to completion — over and over — wasting the time you thought you were saving.

This extension fixes that. It detects when the agent has paused and **automatically nudges it forward** with a directive continue message.

## Features

- **Automatic continue on pause** — Monitors chat session files on disk and detects when the agent's turn has ended or it's presenting a "continue" button, then sends a continue message
- **Indefinite** — Keeps a session going for as many turns as it takes; there is no attempt cap and no cooldown
- **Directive continue message** — Sends "Keep going until the task is fully complete." by default (configurable)
- **Targets the right conversation** — Opens the paused session's own editor before submitting, then restores your previous tab, so the message lands in the correct session
- **Multi-session safe** — Concurrent pauses are queued and handled one at a time; a queued session is never dropped
- **Multi-window safe** — Each VS Code window only monitors its own sessions; continues never leak across windows
- **Loop protection** — A generous, self-clearing rate cap prevents a runaway tight loop, plus a kill switch (enable/disable)
- **Status bar indicator** — Shows current state at a glance: idle, waiting, continuing, or disabled
- **Manual continue command** — Trigger a continue immediately with `Cmd+Shift+R` / `Ctrl+Shift+R`
- **Verbose logging** — Optional detailed diagnostics in the output channel
- **Zero configuration** — Works out of the box with sensible defaults

## How It Works

1. VS Code writes Copilot chat sessions as JSONL files to disk. The extension watches these files (for both folder windows and empty windows) for the latest request's result.
2. A session is considered **paused** (a continue opportunity) when either:
   - The latest request completed with no error — the agent's turn ended and it's idle; or
   - The latest result carries a continue / "Try Again" button — the agent is explicitly asking whether to keep going.
   
   User cancellations (pressing Stop) are never auto-continued.
3. When a pause is detected, the extension waits a short delay, then opens the paused session's editor and submits the continue message into it.
4. Your previously focused tab is restored afterward, so the focus change is brief.
5. When the agent finishes its next turn, the cycle repeats — indefinitely — until the task is done or you stop it.

## Install

This extension isn't published to a marketplace; install it from source.

```bash
npm install
npm run install-local
```

`install-local` compiles, packages a `.vsix`, and installs it into VS Code (`--force` overwrites any existing install). Reload or restart VS Code afterward. To uninstall: `code --uninstall-extension TheoGuidoux.vscode-copilot-long-run`.

## Commands

| Command | Description |
|---|---|
| **Copilot Long Run: Enable** | Enable the extension |
| **Copilot Long Run: Disable** | Disable the extension and cancel active continues |
| **Copilot Long Run: Toggle Enabled** | Toggle on/off (also available by clicking the status bar item) |
| **Copilot Long Run: Continue Now** | Manually send a continue immediately (`Cmd+Shift+R` / `Ctrl+Shift+R`) |
| **Copilot Long Run: Show Status** | Display detailed status in the output channel |

## Settings

All settings are under `copilotLongRun.*` in VS Code Settings.

| Setting | Default | Description |
|---|---|---|
| `copilotLongRun.enabled` | `true` | Enable automatically continuing paused agent sessions |
| `copilotLongRun.continueMessage` | `Keep going until the task is fully complete.` | The message sent to the agent when a pause is detected |
| `copilotLongRun.baseDelayMs` | `2000` | Delay before sending a continue after a pause is detected, in milliseconds (500–15,000) |
| `copilotLongRun.verboseLogging` | `false` | Write detailed diagnostics at info level so they appear in the output channel without changing the log level |

## What Counts as a Pause?

The extension reads chat session JSONL files and classifies the latest request's result:

| Result | Continues? |
|---|---|
| **Turn ended** (completed, no error) | Yes |
| **Continue / "Try Again" button** (e.g. `rateLimited`, `networkError`) | Yes |
| **User cancellation** (`canceled`) | No (user-initiated) |
| **Error without a continue button** | No (nothing to nudge) |

## Status Bar

The status bar item (right side) shows the current state (a `(+N)` suffix means N sessions are queued):

| Icon | State | Meaning |
|---|---|---|
| $(check) Long Run | **Idle** | Monitoring normally, no pause detected |
| $(clock) Continue | **Waiting** | Short delay before sending the continue |
| $(sync~spin) Continuing | **Continuing** | Sending a continue right now |
| $(x) Long Run: Off | **Disabled** | Extension is disabled |

Click the status bar item to toggle the extension on/off.

## Limitations

- **Continuing adds a message to the conversation** — There is no public API to press the built-in continue button, so the extension submits a follow-up message instead, which adds one extra turn.
- **Submitting requires focusing the target session** — `workbench.action.chat.submit` always goes to the focused chat widget. To route to the right session the extension briefly opens that session's editor, then restores your previous tab. If a continue fires while you're actively typing in another chat editor, you may see a brief focus flicker.
- **Relies on VS Code internal file layout** — Session files are stored under `workspaceStorage/<hash>/chatSessions/` (folder windows) and `globalStorage/emptyWindowChatSessions/` (empty windows). If VS Code changes this layout, detection stops but nothing breaks — it degrades gracefully.
- **Active-session check requires the `sqlite3` CLI** — On macOS and most Linux systems it's pre-installed. Where it's missing, the check is skipped and the continue proceeds after focusing the target session.
- **Continues run one at a time** — Concurrent pauses are queued and handled sequentially.

## Requirements

- VS Code 1.90 or later
- GitHub Copilot extension installed

## Privacy

This extension:

- **Reads only result metadata** from chat session files (result/error codes and button data) — it does not read your prompts, responses, or conversation content
- **Does not send** any data to external services (beyond VS Code's built-in commands)
- **Does not wrap** or intercept Copilot's request pipeline

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for architecture details, design decisions, and development setup.

## Credits

Based on [vscode-copilot-auto-retry](https://github.com/Maxim-Mazurok/vscode-copilot-auto-retry) by [Max Mazurok](https://github.com/Maxim-Mazurok), reworked from automatic error retry into keeping long-running agent sessions going.

## License

[MIT](LICENSE)
