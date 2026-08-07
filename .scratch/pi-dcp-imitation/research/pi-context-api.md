# pi Context-Management API & Native Compaction Pipeline

Research map for building a DCP-imitation plugin that triggers pi's **native**
`compact()` at a percentage threshold while preserving pi's native compaction
features. All claims cite `file:line` against the installed pi package
(`@earendil-works/pi-coding-agent` v0.83.0) at
`/Users/crazytomatooo/.nvm/versions/node/v22.23.1/lib/node_modules/@earendil-works/pi-coding-agent`
(henceforth `PI`).

Source files inspected:

- `PI/docs/compaction.md` — native compaction pipeline (authoritative)
- `PI/docs/extensions.md` — extension API
- `PI/dist/core/extensions/types.d.ts` — compiled event/context type defs
- `PI/dist/core/compaction/compaction.d.ts` + `compaction.js` — compaction internals
- `PI/dist/core/agent-session.js` — runtime wiring (where `ctx.compact()` lands)
- `PI/examples/extensions/trigger-compact.ts` — threshold-trigger pattern
- `PI/examples/extensions/custom-compaction.ts` — `session_before_compact` custom-summary pattern

---

## 1. `pi.on("context")` event — field shape, when it fires, "remaining tokens"

**Event shape** (`types.d.ts:499`):

```ts
export interface ContextEvent {
    type: "context";
    messages: AgentMessage[];   // deep copy, safe to modify
}
```

Return type (`types.d.ts:774`): `ContextEventResult { messages?: AgentMessage[] }`.

**When it fires:** before each LLM call, per turn (`extensions.md:651` "Fired
before each LLM call"; lifecycle diagram at `extensions.md:299` shows `context`
inside the turn loop, after `turn_start`, before `before_provider_headers`).

**Token/percent/reserve/remaining — NOT on the event.** The `context` event
carries **only `messages`**. There is no `tokens`, `contextWindow`, `percent`,
`reserve`, or `remaining` field on `ContextEvent`. To read token usage you must
call `ctx.getContextUsage()` inside the handler (`types.d.ts:244`), which
returns `ContextUsage` (`types.d.ts:193`):

```ts
export interface ContextUsage {
    tokens: number | null;        // null right after compaction, before next LLM response
    contextWindow: number;        // model's context window
    percent: number | null;      // tokens / contextWindow * 100, or null if tokens unknown
}
```

**There is no `remaining` or `reserve` field.** Derive remaining yourself:
`remaining = usage.contextWindow - (usage.tokens ?? 0)`. The compaction
reserve (`reserveTokens`, default 16384) is **not** exposed here — read it from
settings or hardcode the default. `getContextUsage()` doc
(`extensions.md:1040`): "Uses last assistant usage when available, then
estimates tokens for trailing messages."

**Bottom line for DCP-imitation:** `pi.on("context")` is usable as a monitoring
hook (call `ctx.getContextUsage()` each LLM call), but it fires mid-turn.
`trigger-compact.ts` instead uses `pi.on("turn_end")` (`trigger-compact.ts:27`)
as the cleaner trigger point — once per turn, after the turn settles. Either
works; `turn_end` avoids firing during a multi-call turn.

---

## 2. `compact()` API — signature, proactive invocation, relation to `/compact` & auto-threshold

### Signature (`types.d.ts:200`, `types.d.ts:246`)

```ts
export interface CompactOptions {
    customInstructions?: string;
    onComplete?: (result: CompactionResult) => void;
    onError?: (error: Error) => void;
}
// on ExtensionContext:
compact(options?: CompactOptions): void;   // "Trigger compaction without awaiting completion"
```

`CompactionResult` (`compaction.d.ts:18`): `{ summary, firstKeptEntryId,
tokensBefore, estimatedTokensAfter?, usage?, details? }`.

### Proactive invocation from a hook — YES, fully supported

An extension can call `ctx.compact({...})` from inside any event handler
(`extensions.md:1048`-1051 "Trigger compaction without awaiting completion.
Use `onComplete` and `onError`"). `trigger-compact.ts` proves this: it calls
`ctx.compact({ customInstructions, onComplete, onError })` from a `turn_end`
handler (`trigger-compact.ts:12`-`23`) and from a registered command
(`trigger-compact.ts:43`-`47`). `onComplete`/`onError` are the only way to
observe the result, since `compact()` returns `void` (fire-and-forget).

### Wiring — `ctx.compact()` IS the `/compact` path

The extension-facing `ctx.compact()` action (`agent-session.js:1911`) is:

```js
compact: (options) => {
    void (async () => {
        try { const result = await this.compact(options?.customInstructions);
              options?.onComplete?.(result); }
        catch (error) { ...; options?.onError?.(err); }
    })();
}
```

It calls `this.compact(customInstructions)` (`agent-session.js:1367`) — the
**same method that the `/compact` slash command invokes**. It is NOT a separate
lightweight path. See §CRITICAL below for the full pipeline trace.

### Relation to auto-threshold & overflow

Auto-compaction uses a *separate* entry point `_runAutoCompaction(reason,
willRetry)` (`agent-session.js:1591`), invoked when:

- `shouldCompact()` returns true → `_runAutoCompaction("threshold", false)`
  (`agent-session.js:1584`);
- a turn overflows the window mid-run → `_runAutoCompaction("overflow",
  willRetry)` (`agent-session.js:1556`).

`ctx.compact()` always reports `reason: "manual"` and `willRetry: false`
(`agent-session.js:1371`, `1396`-`1397`, `1459`). So a DCP plugin triggering at
a percentage threshold will show up as a `manual` compaction (not `threshold`),
because it is the extension deciding when, not pi's built-in threshold check.

---

## 3. `session_before_compact` event — full shape, cancel, custom summary

### Event shape (`types.d.ts:441`)

```ts
export interface SessionBeforeCompactEvent {
    type: "session_before_compact";
    preparation: CompactionPreparation;
    branchEntries: SessionEntry[];
    customInstructions?: string;
    reason: "manual" | "threshold" | "overflow";
    willRetry: boolean;     // true only for overflow recovery
    signal: AbortSignal;
}
```

`CompactionPreparation` (`compaction.d.ts:111`): `{ firstKeptEntryId,
messagesToSummarize, turnPrefixMessages, isSplitTurn, tokensBefore,
previousSummary?, fileOps, settings }`.

### Result — can CANCEL or supply a CUSTOM summary (`types.d.ts:812`)

```ts
export interface SessionBeforeCompactResult {
    cancel?: boolean;              // cancel the compaction entirely
    compaction?: CompactionResult; // provide your own summary (bypasses native summary LLM call)
}
```

- **Cancel:** `return { cancel: true }` (`extensions.md:463`, `compaction.md:283`).
- **Custom summary:** `return { compaction: { summary, firstKeptEntryId,
  tokensBefore, usage?, details? } }` (`extensions.md:465`-`471`,
  `compaction.md:289`-`305`). When an extension returns `compaction`, pi skips
  its own summary-generation LLM call and saves the extension-provided summary
  verbatim (`agent-session.js:1405` sets `fromExtension = true`; `session_compact`
  then carries `fromExtension: true`).

### When it fires — including for `ctx.compact()` / `/compact`

`compaction.md:275`-`276`: "Fired before auto-compaction or `/compact`. Can
cancel or provide custom summary." Confirmed in source: both the manual method
(`agent-session.js:1390`-`1397`) and auto method (`agent-session.js:1617`-`1624`)
emit `session_before_compact` after `prepareCompaction()` and before calling the
summary `compact()` function.

### `custom-compaction.ts` reference

`custom-compaction.ts:22` registers `pi.on("session_before_compact", ...)`;
`:25`-`26` destructures `{ preparation: { messagesToSummarize,
turnPrefixMessages, tokensBefore, firstKeptEntryId, previousSummary } }` plus
`signal`; `:55` uses `serializeConversation(convertToLlm(allMessages))`; `:117`-`121`
returns `{ compaction: { summary, firstKeptEntryId, tokensBefore, usage } }`.
On any failure it `return`s with no result → falls back to native compaction
(`:112`, `:127`).

**DCP-imitation implication:** To *preserve* native features, the plugin must
**NOT** return a custom `compaction` from `session_before_compact` (that would
replace the native summary). It should either not register the handler at all,
or register it only to observe/log (return `undefined`). The native `compact()`
then generates the structured summary itself.

---

## 4. `session_compact` + `session_before_tree` shapes

### `session_compact` (fired AFTER compaction) — `types.d.ts:453`

```ts
export interface SessionCompactEvent {
    type: "session_compact";
    compactionEntry: CompactionEntry;   // the saved entry (summary, firstKeptEntryId, tokensBefore, ...)
    fromExtension: boolean;             // true iff an extension provided the summary via session_before_compact
    reason: "manual" | "threshold" | "overflow";
    willRetry: boolean;
}
```

No result type (informational only). `extensions.md:476`-`481`. Emitted at
`agent-session.js:1442` (manual) and `:1683` (auto).

### `session_before_tree` + `TreePreparation` — `types.d.ts:470`,`484`

```ts
export interface TreePreparation {
    targetId: string;
    oldLeafId: string | null;
    commonAncestorId: string | null;
    entriesToSummarize: SessionEntry[];
    userWantsSummary: boolean;
    customInstructions?: string;
    replaceInstructions?: boolean;
    label?: string;
}
export interface SessionBeforeTreeEvent {
    type: "session_before_tree";
    preparation: TreePreparation;
    signal: AbortSignal;
}
```

`SessionBeforeTreeResult` (`types.d.ts` ~830): can `cancel` navigation, or return
`{ summary: { summary, details?, usage? } }` plus override
`customInstructions`/`replaceInstructions`/`label`. Fires before `/tree`
navigation regardless of whether the user chose to summarize (`compaction.md:349`,
`extensions.md:484`-`489`). The post-navigation event is `session_tree`
(`types.d.ts` ~491): `{ newLeafId, oldLeafId, summaryEntry?, fromExtension? }`.

---

## 5. `before_agent_start` — system-prompt injection mechanism

### Event + result (`types.d.ts:524`, `types.d.ts:800`)

```ts
export interface BeforeAgentStartEvent {
    type: "before_agent_start";
    prompt: string;                 // raw user prompt (after expansion)
    images?: ImageContent[];
    systemPrompt: string;          // fully assembled system prompt (chained across handlers)
    systemPromptOptions: BuildSystemPromptOptions;  // .customPrompt, .selectedTools, .toolSnippets,
                                                     // .promptGuidelines, .appendSystemPrompt, .cwd,
                                                     // .contextFiles, .skills
}
export interface BeforeAgentStartEventResult {
    message?: Pick<CustomMessage, "customType" | "content" | "display" | "details">;
    systemPrompt?: string;   // REPLACE the system prompt for this turn (chained across extensions)
}
```

### How DCP-style injection works

Fires after the user submits a prompt, before the agent loop (`extensions.md:521`).
To inject a "context-constrained, use compress" routing rule, return a rewritten
`systemPrompt` (`extensions.md:533`-`536`):

```ts
pi.on("before_agent_start", async (event, ctx) => {
    return {
        systemPrompt: event.systemPrompt + "\n\nYou operate in a context-constrained environment. ...\nUse the compress tool ...",
    };
});
```

Key behavior (`extensions.md:553`-`556`): `event.systemPrompt` and
`ctx.getSystemPrompt()` both reflect the **chained** prompt as of the current
handler; later handlers can still modify it. Multiple extensions returning
`systemPrompt` are chained in load order. `event.systemPromptOptions`
(`extensions.md:540`-`548`) exposes the structured inputs (custom prompt, active
tools, tool snippets, prompt guidelines, context files, skills) so you can make
informed edits without re-discovering resources.

**Note:** `systemPrompt` changes are NOT reflected by `ctx.getSystemPrompt()`
for later `context`/`before_provider_request` in the sense of provider payload
(`extensions.md:1065`-`1096`); for provider-level system-instruction rewrites
use `before_provider_request` instead. For DCP routing text appended to the
system prompt, `before_agent_start` is the correct, documented hook.

---

## 6. `registerTool` / `registerCommand` — signatures

### `registerTool` (`types.d.ts:890`)

```ts
registerTool<TParams extends TSchema, TDetails = unknown, TState = any>(
    tool: ToolDefinition<TParams, TDetails, TState>
): void;
```

`ToolDefinition` (`types.d.ts:343`): `{ name, label, description, promptSnippet?,
promptGuidelines?, parameters: TSchema, constrainedSampling?, renderShell?,
prepareArguments?, executionMode?, execute(toolCallId, params, signal, onUpdate,
ctx): Promise<AgentToolResult<TDetails>>, renderCall?, renderResult? }`. Works
during load AND after startup (refreshed immediately, no `/reload`)
(`extensions.md:1341`). To make the tool appear in the system prompt's
"Available tools", set `promptSnippet`; to add guideline bullets, set
`promptGuidelines` (bullets must name the tool — `extensions.md:1349`-`1351`).

Full register example: `extensions.md:1355`-`1390` (TypeBox `Type.Object({...})`
for `parameters`, async `execute` returning `{ content, details }`).

### `registerCommand` (`types.d.ts:892`)

```ts
registerCommand(name: string, options: Omit<RegisteredCommand, "name" | "sourceInfo">): void;
```

`RegisteredCommand` (`types.d.ts:840`): `{ name, sourceInfo, description?,
getArgumentCompletions?(prefix), handler: (args: string, ctx:
ExtensionCommandContext) => Promise<void> }`. The handler receives
`ExtensionCommandContext` (extends `ExtensionContext` with `waitForIdle`,
`newSession`, `fork`, `navigateTree`, `switchSession`, `reload`,
`getSystemPromptOptions`). Example: `extensions.md:1500`-`1514`.

DCP-imitation will register `/dcp:*` commands this way (e.g.
`pi.registerCommand("dcp:status", { handler: async (args, ctx) => {...} })`),
and the `compress` tool via `pi.registerTool({ name: "compress", ... })`.

---

## 7. Native compaction pipeline (`docs/compaction.md`)

### Trigger condition (`compaction.md:32`, `compaction.js:160`-`163`)

```
contextTokens > contextWindow - reserveTokens
```

`shouldCompact(contextTokens, contextWindow, settings)` →
`contextTokens > contextWindow - settings.reserveTokens` (`compaction.js:163`).

### Defaults & settings (`compaction.md:35`, `compaction.md:389`-`390`, `compaction.js:76`, `compaction.d.ts:28`-`31`)

- `reserveTokens` default **16384** (`compaction.js:76` `reserveTokens: 16384`)
- `keepRecentTokens` default **20000**
- `enabled` default **true**
- Configurable in `~/.pi/agent/settings.json` or
  `<project-dir>/.pi/settings.json` under `compaction: { enabled,
  reserveTokens, keepRecentTokens }` (`compaction.md:381`-`399`).
- `CompactionSettings` interface (`compaction.d.ts:28`): `{ enabled,
  reserveTokens, keepRecentTokens }`.

### How it works — 5 steps (`compaction.md:39`-`45`)

1. **Find cut point:** walk backwards from newest message, accumulating token
   estimates until `keepRecentTokens` reached (`compaction.d.ts:77`-`94`
   `findCutPoint`).
2. **Extract messages:** collect messages from previous kept boundary (or
   session start) up to the cut point.
3. **Generate summary:** call LLM with the structured summary format, passing
   the previous summary as iterative context when present.
4. **Append entry:** save `CompactionEntry` with summary + `firstKeptEntryId`.
5. **Reload:** session reloads, using summary + messages from
   `firstKeptEntryId` onwards.

### Cut-point rules (`compaction.md:109`-`117`)

Valid cut points: user messages, assistant messages, BashExecution messages,
custom messages (`custom_message`, `branch_summary`). **Never** cut at tool
results (they must stay with their tool call). When cutting at an assistant
message with tool calls, its tool results come after and will be kept
(`compaction.d.ts:84`-`87`).

### Split turns (`compaction.md:81`-`107`)

A "turn" = user message + all assistant responses/tool calls until the next
user message. When a single turn exceeds `keepRecentTokens`, the cut lands
mid-turn → `isSplitTurn = true`, `turnPrefixMessages` holds the early turn
parts. For split turns pi generates **two** summaries and merges them
(`compaction.md:101`-`105`): (1) history summary, (2) turn-prefix summary. In
source: `compaction.js:592` (history via `generateSummaryWithUsage`) and
`compaction.js:596` (`generateTurnPrefixSummary`); non-split path at
`compaction.js:603`.

### `CompactionEntry` structure (`compaction.md:119`-`146`, `compaction.d.ts` `CompactionResult`)

```ts
interface CompactionEntry<T = unknown> {
    type: "compaction";
    id: string; parentId: string; timestamp: number;
    summary: string;
    firstKeptEntryId: string;
    tokensBefore: number;
    usage?: Usage;       // LLM usage that generated the summary
    fromHook?: boolean;  // true if provided by extension (legacy field name)
    details?: T;         // default: CompactionDetails { readFiles: string[]; modifiedFiles: string[] }
}
```

`CompactionResult` (`compaction.d.ts:18`): `{ summary, firstKeptEntryId,
tokensBefore, estimatedTokensAfter?, usage?, details? }` — SessionManager adds
`id`/`parentId` when saving (`compaction.d.ts:16`).

### Summary format (`compaction.md:215`-`253`)

Structured markdown, identical for compaction and branch summarization:

```
## Goal
## Constraints & Preferences
## Progress
### Done / ### In Progress / ### Blocked
## Key Decisions
## Next Steps
## Critical Context
<read-files>...</read-files>
<modified-files>...</modified-files>
```

Sections (`compaction.md:220`-`250`). Message serialization (`compaction.md:255`):
`[User]:`, `[Assistant thinking]:`, `[Assistant]:`, `[Assistant tool calls]:
read(path="..."); bash(command="...")`, `[Tool result]:` — tool results
truncated to 2000 chars during serialization (`compaction.md` ~268).

### Branch summarization (`compaction.md:148`-`213`)

Triggered by `/tree` navigation: find common ancestor, collect entries old→common
ancestor, prepare with budget, generate summary, append `BranchSummaryEntry`
(`compaction.md:154`-`160`). `BranchSummaryEntry` (`compaction.md:187`-`213`):
`{ type: "branch_summary", id, parentId, timestamp, summary, fromId, usage?,
fromHook?, details? }`. Cumulative file tracking across compactions + branch
summaries (`compaction.md:179`-`185`).

### Custom summarization via extensions (`compaction.md:271`-`381`)

`session_before_compact` (`compaction.md:275`): cancel or return
`{ compaction: { summary, firstKeptEntryId, tokensBefore, usage?, details? } }`.
Helpers exported from the package for custom summarizers:
`serializeConversation` + `convertToLlm` (`compaction.md:290`-`296`,
used at `custom-compaction.ts:55`). `session_before_tree` (`compaction.md:349`):
cancel navigation or provide custom branch summary.

---

## CRITICAL QUESTION — Does `ctx.compact()` run the SAME native pipeline?

**YES. `ctx.compact()` runs the identical native pipeline as `/compact` and
auto-compaction. All native features (keepRecentTokens cut-point, split-turn,
structured summary format, CompactionEntry, session reload, cumulative file
tracking) are fully preserved.**

### Evidence trace through `agent-session.js`

1. Extension calls `ctx.compact(options)` → action at `agent-session.js:1911`
   calls `await this.compact(options?.customInstructions)`.
2. `this.compact(customInstructions)` (`agent-session.js:1367`) is the **manual
   compaction method** (also used by the `/compact` slash command):
   - emits `compaction_start { reason: "manual" }` (`agent-session.js:1371`)
   - **`prepareCompaction(pathEntries, settings)`** (`agent-session.js:1379`) —
     the SAME `prepareCompaction` (`compaction.d.ts:128`) that applies
     `keepRecentTokens`, `findCutPoint`, split-turn detection
   - fires **`session_before_compact`** with `reason: "manual"`, `willRetry:
     false`, and the full `preparation` (`agent-session.js:1390`-`1397`) — so
     extensions can still intercept/cancel/customize a programmatic compact
   - if no extension supplies a custom `compaction`, calls **`compact(preparation,
     this.model, apiKey, headers, customInstructions, ...)`**
     (`agent-session.js:1423`) — the SAME native `compact()` function
     (`compaction.d.ts:136`) that generates the structured
     Goal/Constraints/Progress/... summary (incl. split-turn two-summary merge
     at `compaction.js:592`/`596`/`603`)
   - `sessionManager.appendCompaction(summary, firstKeptEntryId, tokensBefore,
     details, fromExtension, usage)` (`agent-session.js:1433`) — saves
     `CompactionEntry`, then session reloads
   - emits **`session_compact`** with `fromExtension`, `reason: "manual"`,
     `willRetry: false` (`agent-session.js:1442`-`1446`)
3. Auto-compaction `_runAutoCompaction(reason, willRetry)` (`agent-session.js:1591`)
   follows the **same shape**: `prepareCompaction` at `:1608`,
   `session_before_compact` at `:1617`, native `compact()` at `:1657`,
   `appendCompaction` at `:1674`, `session_compact` at `:1683`. Only `reason`
   (`"threshold"`/`"overflow"`) and `willRetry` differ.

### What differs for a programmatic `ctx.compact()` vs the auto-threshold

- `reason` is `"manual"` (not `"threshold"`).
- `willRetry` is always `false` (programmatic compaction does not retry an
  aborted turn; only overflow recovery retries).
- Uses a dedicated `_compactionAbortController` (`agent-session.js:1423`) vs
  `_autoCompactionAbortController` (`agent-session.js:1657`) — separate abort
  controllers, otherwise identical.
- `customInstructions` is honored (passed to `compact()` → the native summary
  prompt), so a DCP plugin can still focus the summary without replacing it.

### Conclusion for the DCP-imitation plugin

The user's goal — "preserve pi's native compaction features" — is **fully met**
by calling `ctx.compact({ onComplete, onError })` at the chosen percentage
threshold. The plugin should:

- monitor via `pi.on("turn_end")` + `ctx.getContextUsage()` (percent-based; see
  `trigger-compact.ts:27`-`40`);
- call `ctx.compact({ onComplete, onError })` when the threshold is crossed
  (`trigger-compact.ts:12`-`23`);
- **NOT** return a custom `compaction` from `session_before_compact` (that would
  replace the native structured summary — defeating "preserve native features");
- optionally inject routing text via `before_agent_start`
  (`extensions.md:533`-`536`) and register the `compress` tool + `/dcp:*`
  commands via `registerTool`/`registerCommand` (`types.d.ts:890`,`892`).

The native `keepRecentTokens` (20k), split-turn handling, structured summary
format, `CompactionEntry`, and session reload all run unmodified because the
plugin triggers the same code path as `/compact`.

---

## Appendix — `ContextUsage` has no "remaining"/"reserve" field (surprise)

The `context` event object itself carries **only `messages`**
(`types.d.ts:499`). There is no tokens/percent/reserve/remaining on the event.
Token visibility requires a separate `ctx.getContextUsage()` call
(`types.d.ts:244`) returning `{ tokens: number|null, contextWindow: number,
percent: number|null }` (`types.d.ts:193`). Compute:

- `remaining = contextWindow - (tokens ?? 0)`
- `reserve` = the compaction `reserveTokens` (16384 default) — only known from
  settings, NOT from `ContextUsage`.

`tokens` is `null` right after compaction and before the next LLM response
(`types.d.ts:195`), so a threshold check must guard for null
(`trigger-compact.ts:30`-`32`).
