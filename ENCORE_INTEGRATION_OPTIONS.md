# Encore Test Integration - Options for Reporting Results Back to IDE

## Current State

✅ **Working:** Play button executes `encore test run` in terminal
❌ **Missing:** Test results (✅/❌) don't show up in VS Code UI

## The Problem

We bypassed the entire Vitest worker RPC chain:
```
BEFORE: api.runFiles() → RPC → Worker → Vitest Core → emits onTaskUpdate → Runner marks tests
NOW:    api.runFiles() → terminal.sendText('encore test run') → ??? (no feedback to IDE)
```

## How VS Code Test Results Work

**The Flow:**
1. Runner creates `testRun` (VS Code TestRun object)
2. Worker executes tests and emits events: `onTaskUpdate(testId, result)`
3. Runner listens to events and calls:
   - `testRun.passed(testItem, duration)` → ✅
   - `testRun.failed(testItem, errors, duration)` → ❌
   - `testRun.skipped(testItem)` → ⊘

**Key Objects We Need:**
- `testRun` - VS Code TestRun instance (available in `runner.ts`)
- `testItem` - VS Code TestItem for each test (available via `this.tree.getTestItemByTask(task)`)
- `result` - Test result with state ('pass'/'fail'), errors, duration

## Architecture Context

**runner.ts** - Has everything we need:
- ✅ `this.testRun` - The VS Code TestRun object
- ✅ `this.tree` - TestTree with `getTestItemByTask()` to find test items
- ✅ `request` - The TestRunRequest with list of tests to run
- ✅ Methods to mark results: `markResult()`, `markTestCase()`

**api.ts** - Current location of encore command:
- ❌ No access to `testRun`
- ❌ No access to `tree`
- ✅ Has the file list and testNamePattern

## Options to Report Results

### Option 1: Simple Exit Code (Quick & Dirty)

**Approach:** Mark ALL tests as passed/failed based on exit code

**Pros:**
- ✅ Super simple - no parsing needed
- ✅ Quick to implement (5 min)
- ✅ Shows some feedback in UI

**Cons:**
- ❌ Inaccurate - if 1 test fails, marks ALL as failed
- ❌ No error messages
- ❌ No individual test status

**Implementation:**
```typescript
// In runner.ts - modify scheduleTestItems
const exitCode = await this.api.runFiles(files, testNamePattern)
if (exitCode === 0) {
  tests.forEach(test => this.testRun.passed(test))
}
else {
  tests.forEach(test => this.testRun.failed(test, [new vscode.TestMessage('Test failed')]))
}
```

---

### Option 2: Parse Encore Output (Accurate)

**Approach:** Capture stdout, parse test results, mark each test individually

**Pros:**
- ✅ Accurate per-test results
- ✅ Can show error messages
- ✅ Proper ✅/❌ for each test
- ✅ Can extract durations

**Cons:**
- ❌ Need to parse text output (brittle)
- ❌ Output format might change
- ❌ More complex (~30-60 min)

**Encore Output Format:**
```
✓ authenticated - should update profile first name 1010ms
✓ authenticated - should update profile last name 781ms
× authenticated - should NOT update email or phone 818ms
  → expected true to be false
```

**Implementation:**
```typescript
// In api.ts - capture output instead of inherit
const child = spawn('encore', ['test', 'run', ...files], {
  stdio: ['ignore', 'pipe', 'pipe'],
  shell: true,
})

let stdout = ''
child.stdout.on('data', data => stdout += data.toString())

child.on('close', (code) => {
  const results = parseEncoreOutput(stdout)
  // Pass results back to runner via callback or return value
})
```

Then in runner, map test names to test items and mark them.

---

### Option 3: Move Command Execution to Runner (Clean Architecture)

**Approach:** Keep encore command in runner where we have access to testRun and tree

**Pros:**
- ✅ Clean architecture - runner has all the context
- ✅ Can directly call `testRun.passed/failed`
- ✅ Can look up test items from tree
- ✅ Can parse or use exit code easily

**Cons:**
- ❌ Breaks current api.runFiles() pattern
- ❌ More refactoring needed

**Implementation:**
```typescript
// In runner.ts - in scheduleTestItems()
async scheduleTestItems(request, token) {
  // ... existing code ...

  // Instead of calling this.api.runFiles()
  const command = `encore test run ${testNamePattern} ${files.join(' ')}`
  const terminal = vscode.window.createTerminal('Encore Tests')
  terminal.sendText(command)

  // Then mark tests based on result
  // (still need to solve how to know when done and what results are)
}
```

Still needs solution for getting results back.

---

### Option 4: Hybrid - Use Encore JSON Reporter (If Available)

**Approach:** Check if encore supports JSON output format

**Example:**
```bash
encore test run --reporter=json
```

**Pros:**
- ✅ Structured data - easy to parse
- ✅ Accurate results
- ✅ No brittle text parsing

**Cons:**
- ❌ Need to check if encore supports this
- ❌ If not available, back to option 2

---

## Recommendation

**Phase 1 (Now):** Option 1 - Simple exit code
- Get basic feedback working quickly
- Unblock development
- Shows that approach works

**Phase 2 (Later):** Option 2 or 4 - Parse output or use JSON
- Implement proper per-test results
- Add error messages
- Improve UX

## Next Steps

1. Check if encore supports `--reporter=json` or similar
2. If yes → implement Option 4
3. If no → implement Option 1 now, Option 2 later
4. Move execution to runner (Option 3) for cleaner architecture

## Key Code Locations

- **Play button handler:** `extension.ts:188` → `runner.runTests()`
- **Test scheduling:** `runner.ts:285` → `scheduleTestItems()`
- **API call:** `runner.ts:326` → `api.runFiles()`
- **Current encore execution:** `api.ts:177` → `runFiles()`
- **Test marking:** `runner.ts:536` → `markTestCase()`
