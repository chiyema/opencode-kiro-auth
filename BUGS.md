# Bug Tracking: 400 Improperly Formed Request

## Status: RESOLVED

All bugs below have been fixed and verified. All 16 supported models (4.5 and 4.6 variants
including 1m and thinking) return 200. Plugin hot-swapped and confirmed working after restart.

---

## Bug 1 — Orphaned tool result injection bypasses sanitization ✅ FIXED

**File**: `src/plugin/request.ts` line 184  
**Severity**: High — primary root cause of 400s in multi-turn tool conversations

### Root Cause

Guard condition `if (prev && !prev.userInputMessage)` was wrong when history was empty
(`prev` is undefined), skipping the bridging `userInputMessage` before the synthetic
`assistantResponseMessage`. API rejected the malformed sequence.

### Fix Applied

```ts
// Before (wrong)
if (prev && !prev.userInputMessage) {

// After (correct)
if (!prev || prev.assistantResponseMessage) {
```

---

## Bug 2 — No guard against consecutive `assistantResponseMessage` entries ✅ FIXED

**File**: `src/infrastructure/transformers/history-builder.ts` lines 111–121  
**Severity**: Medium

### Root Cause

No guard before pushing `assistantResponseMessage` in `buildHistory`. Two adjacent assistant
messages could be produced when an empty-content assistant message was skipped, violating
the API's strict alternating sequence requirement.

### Fix Applied

Added guard that inserts a `userInputMessage("Continue")` if the previous history entry is
also an `assistantResponseMessage`.

---

## Bug 3 — `sanitizeHistory` nukes entire history on bad first message ✅ FIXED

**File**: `src/infrastructure/transformers/message-transformer.ts` lines 23–28  
**Severity**: Medium

### Root Cause

`return []` when first message had `toolResults` wiped the entire history silently, losing
all conversation context.

### Fix Applied

```ts
// Before: return [] on bad first message
// After: strip leading invalid messages until valid starting point found
while (result.length > 0) {
  const first = result[0]
  if (first?.userInputMessage && !first.userInputMessage.userInputMessageContext?.toolResults) break
  result.shift()
}
if (result.length === 0) return []
```

---

## Bug 4 — Wrong model IDs causing `ValidationException` ✅ FIXED

**File**: `src/constants.ts`  
**Severity**: High — caused 100% of observed 400s in production logs

### Root Cause

Old uppercase model IDs (`CLAUDE_HAIKU_4_5_20251001_V1_0`, `CLAUDE_SONNET_4_5_20250929_V1_0`,
etc.) were being rejected by the API with `ValidationException`. The title generator
(using `claude-haiku-4-5`) was the most frequent offender, firing on every session.

### Fix Applied

Updated all model IDs to the dotted format accepted by the API:

| Plugin Key | Before | After |
|---|---|---|
| `claude-haiku-4-5` | `CLAUDE_HAIKU_4_5_20251001_V1_0` | `claude-haiku-4.5` |
| `claude-sonnet-4-5` | `CLAUDE_SONNET_4_5_20250929_V1_0` | `claude-sonnet-4.5` |
| `claude-sonnet-4-5-1m` | `CLAUDE_SONNET_4_5_20250929_LONG_V1_0` | `claude-sonnet-4.5-1m` |
| `claude-sonnet-4-6-1m` | `claude-sonnet-4.6` | `claude-sonnet-4.6-1m` |
| `claude-opus-4-5` | `CLAUDE_OPUS_4_5_20251101_V1_0` | `claude-opus-4.5` |
| `claude-opus-4-6-1m` | `claude-opus-4.6` | `claude-opus-4.6-1m` |
| `claude-sonnet-4` | `CLAUDE_SONNET_4_20250514_V1_0` | `claude-sonnet-4` |

---

## Bug 5 — Unused `better-sqlite3` devDependency ✅ FIXED

**File**: `package.json`  
**Severity**: Low — unnecessary bloat

### Root Cause

`better-sqlite3` and `@types/better-sqlite3` were listed as devDependencies but the codebase
uses `bun:sqlite` exclusively. Removed both, saving 39 packages.

---

## Verified Working Models

All tested after fixes, confirmed 200 responses in logs:

| Model | Status |
|---|---|
| claude-haiku-4-5 | ✅ |
| claude-haiku-4-5-thinking | ✅ |
| claude-sonnet-4-5 | ✅ |
| claude-sonnet-4-5-thinking | ✅ |
| claude-sonnet-4-5-1m | ✅ |
| claude-sonnet-4-5-1m-thinking | ✅ |
| claude-sonnet-4-6 | ✅ |
| claude-sonnet-4-6-thinking | ✅ |
| claude-sonnet-4-6-1m | ✅ |
| claude-sonnet-4-6-1m-thinking | ✅ |
| claude-opus-4-5 | ✅ |
| claude-opus-4-5-thinking | ✅ |
| claude-opus-4-6 | ✅ |
| claude-opus-4-6-thinking | ✅ |
| claude-opus-4-6-1m | ✅ |
| claude-opus-4-6-1m-thinking | ✅ |

## Not Fixed / Out of Scope

- `claude-3-7-sonnet` — `ValidationException` regardless of model ID format. Model not
  available on this account.
