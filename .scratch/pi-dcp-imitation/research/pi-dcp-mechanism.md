# pi-dcp (v0.5.0) Mechanism Writeup

> Goal: give a developer enough to build a production imitation of `@pi-vault/pi-dcp`.
> All `file:line` citations are relative to
> `~/.pi/agent/npm/node_modules/@pi-vault/pi-dcp/` unless an absolute path is given.
> Line numbers are from the installed v0.5.0 source (TypeBox runtime dep, jiti-loaded TS).

---

## 0. TL;DR / Mental Model

pi-dcp is a **context-hygiene extension** that rides on pi's `context` event. On every
context pass it runs a pure pipeline that: (a) strips hallucinated `<dcp-*>` tags the model
might emit, (b) prunes duplicate tool outputs and stale failed tool inputs automatically,
(c) injects stable `<dcp-message-id>` anchors so the model can address ranges, (d) injects
frequency-throttled `<dcp-system-reminder>` nudges that *tell the model* to call the
`compress` tool, and (e) applies any active compression blocks (replacing ranges with
summaries). It also registers a `compress` tool the model calls voluntarily, and 10
`dcp:*` commands. State survives pi's native compaction via a serialized snapshot appended
to pi's session branch (`pi.appendEntry`).

**Critical observation (answered in detail in §10): pi-dcp NEVER hard-triggers
compaction.** It only nudges. See §10.

---

## 1. Entry `src/index.ts` — every `pi.on(...)` hook

Entry point is `createExtension(pi)` at `src/index.ts:98`. It closes over module-scoped
`state`, `config`, `latestMessages`, `promptStore`, `runtimePrompts`,
`lastPersistedFingerprint`. Helpers: `persistIfChanged` (110), `getSessionId` (124),
`restoreActiveBranch` (132), `reloadConfig` (160), `executeCompressTool` (174),
`registerCompressTool` (198). Commands registered at 172.

### Hook-by-hook table

| Hook | line | Fires when | What it does |
| --- | --- | --- | --- |
| `before_agent_start` | 259 | before each agent turn, with `event.systemPrompt` | If enabled, not denied, and (not sub-agent unless allowed) → returns `{ systemPrompt: (event.systemPrompt ?? "") + (runtimePrompts?.system ?? DCP_SYSTEM_PROMPT) }` (264-267). Appends DCP's system prompt. |
| `session_start` | 270 | session begins, `event.reason` available | Sets `logDir = <sessionDir>/dcp/logs` (271); `reloadConfig` (272); `registerCompressTool()` (274); `resetSessionState` (276); clears `lastPersistedFingerprint` (277); sets `manualMode`/`compressPermission` from config (278-279); if `experimental.customPrompts`, builds `PromptStore` with project (trusted-only) + global override dirs, reloads, writes defaults (281-296); `restoreActiveBranch` (298); reads `ctx.getContextUsage()` → `state.modelContextWindow` (301-304); logs; `persistIfChanged(forcePersist)` (311). |
| `session_tree` | 314 | a sub-agent/session tree node spawns (e.g. fork) | `resetSessionState` (315); reset `manualMode`/`compressPermission` (316-317); `restoreActiveBranch` (318); detect sub-agent via `PI_SUBAGENT_CHILD==="1"` (319); persist (320). |
| `session_compact` | 323 | **after** pi performs its native compaction (listener, not trigger) | Clears volatile caches: `prune.tools`, `prune.messages.{byMessageIndex,blocksById,activeBlockIds,activeByAnchorIndex}` (324-328), `messageIds.byIndex` (329) — **but retains `byRawId`/`byRef` and `nextRefIndex`** (330-332 comment), `compressionTiming.startTimes` (333), `subAgentResultCache` (334); sets `lastCompaction=Date.now()` (335); logs "compaction detected" (336); `persistIfChanged()` (337). |
| `session_shutdown` | 340 | session ends | `persistIfChanged()` (341); logs "session shutdown" (342). |
| `message_end` | 345 | after a message is finalized, `event.message` | If enabled and `event.message.role === "assistant"` (346): strips hallucinated DCP tags via `mapText(msg, stripHallucinationsFromString)` (348); if changed, returns `{ message: stripped }` (349-352). Otherwise returns nothing. |
| `tool_call` | 355 | when a tool call is requested, before execution | If enabled and `event.toolName !== "compress"` → returns undefined (356-357). If `compressPermission ?? config.compress.permission === "deny"` → returns `{ block: true, reason: "Compression denied by configuration" }` (358-362). I.e. the only tool it gates is `compress`. |
| `tool_execution_start` | 366 | when a tool starts executing | If `event.toolName === "compress"` → records `state.compressionTiming.startTimes.set(event.toolCallId, Date.now())` (367-369). Used to compute compress duration later. |
| `tool_execution_end` | 372 | when a tool finishes | If `compress` → `applyCompressionTiming(state, event)` then persist, return (374-377). Else if `subagent` and not error → reads `event.result.details.childSessionPath`, `parseChildSessionResults`, caches into `state.subAgentResultCache` (379-389). |
| `context` | 395 | every context window assembly (the hot path) | See §2. |

### `pi.on("context")` body (src/index.ts:395-463)

```
395  pi.on("context", async (event, ctx) => {
396    if (!config.enabled) return;
397    if (state.isSubAgent && !config.experimental.allowSubAgents) return;
399    const usage = ctx.getContextUsage();           // ContextUsage | undefined
400    if (usage) state.modelContextWindow = usage.contextWindow;
401-404  if (ctx.model) { state.modelId = ctx.model.id; state.modelProvider = ctx.model.provider; }
405    latestMessages = event.messages;
407-410  if (promptStore) { promptStore.reload(); runtimePrompts = promptStore.getRuntimePrompts(); }
412    const result = runPipeline(state, config, event.messages,
            usage ? { tokens: usage.tokens, contextWindow: usage.contextWindow, percent: usage.percent } : undefined,
            runtimePrompts);
426    if (result.strategyResult.pruned > 0) logger.info(...);
433    persistIfChanged();
435-460  UI notification (toast per-pass or status cumulative)
462    return { messages: result.messages };
463  });
```

Note: it returns `{ messages }` to **transform** the context pi sends to the model. It calls
no compact/summarize API. The `session_compact` hook (323) is purely reactive to pi's own
compaction.

---

## 2. Context monitoring — event shape & threshold math

### Event fields

- `event.messages: AgentMessage[]` — the assembled context (index.ts:405, 415).
- `ctx.getContextUsage()` returns a `ContextUsage` with `.tokens`, `.contextWindow`,
  `.percent` (used at index.ts:399, 417-421; consumed in `commands/context.ts:9-14`).
- `ctx.model` → `{ id, provider }` (index.ts:401-404).
- `ctx.hasUI`, `ctx.ui.notify`/`ctx.ui.setStatus` (index.ts:435-457).

### `isContextOverLimits` (`src/utils/context-limits.ts:40-65`)

Returns `{ overMaxLimit: boolean; overMinLimit: boolean }`.

```
45  if (contextUsage.tokens == null) return { overMaxLimit: false, overMinLimit: false };
49  const tokens = contextUsage.tokens;
50  const modelKey = (state.modelProvider && state.modelId) ? `${provider}/${modelId}` : undefined;
54  const effectiveWindow = state.modelContextWindow ?? (contextUsage.contextWindow > 0 ? contextUsage.contextWindow : undefined);
58  const maxLimit = resolveMaxLimit(config.compress, effectiveWindow, modelKey);
59  const minLimit = resolveMinLimit(config.compress, effectiveWindow, modelKey);
61  return {
62    overMaxLimit: maxLimit !== undefined ? tokens >= maxLimit : false,
63    overMinLimit: minLimit !== undefined ? tokens >= minLimit : false,
64  };
```

### Limit resolution order (per threshold) — `resolveMaxLimit` (67-88) / `resolveMinLimit` (90-111)

For each of max/min, three-tier fallback:

1. **Per-model override** — `compress.modelMaxLimits`/`modelMinLimits[modelKey]` (73-75 / 96-98).
2. **Global absolute** — `compress.maxContextLimit`/`minContextLimit` (78-80 / 101-103).
3. **Legacy percentage fallback** — `maxContextPercent`/`minContextPercent × contextWindow` (83-84 / 106-107).

`resolveContextTokenLimit` (11-27): a `number` passes through; a string matching
`/^(\d+(?:\.\d+)?)%$/` is parsed and multiplied by `contextWindow` (so `"80%"` ≡ percent
field). Out-of-range (>100, ≤0) or no window → `undefined`.

### Defaults & validation (`src/config.ts`)

- `DEFAULT_CONFIG` sets `compress.protectedTools = ["compress"]` and concrete
  `maxContextLimit = 200000`, `minContextLimit = 100000` (config.ts ~line 28-40, the
  Optional schema fields without defaults).
- `loadConfig` post-validates: `maxContextPercent > 100` → reset; `minContextPercent > 100`
  → reset; `maxContextPercent <= minContextPercent` → reset both (config.ts range-fix block).
- Schema defaults: `maxContextPercent=80`, `minContextPercent=50` (config-schema.ts:51-58).

### When it decides to nudge (`src/messages/inject.ts:97-196`, `injectCompressNudges`)

Guard exits: no `contextUsage` (104), `messages.length===0` (105), `state.manualMode` (106),
`state.compressPermission === "deny"` (107).

Decision stage (109-192):

- `isContextOverLimits(...)` → `{overMax, overMin}` (110-114).
- **summaryBuffer** adjustment (117-132): if `effectiveOverMax` and
  `config.compress.summaryBuffer` and tokens known → subtract active summary token usage
  (`getActiveSummaryTokenUsage`) and re-evaluate `effectiveOverMax`. Prevents cascading
  when summaries themselves consume tokens.
- `if (!overMin) return messages;` (134) — **nothing happens until tokens cross the min
  threshold.** This is the gating condition for any nudge.
- Pick the last injectable (user/assistant) message as anchor target (136-145).
- Nudge type (148-165):
  - `effectiveOverMax` → `"contextLimit"` (149-150).
  - else if anchor is `user` → `"turn"` (153-154).
  - else count trailing assistant messages since last user; if
    `>= config.compress.iterationNudgeThreshold` → `"iteration"` (156-163).
- Anchoring (167-191): context-limit nudges **always** anchor (179-180, ignore frequency);
  turn/iteration use `addAnchorIfAllowed(anchorSet, key, index, state, count, nudgeFrequency)`
  (182-189) — only adds if the nearest existing anchor is ≥ `nudgeFrequency` messages away
  (226-256). Anchors are **stable message keys** (content-derived), surviving compaction.

Application stage (194-196 → `applyAnchoredNudges` 261-296): walks all messages, and at any
index whose key is in an anchor set, appends the corresponding nudge text (unless a nudge
already present — `hasExistingNudge` 298-311 checks for `<dcp-system-reminder>`).

So: nudges fire (1) only past `minLimit`, (2) at most every `nudgeFrequency` messages for
non-urgent types, (3) always when past `maxLimit`.

---

## 3. Nudge injection — prompts

### `DCP_SYSTEM_PROMPT` (`src/prompts/system.ts`, whole file)

Appended to pi's system prompt every turn via `before_agent_start` (index.ts:264-267).
Content (verbatim themes): "context-constrained environment"; "The ONLY tool you have for
context management is `compress`"; "`<dcp-message-id>` and `<dcp-system-reminder>` tags are
environment-injected metadata. Do not output them."; the COMPRESS WHEN / DO NOT COMPRESS IF
lists; "Evaluate conversation signal-to-noise REGULARLY." This is the exact text that
appears at the top of THIS agent's own system prompt — confirming the prompt is shipped
verbatim.

### `CONTEXT_LIMIT_NUDGE` (`src/prompts/nudges.ts:6`)

Trigger: `effectiveOverMax` true (inject.ts:149). Wrapped in `<dcp-system-reminder>`.
Content: "CRITICAL WARNING: MAX CONTEXT LIMIT REACHED … You MUST use the `compress` tool
now. Do not continue normal exploration until compression is handled." Includes
SELECTION PROCESS (older resolved history first) and SUMMARY REQUIREMENTS (preserve user
intent, prefer direct quotes).

### `TURN_NUDGE` (`src/prompts/nudges.ts`, after CONTEXT_LIMIT_NUDGE)

Trigger: anchor is a user message and not overMax (inject.ts:153-154). Content: "Evaluate
the conversation for compressible ranges … use the compress tool on them … Keep active
context uncompressed."

### `ITERATION_NUDGE` (`src/prompts/nudges.ts`, last)

Trigger: trailing assistant count `>= iterationNudgeThreshold` (default 15) and not overMax
and anchor is assistant (inject.ts:156-163). Content: "You've been iterating for a while
after the last user message. … use the compress tool on it now."

### Prompt override (`src/prompts/store.ts`)

`PromptStore` (class def in store.ts) precedence: **project > global > bundled defaults**
(loadOverride). Project override dir only when `ctx.isProjectTrusted?.()` (index.ts:282-284):
`.pi/dcp-prompts/overrides/<file>.md`; global: `~/.pi/agent/extensions/dcp-prompts/overrides/`.
Files: `system.md`, `context-limit-nudge.md`, `turn-nudge.md`, `iteration-nudge.md`
(PROMPT_FILES map). `BUNDLED_DEFAULTS` = the in-code constants. HTML comments stripped on
load (`<!--...-->` regex). `writeDefaultPrompts` writes bundled defaults for reference
(store.ts end). Only enabled when `experimental.customPrompts` (index.ts:281). The
`compress` tool description is NOT overridable (store.ts comment).

### `before_agent_start` injection path (index.ts:259-268)

`runtimePrompts?.system ?? DCP_SYSTEM_PROMPT` appended to `event.systemPrompt ?? ""`.
Skipped when disabled, denied, or sub-agent-not-allowed.

---

## 4. Compress pipeline

### `runPipeline` (`src/pipeline.ts:23-73`) — pure (state, config, messages, usage) → messages

```
31  Step 0: result = stripHallucinations(messages)        // strip stale/hallucinated tags
33  assignMessageRefs(state, result)                       // m0001.. stable refs
34  syncToolCache(state, result)                           // rebuild toolParameters
35  buildToolIdList(state, result)                         // ordered call ids
36-51  prune stale entries: tools not in toolParameters; messageIds not in current
      messages; nudge anchors not in current messages
52  syncCompressionBlocks(state, result)                   // reconcile blocks ↔ owning calls
55  Step 1: strategyResult = runStrategies(state, config)  // dedup + purge
58-61  Step 4.5: if compress.mode === "message": priorityMap = buildPriorityMap(...)
64  Step 5: result = injectMessageIds(state, result, priorityMap)
67  Step 6: result = injectCompressNudges(state, config, result, contextUsage, runtimePrompts)
70  Step 7: result = applyPruning(state, result)
72  return { messages: result, strategyResult }
```

### The `compress` tool — `registerCompressTool` (index.ts:198-256)

Two parameter shapes by `config.compress.mode`:

- **`message` mode (index.ts:200-227)**: `parameters = { topic, targets: [{messageId, summary}] }`,
  description = `COMPRESS_MESSAGE_PROMPT`, execute → `executeCompressTool("message", ...)`.
- **`range` mode (default, index.ts:230-255)**: `parameters = { topic, content: [{startId, endId, summary}] }`,
  description = "Compress conversation ranges into summaries. Use message IDs (m0001, m0002...)
  visible in context as boundaries.", execute → `executeCompressTool("range", ...)`.

`executeCompressTool` (174-196): if disabled → error text; else
`handleCompress(state, config, latestMessages, toolCallId, {...params, mode})`,
`sendCompressNotification(...)`, returns `{ content: [{type:"text", text: result.text}], details: {} }`.

### `handleCompress` (`src/compress/handler.ts:65-~130`)

1. `prepareCompressions(...)` resolves each entry's boundaries (see search.ts).
2. **turnProtection guard** (73-78): `getProtectedTurnStart(messages, config.turnProtection)`;
   if any entry's `endIndex >= protectedStart` → throw "overlaps the turnProtection
   protected window".
3. **Overlap guard** (79-84): sort by start; if any start ≤ previous end → throw
   "Overlapping compression selections are not allowed."
4. `allocateRunId(state)` (85), then per entry allocate a block, store state, rebuild.
5. Returns `CompressResult` (17-30): `{ text, messagesCompressed, compressedTokens,
   summaryTokens, blockIds, topic }`.

### Boundary resolution (`src/compress/search.ts`)

- `getProtectedTurnStart` (6-12): index of the Nth-newest user message.
- `resolveBoundaryIndex` (17-): `parseBoundaryId` → message ref (`m0001`) or block ref
  (`b1`). Block refs expand to the block's effective range. Throws helpful errors if the id
  was already pruned/compressed (handler.ts:238-251).
- `resolveSelection` (~182): calls `expandRangeForToolChains` (~52-109) — expands start/end
  so every toolCall has its toolResult and vice-versa (no dangling halves). Uses cached
  `assistantIndex`/`resultIndex` from `toolParameters` for O(C) lookups, falls back to scan.
  Also collects `consumedBlockIds` (blocks fully inside the range).

### Summary block structure (`src/compress/state.ts`)

- `COMPRESSED_BLOCK_HEADER = "Compressed Block"` (11).
- `wrapCompressedSummary(blockId, summary)` (30-33) →
  `[Compressed Block b1]\n${summary}\n[End Block b1]`. The `b1` ref is the model-visible
  anchor for future compress calls.
- `storeCompressionState` (59-): creates a `CompressionBlock` (state/types.ts) with
  `blockId, runId, active:true, deactivatedByUser:false, summaryTokens, mode, startIndex,
  endIndex, anchorIndex, compressToolCallId, startKey, endKey, anchorKey, summary,
  consumedBlockIds, parentBlockIds, directMessageIndices, directToolIds,
  effectiveMessageIndices, effectiveToolIds`.
- `applyCompressionState` (53-56): store then `rebuildCompressionState(state,
  getEligibleCompressionBlockIds(state))`.

### Protected content (`src/compress/protected-content.ts`)

`enrichSummaryWithProtectedContent` (imported handler.ts:15) appends, per config:

- `appendProtectedUserMessages` (27-46): if `compress.protectUserMessages`, appends every
  user message text verbatim under `[Protected User Message]`.
- `appendProtectedPromptInfo` (51-73): if `compress.protectTags`, extracts
  `<protect>…</protect>` content (regex 4) under `[Protected Content]`.
- `appendProtectedToolOutputs` (78-): appends outputs of `compress.protectedTools`
  (glob-matched) verbatim.
These guarantee the summary retains critical user intent / protected tool outputs.

### Orphan-result cleanup (`src/messages/prune.ts`)

- `filterCompressedRanges` (10-43): for each index, if an active block anchors here, push a
  synthetic `user` message with `block.summary` (24-29); skip messages covered by active
  blocks (32-36).
- `removeOrphanedToolResults` (52-71): after range filtering, a toolResult whose toolCall
  was removed becomes orphaned. Collects all toolCall ids from surviving assistant messages
  (55-64) and drops toolResults without a match (67-70). Intentionally does NOT remove
  assistant toolCalls whose result is absent — pi normalizes those with an error result
  (49-50 comment).
- `pruneToolOutputs` (80-): replaces pruned tool-result text with
  `"[Output removed to save context - information superseded or no longer needed]"` (73-74),
  but **never prunes `isError` results** (86) — diagnostics preserved.

### `applyPruning` (pipeline.ts:70, imported from prune.ts)

Pipeline Step 7 = `filterCompressedRanges` then `pruneToolOutputs` (composed in prune.ts).

---

## 5. Strategies — `src/strategies/`

### `runStrategies` (`runner.ts:21-127`) — auto-runs every context pass (pipeline.ts:55)

Guard: `state.toolIdList.length === 0` → no-op (22-24); if
`state.manualMode === "active" && !config.manualMode.automaticStrategies` → no-op (25-27).

**Deduplication** (34-88):

- `protectedTools = [...BASE_PROTECTED_TOOLS, ...config.strategies.deduplication.protectedTools]`
  (35-38). `BASE_PROTECTED_TOOLS = ["compress","write","edit","subagent"]` (config.ts).
- `turnProtection = max(config.turnProtection, config.strategies.deduplication.turnProtection)`
  (39-42).
- `unpruned = state.toolIdList.filter(id => !state.prune.tools.has(id))` (44).
- Group by `createToolSignature(tool, params)` (47-66); skip protected tool names
  (`isToolNameProtected`, 51) and protected file paths (`isFilePathProtected`, 57).
- For groups with >1 call, prune all but the last (68-86): skip if
  `state.currentUserTurn - entry.userTurn < turnProtection` (73-76); tokens =
  `status==="error" ? estimatePurgedInputSavings(params) : entry.tokenCount` (78-81).

**Purge Errors** (90-117):

- `protectedTools = [...BASE_PROTECTED_TOOLS, ...config.strategies.purgeErrors.protectedTools]`
  (92-95).
- `turnThreshold = max(config.turnProtection, config.strategies.purgeErrors.turns)` (96).
- For each unpruned error entry where `isStaleError(entry, currentUserTurn, turnThreshold)`
  (purge-errors.ts:19-29: `status==="error" && currentUserTurn - userTurn >= turnThreshold`),
  mark pruned with `estimatePurgedInputSavings` tokens (98-116).

Stats updated once: `totalPruneTokens += tokensSaved; toolsPruned += pruned` (119-122).

### `sweepAll` (`runner.ts:130`) — used by `dcp:sweep`

A force variant of `runStrategies` (same file, exported separately) that prunes all
eligible tool outputs immediately regardless of normal gating.

### `createToolSignature` (`deduplication.ts:8-11`)

`${toolName}::${JSON.stringify(normalizeParams(parameters))}`. `normalizeParams` (13-27):
recursively sorts object keys, drops `undefined` values — so argument order doesn't defeat
dedup.

### `protected-patterns.ts` — custom glob (no deps)

`globToRegex` (14-45): `*` → `[^/]*`, `**/` → `(?:.*/)?`, `?` → `[^/]`, escapes regex
meta. `isToolNameProtected` (47-55): exact or glob match. `getFilePathsFromParameters`
(57-66): only reads `parameters.filePath`. `isFilePathProtected` (68-71): any path matches
any pattern.

---

## 6. Persistence / snapshot — `src/state/`

### State shape (`state/types.ts`)

`SessionState` (8-): `sessionId, manualMode(false|"active"|"compress-pending"),
compressPermission("allow"|"deny"|undefined), pendingManualTrigger, prune{tools, messages},
nudges{contextLimitAnchors,turnAnchors,iterationAnchors}, stats{pruneTokenCounter,
totalPruneTokens, toolsPruned, messagesCompressed}, toolParameters:Map, toolIdList[],
messageIds{byRawId,byRef,byIndex,nextRefIndex}, lastCompaction, currentUserTurn,
modelContextWindow, modelId, modelProvider, compressionTiming{startTimes}, isSubAgent,
subAgentResultCache`.

`PruneMessagesState` (59-72): `byMessageIndex, blocksById, activeBlockIds,
activeByAnchorIndex, nextBlockId, nextRunId`.

`DcpSnapshotV1` (~142-183): the durable wire format — `version:1, ownerSessionId,
manualMode, compressPermission, lastCompaction, pruneTools:[ [id,tokens] ], blocks:
DcpSnapshotBlockV1[], nextBlockId, nextRunId, messageIds{byRawId:[ [key,ref] ],
nextRefIndex}, nudges{contextLimitAnchors[], turnAnchors[], iterationAnchors[]}`.

### Create / reset (`state/state.ts`)

`createSessionState()` (11-32) — fresh maps/sets, `nextBlockId=1, nextRunId=1,
nextRefIndex=1`.
`resetSessionState` (34-): clears everything **but** does NOT touch `modelContextWindow`,
`modelId`, `modelProvider`, `isSubAgent` (preserved across resets — see state-state.ts
reset body).

### Snapshot round-trip (`state/persistence.ts`)

- `serializeDcpSnapshot(state, ownerSessionId=state.sessionId)` (35-68): returns
  `DcpSnapshotV1 | undefined` (undefined if no ownerSessionId). Sorts everything for
  deterministic output (sorted helper, 8-10).
- `durableStateFingerprint(state)` (72-74): `JSON.stringify(serializeDcpSnapshot(state,
  "owner"))` — stable comparison key.
- `parseDcpSnapshot(value, warn)` (183-): validates untrusted payload; requires
  `version===1`, string `ownerSessionId`; retains valid subentries only.
- `restoreDcpSnapshot(rawSnapshot, state, currentSessionId, warn)` (286-): parse →
  `resetSessionState` → rehydrate `manualMode, compressPermission, lastCompaction,
  prune.tools, blocks, nextBlockId/nextRunId, messageIds.byRawId/byRef/nextRefIndex,
  nudges.*`. Returns boolean.

### How state syncs across pi's native compaction

- **Persist** (`persistIfChanged`, index.ts:110-122): serialize → fingerprint; if changed
  (or `force`) → `pi.appendEntry("pi-dcp-state", snapshot)` (117). This appends a custom
  entry to pi's session branch. Called after session_start (force), session_compact,
  session_shutdown, context, tool_execution_end.
- **Restore** (`restoreActiveBranch`, index.ts:132-158): walks
  `ctx.sessionManager.getBranch()` **newest-first** (139), finds the first
  `customType==="pi-dcp-state"` entry (141), `parseDcpSnapshot` (142), `restoreDcpSnapshot`
  (147-149). If the snapshot's `ownerSessionId !== currentSessionId` it's inherited
  (branch-from-parent) — restored but `force=true` returned so it re-persists under the
  new session (150-154). If no valid snapshot, just sets `state.sessionId` (156-157).
- **On `session_compact`** (index.ts:323-338): pi has just rewritten the message array
  (summaries inlined, old messages dropped). DCP clears its **index-based** caches
  (`byMessageIndex`, `activeByAnchorIndex`, `byIndex`) because indices are now stale, but
  **retains key-based maps** (`byRawId`, `byRef`, `nextRefIndex`) — content-derived keys
  survive compaction so refs stay stable. `syncCompressionBlocks` (messages/sync.ts:8) then
  re-derives indices from keys on the next context pass. This is the core of cross-compaction
  continuity.

### Tool cache (`state/tool-cache.ts`)

`syncToolCache` (13-64): scans messages; first pass collects toolResults by callId with
`isError, errorText, tokenCount, index`; second pass walks assistant toolCalls, increments
`currentUserTurn` on each user message, builds `ToolParameterEntry{tool, parameters, status
("pending"|"running"|"completed"|"error"), error, userTurn, tokenCount, assistantIndex,
resultIndex}`. `buildToolIdList` (69-): ordered assistant toolCall ids.

### `loadAllSessionStats` (persistence.ts, used by `dcp:lifetime`)

Walks sibling session dirs under the parent of `getSessionDir()` and aggregates
`totalTokensSaved/totalToolsPruned/totalMessagesCompressed/sessionCount`
(commands/lifetime.ts:3-13).

---

## 7. Commands — `src/commands/`

Registered by `registerDcpCommands` (register.ts:16-114), called once at index.ts:172.
`rejectWhenDisabled` (22-26) guards all mutating commands. Each handler calls
`onStateChange` (= `persistIfChanged`) for state-mutating ones.

| Command | register.ts line | Handler file | What it does |
| --- | --- | --- | --- |
| `dcp:compress [focus]` | 28 | compress.ts:8 | If disabled/denied → message; else `pi.sendMessage({customType:"dcp-compress-trigger", content: TRIGGER [+focus], display:false}, {triggerTurn:true, deliverAs:"followUp"})` (19-26). Sends a **hidden follow-up** asking the model to run `compress`. Nudge-based, not a hard trigger. |
| `dcp:help` | 35 | help.ts:1 | Lists all commands. |
| `dcp:context` | 42 | context.ts:3 | Reads `ctx.getContextUsage()` and prints tokens/window/percent plus `prune.tools.size`, active/total blocks, tool cache size, current user turn, manual mode. |
| `dcp:stats` | 50 | stats.ts:3 | Prints `toolsPruned, totalPruneTokens, messagesCompressed, pruneTokenCounter`. |
| `dcp:sweep` | 57 | sweep.ts:5 | Calls `sweepAll(state, config)` — force-prune all eligible tool outputs now. |
| `dcp:manual [on\|off]` | 67 | manual.ts:3 | `on` → `state.manualMode="active"` (pauses auto compression nudges); `off` → `false`; no arg → status. |
| `dcp:decompress <blockId>` | 77 | decompress.ts:4 | Sets `block.active=false, deactivatedByUser=true, deactivatedAt=now`; `rebuildCompressionState`. Original messages restored next context pass. |
| `dcp:recompress <blockId>` | 87 | recompress.ts:4 | Only if `deactivatedByUser`; re-adds to eligible set, clears flags, rebuilds. |
| `dcp:lifetime` | 97 | lifetime.ts:3 | `loadAllSessionStats(parentDir)` → aggregate across sessions. |
| `dcp:permission` | 105 | permission.ts:3 | Toggles `state.compressPermission` between `"allow"`/`"deny"`. When `deny`, `tool_call` hook blocks `compress` (index.ts:358-362) and nudges skip (inject.ts:107). |

---

## 8. UI & message injection

### `src/ui/notification.ts`

- `formatTokens` (18-23): `12400 → "~12.4K"`.
- `buildMinimalMessage` (29-32): `"DCP: ~12.4K tokens saved (5 items pruned)"` or undefined.
- `buildDetailedMessage` (38-47): minimal + deduped pruned tool names.
- `buildCompressNotificationMinimal` (53-56):
  `"DCP: ~12.4K tokens compressed (~2.1K summary, 5 messages)"`.
- `buildCompressNotificationDetailed` (62-69): minimal + `Topic:` + (if `showCompression`)
  summary text.
- Delivery chosen in index.ts:435-460: `nudgeNotificationType==="toast"` →
  `ctx.ui.notify(msg,"info")` (per-pass, only when pruned>0); else `ctx.ui.setStatus("dcp",
  msg)` (cumulative). Skipped entirely when `nudgeNotification==="off"` (435).

### Message-id injection — `src/messages/inject.ts`

- `assignMessageRefs` (22-54): clears `byIndex`; for each message derives a stable key via
  `getMessageKey` (utils/message-ids.ts:8-13): `toolResult:<toolCallId>` or
  `<role>:<timestamp>:<counter>`. Allocates `m####` refs (formatMessageRef, 15-17:
  `m${String(index).padStart(4,"0")}`) from `nextRefIndex`, storing `byRawId`/`byRef`
  (persistent) and `byIndex` (cache).
- `injectMessageIds` (63-86): for user/assistant messages, strips any stale DCP tags
  (84-85) then `appendText(cleaned, "\n\n"+tag)`. Tag = `formatMessageIdTag(ref, {priority})`
  (message-ids.ts:55-60): `<dcp-message-id>m0001</dcp-message-id>` or, in message mode,
  `<dcp-message-id priority="3">m0001</dcp-message-id>`.
- `injectCompressNudges` (97-196): see §2/§3.
- `applyAnchoredNudges` (261-296): appends nudge text at anchored indices (only if not
  already present — `hasExistingNudge` 298-311).

### Stripping — `src/messages/strip.ts`

`stripHallucinationsFromString` (20-26) runs four regexes in order (order matters —
complete pairs consume the closing `>` first):

1. `DCP_COMPLETE_PAIR` (5): `<dcp-foo …>…</dcp-foo>`.
2. `DCP_TRUNCATED_PAIR` (7): `<dcp-foo>…</dcp-foo` (no final `>`).
3. `DCP_UNPAIRED_TAG` (9): `</dcp-foo>` or `<dcp-foo>`.
4. `DCP_PARTIAL_TAG` (12, multiline): `<dcp-message-id` or `</dcp` at end of line (uses
   `[^\S\n]` so attributes don't cross lines).
`stripHallucinations` (32-37): applies to **assistant** messages only (33-34). Called both
in the pipeline (Step 0, pipeline.ts:31) and in `message_end` (index.ts:348) — belt and
suspenders: pipeline strips from the assembled context; `message_end` strips from the
just-finalized assistant message before it's stored.

### `sync.ts` — `syncCompressionBlocks` (8-65)

Reconciles blocks with owning assistant tool calls: builds `keyToIndex` from
`messageIds.byIndex`/`byRef`; for each block, resolves `startKey/endKey/anchorKey` to
indices; **deletes blocks whose keys are gone or whose `compressToolCallId` is no longer in
`toolParameters`** (23-32) — this is how compression blocks vanish when pi compacts away
their owning call. Rebuilds `parentBlockIds`, `consumedBlockIds`, `directMessageIndices`,
`effectiveMessageIndices/ToolIds`, then calls `rebuildCompressionState`.

### `priority.ts` — `buildPriorityMap` (23-71)

Message-mode only. Score = `positionScore*0.6 + tokenScore*0.3 + roleWeight` (41-47):
position `(len-i)/len`, token `min(tokens/500,1)`, role `toolResult?0.2:0`. Sorted desc,
assigned priority 1-5 by quintiles (54-68). Surfaced as `priority` attr on the message-id
tag so the model knows what to compress first.

---

## 9. Config — `src/config-schema.ts` + `dcp.schema.json`

TypeBox schemas; `dcp.schema.json` is the JSON Schema emitted from the same source of truth
(README "What's new in 0.4.0": "Config validation and the shipped `dcp.schema.json` now come
from the same TypeBox source of truth").

### Top-level `DcpConfigSchema` (config-schema.ts:140)

| Field | line | type / default | Semantics |
| --- | --- | --- | --- |
| `enabled` | 141 | bool, true | Master kill switch. |
| `debug` | 145 | bool, false | Debug logging to `<sessionDir>/dcp/logs`. |
| `nudgeNotification` | 149 | `"off"\|"minimal"\|"detailed"`, `minimal` | Pruning-event notification verbosity. |
| `nudgeNotificationType` | (after 149) | `"toast"\|"status"`, `status` | toast=ephemeral per-pass; status=persistent cumulative. |
| `protectedFilePatterns` | (top-level) | string[], [] | Globs for file paths protected from pruning. |
| `turnProtection` | (top-level) | int≥0, 0 | Protect newest N user turns from ALL DCP transforms (compress + strategies). |
| `compress` | — | CompressConfig | see below |
| `manualMode` | 113 | ManualModeConfig | see below |
| `strategies` | 135 | StrategiesConfig | dedup + purgeErrors |
| `experimental` | 124 | ExperimentalConfig | see below |

### `CompressConfigSchema` (36-111)

| Field | line | default | Semantics |
| --- | --- | --- | --- |
| `mode` | 37 | `"range"` | `range`=compress spans {startId,endId,summary}; `message`=compress individual messages {messageId,summary}. Changes tool param shape (index.ts:200 vs 230). |
| `permission` | 42 | `"allow"` | `deny` blocks the `compress` tool (tool_call hook, 358) and disables nudges. |
| `showCompression` | 46 | false | Include summary text in user notifications (UI only, not model context). |
| `maxContextPercent` | 51 | 80 | Legacy % threshold (fallback when no absolute limit). |
| `minContextPercent` | 55 | 50 | Legacy % min threshold. |
| `maxContextLimit` | 59 | 200000 (config.ts) | Absolute token max, or `"80%"` string. Resolution tier 2. |
| `minContextLimit` | 65 | 100000 (config.ts) | Absolute token min, or `"50%"`. Nudges only fire past this. |
| `modelMaxLimits` | 71 | — | `Record<string, number\|string>` keyed `provider/modelId`. Tier 1 (highest precedence). |
| `modelMinLimits` | 76 | — | Per-model min overrides. |
| `nudgeFrequency` | 81 | 5 (min 1) | Min messages between non-urgent (turn/iteration) nudges. |
| `iterationNudgeThreshold` | 86 | 15 (min 1) | Trailing assistant messages without user input before iteration nudge. |
| `nudgeForce` | 91 | `"soft"` | **⚠ Vestigial.** Declared `strong\|soft` but grep shows NO consumer in src/ — nudges are always imperative text regardless. Do not rely on this field. |
| `protectedTools` | 95 | [] (+ `["compress"]` forced in config.ts) | Tool outputs preserved during compression (glob). `compress` always protected to prevent recursive compression. |
| `protectUserMessages` | 99 | false | Append user message text to summaries. |
| `protectTags` | 103 | false | Preserve `<protect>…</protect>` content in summaries. |
| `summaryBuffer` | 107 | true | Exclude active summary tokens from threshold comparison (prevents cascading). |

### `ManualModeConfigSchema` (113-122)

`default` (114): `false` or `"active"` — initial manual mode. `automaticStrategies` (118,
default true): run dedup/purge even in manual mode.

### `ExperimentalConfigSchema` (124-133)

`allowSubAgents` (125, false): enable DCP in sub-agent child sessions (detected via
`PI_SUBAGENT_CHILD==="1"`, index.ts:319). `customPrompts` (129, false): enable filesystem
prompt overrides.

### `DeduplicationConfigSchema` (3-18)

`enabled` (4, true); `protectedTools` (8, [], glob patterns excluded from dedup);
`turnProtection` (12, 0, min 0): protect duplicate outputs for N turns.

### `PurgeErrorsConfigSchema` (20-34)

`enabled` (21, true); `turns` (25, 4, min 1): prune failed inputs after this many turns;
`protectedTools` (30, [], globs excluded from purge).

---

## 10. CRITICAL OBSERVATION — hard-trigger vs nudge-only

**Answer: pi-dcp is strictly nudge-only. It NEVER programmatically triggers compaction.**

Evidence (every code path that could force compaction):

1. **`pi.on("context")`** (index.ts:395-463) — the hot path. It calls `runPipeline` (412)
   and `return { messages: result.messages }` (462). It transforms the message array pi
   already assembled; it does **not** call any `pi.compact()`/`pi.summarize()`/similar API.
   No compaction invocation anywhere in the handler.

2. **`pi.on("session_compact")`** (index.ts:323-338) — the name is misleading. This is a
   **reactive listener**: it fires *after* pi performs its own native compaction. DCP's
   response is to clear index-based caches and re-persist (324-337). It is not a trigger;
   it is a "compaction happened, clean up" handler. The log line literally says
   "compaction detected" (336).

3. **Nudges inject TEXT, not actions** (`injectCompressNudges`, inject.ts:97-196). When
   over limits, it appends a `<dcp-system-reminder>` block (nudges.ts:6) whose content is
   the string "You MUST use the `compress` tool now." The model must *choose* to call the
   tool. There is no `pi.callTool("compress", …)` or forced tool invocation.

4. **`dcp:compress` command** (compress.ts:19-26) — the closest thing to a manual trigger.
   It does **not** call compact either; it calls
   `pi.sendMessage({customType:"dcp-compress-trigger", content: TRIGGER, display:false},
   {triggerTurn:true, deliverAs:"followUp"})` — i.e. it sends a **hidden follow-up user
   message** instructing the model to run the compress tool. README confirms: "It sends Pi
   a hidden follow-up that asks it to use the `compress` tool; it does nothing while DCP or
   compression permission is disabled." Still nudge-based.

5. **The `compress` tool itself** (index.ts:198-256, handler.ts:65) — only executes when
   the model calls it voluntarily. It replaces message ranges with summaries in DCP's
   internal state (compression blocks), which `applyPruning` (pipeline.ts:70) later
   realizes into the context. This is DCP's *own* compression mechanism (range→summary),
   entirely distinct from pi's native compaction. It does not invoke pi's compaction.

6. **No `compact`/`summarize`/`pi.compact` symbol** appears in the source. A grep for
   `compact` in src/ returns only: the `session_compact` hook listener (index.ts:323), the
   log string "compaction detected" (336), and `lastCompaction` state field — all reactive.

**Conclusion:** pi-dcp's entire leverage over compaction is (a) a system prompt that tells
the model `compress` is its only context tool, (b) injected reminder text when thresholds
are crossed, and (c) a hidden follow-up message from the `dcp:compress` command. If the
model ignores the nudge, no compaction happens. A production imitation that wants a hard
guarantee must add a programmatic compaction call (e.g. `pi.compact()` if the API exposes
one) — pi-dcp deliberately does not.

---

## 11. Surprises / gotchas for an imitation

- **`nudgeForce` is dead config** (config-schema.ts:91) — declared, validated, never read.
  The nudges are always imperative. An imitation should either wire it or drop it.
- **`message_end` double-strips** — `stripHallucinations` runs both in the pipeline (on
  the assembled context, pipeline.ts:31) and in `message_end` (on the stored message,
  index.ts:348). Belt-and-suspenders against the model emitting fresh tags mid-turn.
- **`session_compact` retains key maps, drops index maps** (index.ts:329-332) — the
  content-derived keys (`byRawId`/`byRef`) survive pi's compaction so `m0001` keeps meaning
  the same message even after pi rewrites the array. `syncCompressionBlocks`
  (messages/sync.ts:8) re-derives indices from keys next pass. This is the linchpin of
  cross-compaction stability.
- **`summaryBuffer`** (config.ts, inject.ts:117-132) — subtracts active summary tokens
  before re-evaluating `overMax`, so a session that just compressed doesn't immediately
  re-trigger because the summaries themselves pushed tokens up. Prevents nudge storms.
- **Block refs are first-class** — `b1` is a valid boundary id (parseBoundaryId,
  message-ids.ts:45-53), so the model can compress "everything up to block b2" into a new
  nested block. `consumedBlockIds`/`parentBlockIds` track nesting (sync.ts:45-63).
- **`getFilePathsFromParameters` only reads `parameters.filePath`**
  (protected-patterns.ts:62) — tools that use `path`/`file`/`files` args aren't
  file-protected. Narrow by design.
- **Sub-agent gating via env var** — `PI_SUBAGENT_CHILD==="1"` (index.ts:319) plus
  `experimental.allowSubAgents`. Sub-agents are off by default.
- **`compress` tool is always protected from pruning** (config.ts DEFAULT_CONFIG forces
  `protectedTools=["compress"]`) so a compression block's owning call is never deduped
  away.
- **`dcp:compress` is nudge-based** (compress.ts:19-26) — even the "manual trigger" just
  sends a hidden message. This is the clearest expression of the nudge-only philosophy.
