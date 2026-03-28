<purpose>
Autonomously verify built features using Playwright browser automation. Extracts tests from SUMMARY.md files (same as verify-work), then executes each test via Playwright — navigating pages, performing actions, taking screenshots, and evaluating pass/fail based on observable DOM state.

Produces the same UAT.md format as verify-work for full compatibility with downstream GSD commands (diagnose-issues, plan-phase --gaps).
</purpose>

<available_agent_types>
Valid GSD subagent types (use exact names — do not fall back to 'general-purpose'):
- gsd-planner — Creates detailed plans from phase scope
- gsd-plan-checker — Reviews plan quality before execution
</available_agent_types>

<philosophy>
**Autonomous verification through browser automation.**

Claude navigates the app, performs each test action via Playwright, observes the result (DOM snapshots, screenshots, console errors, network failures), and records pass/fail.

- No user interaction during test execution
- Each test: navigate → act → observe → record
- Screenshots saved as evidence for every test
- Console errors and network failures captured as additional signals
- Issues detected by comparing expected behavior against observed DOM state
</philosophy>

<template>
@$HOME/.claude/get-shit-done/templates/UAT.md
</template>

<process>

<step name="parse_arguments" priority="first">
Parse $ARGUMENTS for:
- **Phase number** (required): first numeric argument
- **--base-url=URL** (optional): application URL, default `http://localhost:3000`

If no phase number provided:
```
Phase number required.

Usage: /gsd:auto-verify 4
       /gsd:auto-verify 4 --base-url=http://localhost:5173
```
Exit.
</step>

<step name="initialize">
Load phase context:

```bash
INIT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" init verify-work "${PHASE_NUM}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
AGENT_SKILLS_PLANNER=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" agent-skills gsd-planner 2>/dev/null)
AGENT_SKILLS_CHECKER=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" agent-skills gsd-checker 2>/dev/null)
```

Parse JSON for: `planner_model`, `checker_model`, `commit_docs`, `phase_found`, `phase_dir`, `phase_number`, `phase_name`, `has_verification`, `uat_path`.

If `phase_found` is false, display error and exit.

Create screenshots directory:
```bash
mkdir -p "${phase_dir}/screenshots"
```
</step>

<step name="preflight_check">
**Verify the application is reachable before running tests:**

Navigate to {base_url}:
```
mcp__playwright__browser_navigate(url="{base_url}")
```

Take a snapshot to confirm the page loaded:
```
mcp__playwright__browser_snapshot()
```

**If navigation fails or page is empty/error:**
```
╔══════════════════════════════════════════════════════════════╗
║  ERROR                                                       ║
╚══════════════════════════════════════════════════════════════╝

Application not reachable at {base_url}

**To fix:**
1. Start the application server
2. Retry: /gsd:auto-verify {phase_num} --base-url={base_url}
```
Exit.

**If page loads successfully:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► AUTO-VERIFY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Phase {phase_number}: {phase_name}
 Base URL: {base_url}
 Mode: Autonomous (Playwright)

 ◆ Application reachable — starting verification...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
</step>

<step name="find_summaries">
**Find what to test:**

```bash
ls "${phase_dir}"/*-SUMMARY.md 2>/dev/null || true
```

Read each SUMMARY.md to extract testable deliverables.
</step>

<step name="extract_tests">
**Extract testable deliverables from SUMMARY.md:**

Parse for:
1. **Accomplishments** — features/functionality added
2. **User-facing changes** — UI, workflows, interactions

For each deliverable, create a test with:
- **name**: brief test name
- **expected**: what should be observable in the browser (specific DOM elements, text, behavior)
- **actions**: sequence of Playwright actions to perform the test (navigate, click, type, wait)
- **verify**: what to check in the DOM snapshot or screenshot to confirm pass/fail

Focus on USER-OBSERVABLE outcomes. Skip internal/non-observable items (refactors, type changes).

**Test action planning:** For each test, determine:
1. **URL path** to navigate to (relative to base_url)
2. **Setup actions** (login if needed, navigate to correct page)
3. **Test actions** (click buttons, fill forms, trigger features)
4. **Verification criteria** (text visible, element present, correct state, no console errors)

**Cold-start smoke test injection:**

After extracting tests from SUMMARYs, scan for server/startup file paths. If found, prepend:
- name: "Cold Start Smoke Test"
- actions: Navigate to base_url, check page loads, verify no console errors
- verify: Page renders content, no error states, HTTP 200

**Authentication handling:**

Scan SUMMARY files and test expectations for auth-related patterns (login, dashboard, profile, admin, protected).
If ANY test requires authentication:
- Add an auth setup step before the first test that needs it
- Use project's default test credentials if available in CLAUDE.md or .env
- Record the auth method used in UAT.md frontmatter
</step>

<step name="create_uat_file">
**Create UAT file with all tests:**

```bash
mkdir -p "${phase_dir}"
```

Create file at `${phase_dir}/{phase_num}-UAT.md`:

```markdown
---
status: testing
phase: {XX-name}
source: [list of SUMMARY.md files]
mode: autonomous
base_url: {base_url}
started: [ISO timestamp]
updated: [ISO timestamp]
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

number: 1
name: [first test name]
expected: |
  [what should be observable]
actions: |
  [playwright actions to perform]
status: executing

## Tests

### 1. [Test Name]
expected: [observable behavior]
actions: [playwright action sequence]
result: [pending]

### 2. [Test Name]
expected: [observable behavior]
actions: [playwright action sequence]
result: [pending]

...

## Summary

total: [N]
passed: 0
issues: 0
pending: [N]
skipped: 0
blocked: 0

## Gaps

[none yet]
```

Proceed to `execute_tests`.
</step>

<step name="execute_tests">
**Execute each test autonomously via Playwright:**

For each test in order:

### 1. Update Current Test
Update UAT.md Current Test section with current test info and `status: executing`.

### 2. Display Progress
```
◆ Test {N}/{total}: {test_name}
```

### 3. Capture Pre-State
Check console messages before test:
```
mcp__playwright__browser_console_messages()
```

### 4. Navigate
```
mcp__playwright__browser_navigate(url="{base_url}{test_path}")
```

### 5. Perform Actions
Execute the planned Playwright action sequence for this test. Common patterns:

**Click a button/link:**
```
mcp__playwright__browser_click(element="Button text or selector")
```

**Fill a form:**
```
mcp__playwright__browser_fill_form(formData=[{"selector": "input[name=email]", "value": "test@example.com"}])
```

**Type text:**
```
mcp__playwright__browser_type(element="selector", text="content")
```

**Wait for content:**
```
mcp__playwright__browser_wait_for(selector=".expected-element", timeout=5000)
```

**Select dropdown:**
```
mcp__playwright__browser_select_option(element="select", values=["option"])
```

### 6. Observe Result
Take a DOM snapshot to analyze page state:
```
mcp__playwright__browser_snapshot()
```

Take a screenshot for evidence:
```
mcp__playwright__browser_take_screenshot()
```

Check for new console errors:
```
mcp__playwright__browser_console_messages()
```

### 7. Evaluate Pass/Fail

**PASS criteria (ALL must be true):**
- Expected DOM elements are present in snapshot
- Expected text/content is visible
- No new console errors (warnings are acceptable)
- No network failures for critical requests
- Page is not in an error state (no error boundaries, 404/500 pages)

**ISSUE detection (ANY triggers issue):**
- Expected element missing from DOM
- Wrong text/content displayed
- JavaScript errors in console
- Network request failures (4xx/5xx for app routes)
- Error boundary or crash screen visible
- Timeout waiting for expected element

**BLOCKED detection:**
- Server unreachable (connection refused)
- Authentication required but no credentials available
- Feature depends on external service not running

### 8. Record Result

**If PASS:**
```
### {N}. {name}
expected: {expected}
result: pass
screenshot: screenshots/{phase_num}-test-{N}-pass.png
observed: "{brief description of what was observed in DOM}"
```

Display: `  ✓ Pass`

**If ISSUE:**

Infer severity from observation:
- Console errors, crash screens, error boundaries → blocker
- Missing functionality, wrong behavior, broken UI → major
- Slow loading, minor visual glitch, non-critical → minor
- Spacing, alignment, color differences → cosmetic

```
### {N}. {name}
expected: {expected}
result: issue
screenshot: screenshots/{phase_num}-test-{N}-issue.png
observed: "{what was actually seen}"
severity: {inferred}
console_errors: [any JS errors captured]
```

Append to Gaps section:
```yaml
- truth: "{expected behavior}"
  status: failed
  reason: "Auto-verify observed: {what was actually seen}"
  severity: {inferred}
  test: {N}
  screenshot: "screenshots/{phase_num}-test-{N}-issue.png"
  console_errors: [{errors if any}]
  artifacts: []
  missing: []
```

Display: `  ✗ Issue ({severity}): {brief observation}`

**If BLOCKED:**
```
### {N}. {name}
expected: {expected}
result: blocked
blocked_by: {server|auth|third-party|other}
reason: "{why blocked}"
screenshot: screenshots/{phase_num}-test-{N}-blocked.png
```

Display: `  ○ Blocked: {reason}`

### 9. Update Summary
Update counts in Summary section after each test.

### 10. Continue
If more tests remain → next test.
If all tests done → proceed to `complete_session`.
</step>

<step name="complete_session">
**Complete testing and report results:**

Update frontmatter:
- status: complete
- updated: [now]

Clear Current Test:
```
## Current Test

[auto-verification complete]
```

Commit:
```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" commit "test({phase_num}): auto-verify UAT - {passed} passed, {issues} issues" --files ".planning/phases/{XX-name}/{phase_num}-UAT.md" ".planning/phases/{XX-name}/screenshots/"
```

Display results:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► AUTO-VERIFY COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Phase {phase_number}: {phase_name}

 | Result  | Count |
 |---------|-------|
 | Passed  | {N}   |
 | Issues  | {N}   |
 | Blocked | {N}   |
 | Skipped | {N}   |

 {PROGRESS_BAR} {pct}% passed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**If issues > 0:** Display each issue briefly, then proceed to `diagnose_issues`.
**If issues == 0:**
```
All tests passed autonomously.

───────────────────────────────────────────────────────────────

## ▶ Next Up

- `/gsd:plan-phase {next}` — Plan next phase
- `/gsd:execute-phase {next}` — Execute next phase
- `/gsd:verify-work {phase}` — Manual re-verify if needed
- `/gsd:ui-review {phase}` — Visual quality audit

───────────────────────────────────────────────────────────────
```
</step>

<step name="diagnose_issues">
**Diagnose root causes before planning fixes:**

```
───────────────────────────────────────────────────────────────

{N} issues found. Diagnosing root causes...

◆ Spawning parallel debug agents to investigate each issue.
```

Follow @$HOME/.claude/get-shit-done/workflows/diagnose-issues.md:
- Spawn parallel debug agents for each issue
- Include screenshot paths and console errors as additional context
- Collect root causes
- Update UAT.md Gaps section with diagnoses
- Proceed to `plan_gap_closure`
</step>

<step name="plan_gap_closure">
**Auto-plan fixes from diagnosed gaps:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PLANNING FIXES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

◆ Spawning planner for gap closure...
```

Spawn gsd-planner in --gaps mode:

```
Task(
  prompt="""
<planning_context>

**Phase:** {phase_number}
**Mode:** gap_closure

<files_to_read>
- {phase_dir}/{phase_num}-UAT.md (UAT with diagnoses and screenshots)
- .planning/STATE.md (Project State)
- .planning/ROADMAP.md (Roadmap)
</files_to_read>

${AGENT_SKILLS_PLANNER}

</planning_context>

<downstream_consumer>
Output consumed by /gsd:execute-phase
Plans must be executable prompts.
</downstream_consumer>
""",
  subagent_type="gsd-planner",
  model="{planner_model}",
  description="Plan gap fixes for Phase {phase}"
)
```

On return:
- **PLANNING COMPLETE:** Proceed to `verify_gap_plans`
- **PLANNING INCONCLUSIVE:** Report and offer manual intervention
</step>

<step name="verify_gap_plans">
**Verify fix plans with checker:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► VERIFYING FIX PLANS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

◆ Spawning plan checker...
```

Initialize: `iteration_count = 1`

Spawn gsd-plan-checker:

```
Task(
  prompt="""
<verification_context>

**Phase:** {phase_number}
**Phase Goal:** Close diagnosed gaps from auto-verify UAT

<files_to_read>
- {phase_dir}/*-PLAN.md (Plans to verify)
</files_to_read>

${AGENT_SKILLS_CHECKER}

</verification_context>

<expected_output>
Return one of:
- ## VERIFICATION PASSED — all checks pass
- ## ISSUES FOUND — structured issue list
</expected_output>
""",
  subagent_type="gsd-plan-checker",
  model="{checker_model}",
  description="Verify Phase {phase} fix plans"
)
```

On return:
- **VERIFICATION PASSED:** Proceed to `present_ready`
- **ISSUES FOUND:** Proceed to `revision_loop`
</step>

<step name="revision_loop">
**Iterate planner <> checker until plans pass (max 3):**

**If iteration_count < 3:**

Display: `Sending back to planner for revision... (iteration {N}/3)`

Spawn gsd-planner with revision context (same as verify-work revision_loop).

After planner returns, spawn checker again.
Increment iteration_count.

**If iteration_count >= 3:**

Display: `Max iterations reached. {N} issues remain.`

Offer options:
1. Force proceed (execute despite issues)
2. Run manual /gsd:verify-work for human judgment
3. Abandon (exit)
</step>

<step name="present_ready">
**Present completion and next steps:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► FIXES READY ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {X}: {Name}** — {N} gap(s) diagnosed, {M} fix plan(s) created

| Gap | Root Cause | Fix Plan |
|-----|------------|----------|
| {truth 1} | {root_cause} | {phase}-04 |
| {truth 2} | {root_cause} | {phase}-04 |

Plans verified and ready for execution.

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Execute fixes** — run fix plans

`/clear` then `/gsd:execute-phase {phase} --gaps-only`

───────────────────────────────────────────────────────────────
```
</step>

</process>

<update_rules>
**Write to UAT.md after every test** (not batched — autonomous mode has no risk of interruption):

| Section | Rule | When Written |
|---------|------|--------------|
| Frontmatter.status | OVERWRITE | Start, complete |
| Frontmatter.updated | OVERWRITE | After each test |
| Current Test | OVERWRITE | Before each test |
| Tests.{N}.result | OVERWRITE | After each test |
| Summary | OVERWRITE | After each test |
| Gaps | APPEND | When issue found |
</update_rules>

<success_criteria>
- [ ] Application reachable via preflight check before testing
- [ ] UAT file created with all tests from SUMMARY.md
- [ ] Each test executed autonomously via Playwright
- [ ] Screenshots captured for every test result
- [ ] Console errors and network failures checked
- [ ] Pass/fail determined from DOM state observation
- [ ] Severity inferred from observation type (never asked)
- [ ] UAT.md written after every test for crash safety
- [ ] Compatible with downstream GSD commands (same UAT format)
- [ ] If issues: parallel debug agents diagnose root causes
- [ ] If issues: gsd-planner creates fix plans (gap_closure mode)
- [ ] If issues: gsd-plan-checker verifies fix plans
- [ ] Ready for `/gsd:execute-phase --gaps-only` when fixes planned
</success_criteria>
