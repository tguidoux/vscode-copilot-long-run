# Contributing to Copilot Long Run

Thank you for your interest in contributing! This document covers the architecture, the real-world challenges behind the extension, and development setup.

## Architecture Overview

```
SessionWatcher ─── (pause detected) ──> ContinueEngine ──> open session editor
     ^   (fs.watch + poll                    |                + chat.submit
     |    + baseline)                         v
     |                                    Guardrails (enabled + loop cap)
     +-- ActiveSessionResolver                |
     |   (reads state.vscdb via sqlite3)      v
     +───────── StatusBar (read-only view) ───+
```

### Component Responsibilities

| Component | File | Purpose |
|---|---|---|
| **SessionWatcher** | `src/sessionWatcher.ts` | Watches chat session JSONL files (workspace and empty-window storage) via a VS Code watcher plus a native `fs.watch` and a polling backstop. Baselines existing sessions on startup, then classifies the latest request's result and fires a pause event (turn ended or continue button) or a resume event (newer request in flight). |
| **ActiveSessionResolver** | `src/activeSessionResolver.ts` | Reads `state.vscdb` (SQLite) to determine which chat session is currently active. Used as a best-effort gate: the continue is skipped only when a positively-different session is active. |
| **ContinueEngine** | `src/continueEngine.ts` | Continues paused sessions indefinitely (no cooldown, no attempt cap). Opens the target session's editor to route the submit, restores the user's previous focus, and queues concurrent pauses (FIFO, deduped by session). |
| **Guardrails** | `src/guardrails.ts` | Light safety layer: an enabled check plus a self-clearing loop-protection cap (30 continues/minute) so a pathological tight loop can't run away. No cooldown, no permanent cap. |
| **StatusBar** | `src/statusBar.ts` | Read-only status bar item showing the current engine state (idle, waiting, continuing, disabled) and the queued-session count. |
| **Logger** | `src/logger.ts` | Structured logging to a dedicated "Copilot Long Run" output channel. |
| **Configuration** | `src/configuration.ts` | Live-reading configuration wrapper over VS Code settings. |

### Data Flow

1. **Detection**: SessionWatcher rebuilds the final `requests[]` state from the JSONL entries and inspects the last request's `result`.
2. **Classification**: The result is a continue opportunity when it completed with no error (turn ended), or when it carries a `copilotContinueOnError` confirmation button. `canceled` results and errors without a continue button are ignored. A freshness gate rejects stale completions; a per-session signature (including the turn's finish time) dedupes.
3. **Trigger**: A `ContinueTrigger` (session id + reason) is created. If a continue is already running, it's queued (deduped by session).
4. **Guardrail check**: ContinueEngine confirms the extension is enabled and the loop cap isn't momentarily hit (otherwise it re-queues and retries shortly).
5. **Schedule**: A short fixed delay (with light jitter) is applied before submitting.
6. **Target**: The engine opens the paused session's editor URI (`vscode-chat-session://local/<b64url(id)>`) so the submit lands in the right conversation; it skips only if `state.vscdb` reports a positively-different active session.
7. **Submit**: The continue message is sent via `workbench.action.chat.submit`, then the user's previous tab is restored.
8. **Repeat / resume**: The queue is drained for other paused sessions; when the agent starts its next turn the watcher fires a resume (cancelling any active continue), and the next turn's completion triggers the next continue.

## Challenges and How We Overcame Them

### Challenge 1: No Public API for Chat Session State

**Problem**: VS Code has no public extension API to observe whether a Copilot chat conversation has paused or ended. `ChatResponseTurn.result` is only available inside a chat participant's own response handler — not to third-party observers.

**Solution**: VS Code Core writes chat sessions to disk as JSONL files at `workspaceStorage/<hash>/chatSessions/<sessionId>.jsonl`. These files contain the full conversation state, including each request's `result`. The SessionWatcher monitors these files and classifies the latest result.

### Challenge 2: Understanding the JSONL Format

**Problem**: The JSONL session files use three kinds of entries that are not documented:

| Kind | Name | Meaning |
|---|---|---|
| `kind: 0` | Snapshot | Full conversation state — contains a `requests[]` array |
| `kind: 1` | Key-path patch | Updates a single nested property — e.g., `["requests", 0, "result"]` |
| `kind: 2` | Key replacement | Replaces an entire top-level key — e.g., the whole `requests` array |

**Solution**: We reverse-engineered the format from real session files. A completed request writes a `result` into `requests[N].result`. A clean turn-ended result has `metadata` and no `errorDetails`. A result that stopped and offers a continue carries `errorDetails.confirmationButtons` with `data.copilotContinueOnError: true`.

### Challenge 3: No Way to Press the Continue Button Programmatically

**Problem**: The built-in continue / "Try Again" button is proposed-only (`chatParticipantPrivate`), restricted to first-party extensions. `workbench.action.chat.retry` requires an internal view-model object and no-ops without it.

**Solution**: We use `workbench.action.chat.submit` with an `inputValue`. This submits the continue message in the current conversation. It costs one extra turn but preserves full context and is the only approach that works from a third-party extension.

### Challenge 4: Multi-Session Targeting

**Problem**: A single VS Code window can have multiple chat sessions. `workbench.action.chat.submit` always submits to the currently active (visible) chat widget. If the user switched sessions after a pause, the continue message would go to the wrong conversation.

**Solution**: We read the active session ID from `state.vscdb` — a SQLite database at `workspaceStorage/<hash>/state.vscdb`. The key `memento/interactive-session-view-copilot` contains JSON with a `sessionId` field matching the JSONL filename. We query it via the `sqlite3` CLI before each attempt. If the active session doesn't match the paused session, the continue is skipped.

This degrades gracefully: if `sqlite3` is unavailable, the database is locked, or the key is missing, we proceed (assume the correct session is active).

## Key Design Decision: Why "Chat Submit" Instead of the Continue Button

There is no public API for a third-party extension to trigger VS Code's internal continue button (see Challenge 3). The `chat.submit` approach is the only viable method.

**Trade-offs**:
- Sends a new turn in the conversation (costs one extra message)
- The agent sees its own paused state in history, plus our continue prompt
- Preserves full conversation context
- Works reliably from any third-party extension
- Does not depend on proposed or internal APIs

## Safety Guardrails

Continuation is indefinite by design — there is no cooldown and no attempt cap. The `Guardrails` class keeps only the minimum needed to stay safe:

| Guardrail | Value | Purpose |
|---|---|---|
| Extension enabled check | Must be enabled in settings | User kill switch |
| Loop-protection cap | 30 continues per rolling 60-second window (self-clearing) | Prevents a pathological tight loop; never blocks permanently |
| Active-session check | Skips only if a *different* session is positively active | Avoids continuing the wrong conversation |

The delay before a continue is a fixed base delay with light jitter (`baseDelayMs * jitter(0.85..1.15)`) — no exponential escalation.

## Known Limitations

1. **Session file format is undocumented** — The JSONL format (`kind: 0/1/2`) was reverse-engineered. VS Code could change it without notice. The extension degrades gracefully if parsing fails (no crash, just no detection).
2. **Submitting requires focusing the target session** — `workbench.action.chat.submit` goes to the focused chat widget, so the engine opens the target session's editor before submitting and restores the previous tab afterward. A continue firing while you type in another chat editor can cause a brief focus flicker.
3. **`sqlite3` dependency for the active-session check** — Used to read `state.vscdb`. Pre-installed on macOS and most Linux distros; may be missing on Windows or minimal containers. Without it, the check is skipped and the continue proceeds after focusing the target session.
4. **Continuing adds a conversation turn** — Unlike the native continue button, submitting a message costs one additional LM turn.
5. **Continues run one at a time** — Concurrent pauses are queued and handled sequentially (never dropped).

## Development Setup

### Prerequisites

- Node.js 18+
- VS Code 1.90+

### Build

```bash
cd vscode-copilot-long-run
npm install
npm run compile
```

### Watch Mode

```bash
npm run watch
```

### Run Tests

```bash
npm test
```

Tests use vitest. The `vscode` module is mocked (see `src/__mocks__/vscode.ts`) so tests run without the VS Code extension host.

### Package as VSIX

```bash
npm run package
```

This produces a `.vsix` file you can install via `code --install-extension <file>.vsix`. To compile, package, and install in one step:

```bash
npm run install-local
```

(The `package` script pins `@vscode/vsce@3.2.1`; the latest version was blocked by a supply-chain cooldown.)

### Testing Locally

1. Open the extension folder in VS Code
2. Press `F5` to launch the Extension Development Host
3. Use the command palette commands:
   - **Copilot Long Run: Show Status** to verify the extension is running
   - **Copilot Long Run: [Dev] Simulate Pause** to test the full continue pipeline end-to-end

### Project Structure

```
src/
  extension.ts                Entry point, wiring, command registration
  sessionWatcher.ts           Pause detection via chat session JSONL files
  activeSessionResolver.ts    Active session verification via state.vscdb (sqlite3)
  continueEngine.ts           Continues sessions indefinitely; session targeting + queue
  guardrails.ts               Enabled check + loop-protection cap
  statusBar.ts                Status bar UI
  logger.ts                   Output channel logging (with verbose mode)
  configuration.ts            Settings wrapper
  sessionWatcher.spec.ts      Tests: JSONL parsing and pause classification
  activeSessionResolver.spec.ts  Tests: sqlite3 integration and session matching
  guardrails.spec.ts          Tests: guardrail logic
  continueEngine.spec.ts      Tests: trigger factories + session URI builder
  __mocks__/
    vscode.ts                 Minimal vscode module mock for tests
```

## Code Conventions

- TypeScript strict mode
- No default exports
- Descriptive variable names (no abbreviations)
- All continue attempts must pass through `Guardrails.canContinue()` before execution
- All disposable resources must be registered in `context.subscriptions`
- Detection must be passive (no interception of Copilot's request pipeline)
- Zero runtime npm dependencies — only `vscode` (provided by the host) and dev dependencies

## Future Considerations

- If VS Code exposes an API to programmatically trigger confirmation buttons or continue chat responses, the continue mechanism should switch to it instead of the "chat submit" approach
- The `chatParticipantPrivate` proposed API may eventually become stable and accessible to third-party extensions
- If VS Code adds a public API for reading chat session state (active session, turn state), the filesystem approach can be replaced
