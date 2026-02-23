# OpenCode Agent Integration Plan

This plan describes how to extend Pixel Agents so it can connect to and visualize **OpenCode** agents (in addition to Claude Code), while preserving the current user experience and backwards compatibility.

## 1) Current architecture and extension points

Pixel Agents currently assumes one provider (Claude Code) in three core places:

1. **Agent launch + transcript path discovery**
   - `src/agentManager.ts` creates terminals with `claude --session-id ...`.
   - It resolves transcript files from `~/.claude/projects/<workspace>/`.
2. **Transcript parsing + status inference**
   - `src/transcriptParser.ts` parses Claude JSONL records (`assistant`, `user`, `system.turn_duration`, `progress`).
3. **Project scan + auto-adoption of sessions**
   - `src/fileWatcher.ts` scans project transcript directories and adopts JSONL files.

The webview protocol is already provider-agnostic enough for rendering (`agentCreated`, `agentStatus`, `agentToolStart`, etc.), so the main work is in the extension backend abstraction layer.

---

## 2) Target outcome

After implementation, a user should be able to:

- Create a new agent and choose **Claude Code** or **OpenCode**.
- Have Pixel Agents discover and attach to OpenCode transcript/log files.
- See OpenCode agent activities mapped to existing visual states (walk/type/read/waiting/permission).
- Restore OpenCode agents across VS Code reloads.

Non-goals for first release:

- Perfect one-to-one tool semantics for every OpenCode event.
- Complex multi-provider orchestration UI (keep UI minimal initially).

---

## 3) Implementation strategy: provider abstraction

### 3.1 Introduce provider interfaces

Create a provider contract so Claude and OpenCode are pluggable:

- New file: `src/providers/types.ts`
  - `AgentProviderId = 'claude' | 'opencode'`
  - `ProviderLaunchConfig`
  - `ProviderSessionLocator`
  - `ProviderTranscriptParser`
  - `NormalizedAgentEvent` (provider-neutral event model)

Define a normalized event model used by existing webview messages:

- `tool_start`, `tool_done`
- `status_active`, `status_waiting`, `status_permission_required`
- `subagent_tool_start`, `subagent_tool_done`, `subagent_clear`
- `turn_end`

### 3.2 Extract Claude implementation behind the interface

- Move Claude-specific launch/path logic from `src/agentManager.ts` into:
  - `src/providers/claude/claudeProvider.ts`
- Move Claude parsing logic from `src/transcriptParser.ts` into:
  - `src/providers/claude/claudeTranscriptParser.ts`

Keep behavior identical during this step (pure refactor).

### 3.3 Add OpenCode provider implementation

Create:

- `src/providers/opencode/opencodeProvider.ts`
- `src/providers/opencode/opencodeTranscriptParser.ts`

Responsibilities:

- Spawn OpenCode command (configurable command string).
- Resolve OpenCode transcript directory and session-file naming patterns.
- Parse OpenCode records and emit normalized events.

> Important: Do not hardcode uncertain OpenCode formats. Add configuration and feature flags so the parser can evolve safely.

---

## 4) Data model and persistence changes

Update `src/types.ts` and persisted schemas to include provider identity:

- Add `providerId` to `AgentState` and `PersistedAgent`.
- Add optional provider metadata (`providerSessionId`, `providerData`) for diagnostics.

Migration logic:

- Existing agents without `providerId` default to `'claude'`.
- Keep workspace state keys unchanged to avoid data loss; migrate in-place when loading.

---

## 5) Configuration and settings

Add VS Code settings in `package.json` contributes.configuration:

- `pixel-agents.defaultProvider` (`claude` | `opencode`, default `claude`)
- `pixel-agents.opencode.command` (default e.g. `opencode`)
- `pixel-agents.opencode.args` (array)
- `pixel-agents.opencode.sessionsPath` (optional override)
- `pixel-agents.opencode.enableAutoDiscover` (boolean)

Then add constants in `src/constants.ts` and a setting reader helper.

---

## 6) UI/webview changes (minimal)

In `webview-ui/src/components/BottomToolbar.tsx` and message plumbing:

- Change “+ Agent” action to either:
  1. immediate spawn with default provider, or
  2. small dropdown/menu: **Claude Agent** / **OpenCode Agent**.

Message updates:

- `openClaude` → generalized `openAgent` with payload `{ providerId }`.
- Keep backwards compatibility by still handling `openClaude` on extension side for one version.

Optional UX improvement:

- Display provider badge in `webview-ui/src/components/AgentLabels.tsx`.

---

## 7) Backend flow updates

### 7.1 Launch flow

In `src/PixelAgentsViewProvider.ts`:

- Route `openAgent` to `launchAgent(providerId, ...)`.
- Provider returns launch details (terminal name prefix, expected session id/file).

In `src/agentManager.ts`:

- Replace direct Claude command/path assumptions with provider calls.
- Keep shared lifecycle (timers, watchers, cleanup) unchanged.

### 7.2 File watching and adoption

In `src/fileWatcher.ts`:

- Support provider-specific project/session directories.
- Track known files keyed by provider + path, not just path.
- During scan, infer provider via directory source or file signature.

### 7.3 Transcript parsing

- Delegate line parsing to provider parser.
- Provider parser emits normalized events.
- Existing timer behavior (`waiting`, `permission`) should consume normalized events, not raw record shapes.

---

## 8) Robustness and observability

Add diagnostics and guardrails:

- Structured logs with provider tag: `[Pixel Agents][opencode] ...`
- Graceful fallback when OpenCode transcript format is unknown:
  - Mark agent active/idle via conservative heuristics.
  - Avoid crashing on malformed records.
- Add command: `Pixel Agents: Export Provider Diagnostics` to dump recent parse samples and errors.

---

## 9) Testing plan

### 9.1 Unit tests (new)

Introduce test setup (if absent) for parser-level tests:

- Claude parser regression fixtures (ensure no behavior change).
- OpenCode parser fixtures for expected transcript variants.
- Normalization tests: raw record → normalized events.

### 9.2 Integration checks

Manual scenarios in Extension Development Host:

1. Launch Claude agent and verify existing behavior.
2. Launch OpenCode agent via new UI action.
3. Confirm tool start/done and waiting transitions.
4. Reload window and verify persisted provider restoration.
5. Verify mixed-provider sessions in same workspace.

### 9.3 Failure-path tests

- Missing OpenCode binary.
- Invalid `sessionsPath`.
- Corrupt transcript lines.

Expected behavior: user-facing warning + no extension crash.

---

## 10) Delivery phases (recommended PR sequence)

### Phase 1 — Refactor only (no functional change)

- Add provider interfaces.
- Move Claude logic behind provider abstraction.
- Ensure all current tests/manual behavior pass.

### Phase 2 — OpenCode MVP backend

- Add OpenCode provider launch + transcript discovery.
- Add OpenCode parser with basic activity mapping.
- Add settings for command/path overrides.

### Phase 3 — UI support + persistence

- Add provider selection in create-agent flow.
- Persist/restore provider metadata.
- Mixed-provider support in scanning/adoption.

### Phase 4 — Hardening + docs

- Add diagnostics command.
- Add docs for OpenCode setup and troubleshooting.
- Expand fixtures and parser compatibility table.

---

## 11) Acceptance criteria

Feature is complete when:

- Users can start and visualize OpenCode agents from Pixel Agents.
- Claude behavior remains unchanged.
- Provider identity persists across reloads.
- No unhandled exceptions from unknown OpenCode records.
- README includes setup steps and known limitations for OpenCode.

---

## 12) File-level change checklist

- `src/types.ts`
- `src/constants.ts`
- `src/agentManager.ts`
- `src/fileWatcher.ts`
- `src/PixelAgentsViewProvider.ts`
- `src/transcriptParser.ts` (or replaced by provider parsers)
- `src/providers/types.ts` (new)
- `src/providers/claude/*` (new)
- `src/providers/opencode/*` (new)
- `webview-ui/src/components/BottomToolbar.tsx`
- `webview-ui/src/components/AgentLabels.tsx` (optional)
- `package.json` (contributes.configuration)
- `README.md` (OpenCode setup docs)

This sequence minimizes risk by isolating refactor, then OpenCode behavior, then UI/persistence changes.
