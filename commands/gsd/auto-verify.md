---
name: gsd:auto-verify
description: Autonomously verify built features using Playwright browser automation
argument-hint: "[phase number, e.g., '4'] [--base-url=http://localhost:3000]"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Edit
  - Write
  - Task
  - mcp__playwright__browser_navigate
  - mcp__playwright__browser_click
  - mcp__playwright__browser_snapshot
  - mcp__playwright__browser_take_screenshot
  - mcp__playwright__browser_fill_form
  - mcp__playwright__browser_type
  - mcp__playwright__browser_press_key
  - mcp__playwright__browser_select_option
  - mcp__playwright__browser_hover
  - mcp__playwright__browser_wait_for
  - mcp__playwright__browser_evaluate
  - mcp__playwright__browser_console_messages
  - mcp__playwright__browser_network_requests
  - mcp__playwright__browser_tabs
  - mcp__playwright__browser_navigate_back
  - mcp__playwright__browser_drag
  - mcp__playwright__browser_handle_dialog
  - mcp__playwright__browser_resize
---
<objective>
Autonomously verify built features using Playwright browser automation. Replaces manual UAT by navigating the application, performing test actions, taking screenshots, and recording pass/fail results without user interaction.

Same output format as /gsd:verify-work (UAT.md) so downstream commands (/gsd:plan-phase --gaps, diagnose-issues) work unchanged.

Output: {phase_num}-UAT.md with automated test results and screenshots.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/auto-verify.md
@$HOME/.claude/get-shit-done/templates/UAT.md
</execution_context>

<context>
Phase: $ARGUMENTS (required — phase number)
Optional flags:
  --base-url=URL (default: http://localhost:3000)

Context files are resolved inside the workflow (`init verify-work`) and delegated via `<files_to_read>` blocks.
</context>

<process>
Execute the auto-verify workflow from @$HOME/.claude/get-shit-done/workflows/auto-verify.md end-to-end.
Run all tests autonomously using Playwright. No user interaction during test execution.
</process>
