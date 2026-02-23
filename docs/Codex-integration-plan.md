# Codex CLI Agent Integration Plan

This plan explains how to extend Pixel Agents to connect and visualize **Codex CLI** agents (alongside Claude Code), with incremental delivery and backward compatibility.

## 1) Current architecture: where Claude assumptions exist

Pixel Agents has three Claude-coupled areas:

1. **Launch and session discovery**
   - `src/agentManager.ts` launches `claude --session-id ...` and predicts transcript path in `~/.claude/projects/<workspace>/`.
2. **Transcript parsing and state mapping**
   - `src/transcriptParser.ts` parses Claude JSONL record shapes and infers agent states (active/waiting/permission).
3. **Session adoption / scan logic**
   - `src/fileWatcher.ts` scans the Claude projects directory and adopts new JSONL files.

The rendering side (webview messages such as `agentToolStart`, `agentStatus`) is already mostly provider-neutral.

---

## 2) Goals and non-goals

### Goals

- Let users create a new Pixel Agent backed by **Codex CLI**.
- Track Codex agent activity from machine-readable logs/events.
- Keep Claude behavior unchanged.
- Allow mixed-provider workspaces (Claude + Codex) in one panel.

### Non-goals (MVP)

- Exact parity for every Codex event subtype from day one.
- A large provider-management UI.

---

## 3) Integration model (recommended)

Implement provider abstraction first, then add Codex provider.

### 3.1 New provider contract

Create `src/providers/types.ts` with:

- `AgentProviderId = 'claude' | 'codex'`
- `ProviderLauncher` (build terminal command and session identifiers)
- `ProviderSessionLocator` (find session/log files)
- `ProviderParser` (convert raw provider event lines into normalized events)
- `NormalizedAgentEvent` union:
  - `tool_start`, `tool_done`
  - `status_active`, `status_waiting`, `status_permission_required`
  - `subagent_tool_start`, `subagent_tool_done`, `subagent_clear`
  - `turn_end`, `parse_warning`

### 3.2 Claude refactor (no behavior change)

- Move Claude launch/path logic to `src/providers/claude/provider.ts`.
- Move current parsing logic to `src/providers/claude/parser.ts`.
- Keep output message semantics exactly the same.

### 3.3 Codex provider implementation

Add:

- `src/providers/codex/provider.ts`
- `src/providers/codex/parser.ts`

Codex provider responsibilities:

- Launch command (`codex` by default, configurable).
- Start session in mode that emits machine-readable events/logs (configurable flags).
- Resolve session/log directory and file naming.
- Parse Codex events and emit normalized events.

> Because Codex CLI output contracts can evolve, keep parser tolerant and configuration-driven.

---

## 4) Codex CLI technical approach

Based on Codex CLI public README:

- Binary/command expected: `codex` (or configured override).
- Interactive mode exists by default; Pixel Agents should prefer a deterministic event/log source.

### 4.1 Prefer explicit event-output mode

Implement a settings-backed launch template so users can provide stable flags if needed:

- `pixel-agents.codex.command` (default `codex`)
- `pixel-agents.codex.args` (default empty array)
- `pixel-agents.codex.sessionArgTemplate` (optional template for session ID injection)
- `pixel-agents.codex.logPath` (optional explicit path)
- `pixel-agents.codex.enableAutoDiscover` (default true)

If Codex has JSON/JSONL mode in your chosen version, use that first. If unavailable, implement a conservative line parser with strict guards and diagnostics.

### 4.2 Session and file discovery strategy

Support both:

1. **Explicit path mode**: user provides `logPath`.
2. **Auto-discovery mode**: provider scans known Codex directories and matches by session ID, mtime, and workspace hints.

Adopt files only when confidence is high to avoid cross-project mis-assignment.

### 4.3 Event normalization examples

- Codex tool invocation event -> `tool_start`
- Codex tool completion event -> `tool_done`
- Codex turn-complete marker -> `turn_end` + `status_waiting`
- Codex approval/confirmation wait signal -> `status_permission_required`
- Unknown event -> `parse_warning` (no crash)

---

## 5) Data model and persistence updates

Update `src/types.ts`:

- `AgentState.providerId: AgentProviderId`
- `PersistedAgent.providerId?: AgentProviderId` (optional for migration)
- Optional provider metadata (`providerSessionId`, `providerHints`)

Migration rules:

- Missing `providerId` defaults to `'claude'`.
- Keep existing workspaceState keys to avoid breaking current users.

---

## 6) Extension/backend changes

### 6.1 `src/PixelAgentsViewProvider.ts`

- Add generalized `openAgent` message with payload `{ providerId }`.
- Keep temporary compatibility for old `openClaude` message.
- Dispatch launch through provider registry.

### 6.2 `src/agentManager.ts`

- Replace hardcoded `claude --session-id` with provider launcher outputs.
- Keep common lifecycle logic (timers, cleanup, terminal tracking).
- Store provider ID on each agent.

### 6.3 `src/fileWatcher.ts`

- Track known session files by `(providerId, filepath)`.
- Run provider-specific scanners.
- Let provider assert whether a discovered file is adoptable.

### 6.4 parsing path

- Replace direct `processTranscriptLine(...)` coupling with provider parser dispatch.
- Feed normalized events into existing webview message behavior.
- Keep timer heuristics as shared logic consuming normalized events.

---

## 7) Webview/UI changes (minimal but clear)

In `webview-ui/src/components/BottomToolbar.tsx`:

- Change `+ Agent` to either:
  - Spawn default provider immediately, or
  - Show two quick actions: `Claude Agent`, `Codex Agent`.

Message change:

- Send `{ type: 'openAgent', providerId: 'claude' | 'codex' }`.

Optional:

- Provider badge in `webview-ui/src/components/AgentLabels.tsx`.

---

## 8) Settings and package metadata

Update `package.json` `contributes.configuration` with:

- `pixel-agents.defaultProvider`
- `pixel-agents.codex.command`
- `pixel-agents.codex.args`
- `pixel-agents.codex.logPath`
- `pixel-agents.codex.enableAutoDiscover`

Also add constants and config helpers in `src/constants.ts`.

---

## 9) Reliability and diagnostics

Add safe observability for Codex integration:

- Provider-tagged logs: `[Pixel Agents][codex] ...`
- Rolling parse diagnostics buffer (e.g., last 200 warnings/events)
- Command: `Pixel Agents: Export Provider Diagnostics`

Failure behavior:

- If Codex binary missing: show user-friendly actionable error.
- If no logs discovered: show warning + keep terminal running.
- If parsing fails: continue with coarse active/idle heuristics.

---

## 10) Testing strategy

### 10.1 Unit tests

- Parser fixtures:
  - Claude regression fixtures (no behavior drift)
  - Codex fixtures (expected event variants + unknown lines)
- Normalization tests:
  - Raw provider events -> normalized events

### 10.2 Integration tests / manual validation

1. Launch Claude agent, verify no regressions.
2. Launch Codex agent from toolbar.
3. Validate tool start/done transitions in UI.
4. Validate waiting/permission states.
5. Reload VS Code window and verify restoration (mixed providers).

### 10.3 Failure-path validation

- `codex` command absent.
- Invalid codex log path.
- Corrupted log lines.

Expected: graceful warnings, no extension crash.

---

## 11) Rollout plan (PR sequence)

### Phase 1: Provider abstraction refactor

- Add provider interfaces.
- Move Claude code behind provider implementation.
- No functional changes.

### Phase 2: Codex backend MVP

- Add Codex launcher + log discovery.
- Add Codex parser and normalized event adapter.
- Add Codex settings.

### Phase 3: UI and persistence

- Add provider selection at agent creation.
- Persist provider metadata.
- Support mixed-provider restore/adoption.

### Phase 4: Hardening and docs

- Add diagnostics command.
- Expand troubleshooting docs.
- Add parser compatibility notes.

---

## 12) Acceptance criteria

Done means:

- Users can launch Codex-backed agents from Pixel Agents.
- Codex events drive character animations/status updates.
- Claude behavior remains unchanged.
- Provider identity persists across reloads.
- Unknown Codex lines never crash the extension.
- README contains Codex setup and troubleshooting instructions.

---

## 13) File-level checklist

- `src/types.ts`
- `src/constants.ts`
- `src/agentManager.ts`
- `src/fileWatcher.ts`
- `src/PixelAgentsViewProvider.ts`
- `src/transcriptParser.ts` (or provider dispatch replacement)
- `src/providers/types.ts` (new)
- `src/providers/claude/*` (new)
- `src/providers/codex/*` (new)
- `webview-ui/src/components/BottomToolbar.tsx`
- `webview-ui/src/components/AgentLabels.tsx` (optional)
- `package.json` (new configuration keys)
- `README.md` (Codex setup docs)

This order keeps risk low by separating refactor from new provider behavior.
