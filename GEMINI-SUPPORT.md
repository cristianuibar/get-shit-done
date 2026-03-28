# Adding Native Gemini CLI Support to GSD

> Technical specification for adding Google's Gemini CLI as a first-class GSD runtime alongside Claude Code and OpenCode.

---

## Overview

Gemini CLI is Google's open-source terminal-based AI coding agent ([github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)). It follows a similar architecture to Claude Code — system prompts, custom commands, agent delegation, tool access — but uses different file formats, directory conventions, and tool names.

This document specifies every change required to add Gemini CLI as a third supported runtime in the GSD installer and repo.

---

## Table of Contents

1. [Architecture Comparison](#1-architecture-comparison)
2. [Installer Changes (bin/install.js)](#2-installer-changes)
3. [Command Format Conversion (MD → TOML)](#3-command-format-conversion)
4. [Agent File Format](#4-agent-file-format)
5. [Tool Name Mapping](#5-tool-name-mapping)
6. [Model Profile Mapping](#6-model-profile-mapping)
7. [Path and Reference Conventions](#7-path-and-reference-conventions)
8. [Subagent Delegation](#8-subagent-delegation)
9. [Settings and Configuration](#9-settings-and-configuration)
10. [Hooks and Statusline](#10-hooks-and-statusline)
11. [Verification Checklist](#11-verification-checklist)
12. [Source References](#12-source-references)

---

## 1. Architecture Comparison

| Aspect | Claude Code | OpenCode | Gemini CLI |
|--------|-------------|----------|------------|
| **Config dir (global)** | `~/.claude/` | `~/.config/opencode/` | `~/.gemini/` |
| **Config dir (local)** | `.claude/` | `.opencode/` | `.gemini/` |
| **Config dir env var** | `CLAUDE_CONFIG_DIR` | `OPENCODE_CONFIG_DIR` | _(none known)_ |
| **Instructions file** | `CLAUDE.md` | _(via config)_ | `GEMINI.md` |
| **Command dir** | `commands/gsd/*.md` | `command/gsd-*.md` (flat) | `commands/gsd/*.toml` |
| **Command format** | Markdown (full file = prompt) | Markdown (flat, converted frontmatter) | TOML (`description` + `prompt` fields) |
| **Command namespace** | `/gsd:help` (colon from subdir) | `/gsd-help` (flat, dash) | `/gsd:help` (colon from subdir) |
| **Agent dir** | `agents/*.md` | `agents/*.md` | `agents/*.md` |
| **Agent format** | Markdown + YAML frontmatter | Markdown + converted frontmatter | Markdown + YAML frontmatter |
| **Agent frontmatter fields** | `name`, `description`, `allowed-tools`, `color` | `description`, `tools` (key:true map), `color` (hex) | `name`, `description`, `tools` (YAML array), `kind`, `model`, `max_turns`, `timeout_mins` — **no `color`** |
| **Argument placeholder** | `$ARGUMENTS` | `$ARGUMENTS` | `{{args}}` |
| **File include syntax** | `@path` | `@path` | `@path` |
| **Settings file** | `settings.json` | `opencode.json` | `settings.json` |
| **Subagent mechanism** | `Task()` tool (structured) | `Task()` tool (same as Claude) | `delegate_to_agent` / per-agent tool (model-driven) |
| **Shell in commands** | Not built-in | Not built-in | `!{shell command}` syntax |

---

## 2. Installer Changes

### 2.1 New Runtime Option

Add `'gemini'` as a third runtime in `bin/install.js`.

**CLI flags to add:**

```
--gemini              Install for Gemini CLI only
--all                 Install for all runtimes (Claude + OpenCode + Gemini)
```

Update `--both` to remain Claude + OpenCode for backward compatibility, or deprecate in favor of `--all`.

**Interactive prompt update:**

```
Which runtime(s) would you like to install for?

1) Claude Code    (~/.claude)
2) OpenCode       (~/.config/opencode) - open source, free models
3) Gemini CLI     (~/.gemini) - Google's AI agent
4) All
```

### 2.2 Directory Helper

```javascript
function getDirName(runtime) {
  if (runtime === 'opencode') return '.opencode';
  if (runtime === 'gemini') return '.gemini';
  return '.claude';
}

function getGlobalDir(runtime, explicitDir = null) {
  if (runtime === 'gemini') {
    if (explicitDir) return expandTilde(explicitDir);
    return path.join(os.homedir(), '.gemini');
  }
  // ... existing claude/opencode logic
}
```

### 2.3 Install Function Branching

The `install()` function needs a third branch for Gemini:

```javascript
if (isGemini) {
  // Gemini CLI: nested structure in commands/ directory, but TOML format
  const commandsDir = path.join(targetDir, 'commands');
  fs.mkdirSync(commandsDir, { recursive: true });
  const gsdSrc = path.join(src, 'commands', 'gsd');
  const gsdDest = path.join(commandsDir, 'gsd');
  copyAsTomlCommands(gsdSrc, gsdDest, pathPrefix, runtime);
} else if (isOpencode) {
  // existing opencode logic (flat structure)
} else {
  // existing claude logic (nested .md)
}
```

### 2.4 Banner Update

```javascript
const banner = `...
  A meta-prompting, context engineering and spec-driven
  development system for Claude Code, OpenCode, and Gemini CLI by TÂCHES.
`;
```

---

## 3. Command Format Conversion

### 3.1 Source Format (Claude Code Markdown)

```markdown
---
name: gsd:execute-phase
description: Execute all plans in a phase with wave-based parallelization
argument-hint: "<phase-number> [--gaps-only]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - TodoWrite
  - AskUserQuestion
---

<objective>
Execute all plans in a phase...
</objective>

<context>
Phase: $ARGUMENTS
@~/.claude/get-shit-done/references/ui-brand.md
</context>
```

### 3.2 Target Format (Gemini CLI TOML)

```toml
description = "Execute all plans in a phase with wave-based parallelization"

prompt = """
<objective>
Execute all plans in a phase...
</objective>

<context>
Phase: {{args}}
@~/.gemini/get-shit-done/references/ui-brand.md
</context>
"""
```

### 3.3 Conversion Rules

The installer must apply these transformations when writing command files for Gemini:

1. **Strip YAML frontmatter entirely** — Gemini TOML does not use it
2. **Extract `description`** from frontmatter → TOML `description` field
3. **Wrap body as `prompt`** → TOML multi-line string (`"""..."""`)
4. **Replace `$ARGUMENTS`** → `{{args}}`
5. **Replace path prefix** `~/.claude/` → `~/.gemini/` (or target path)
6. **Discard `allowed-tools`** — Gemini commands don't restrict tool access (agents do)
7. **Discard `name`** — Gemini derives name from file path
8. **Discard `argument-hint`** — No equivalent in Gemini TOML
9. **Change file extension** — `.md` → `.toml`
10. **Escape TOML special chars** — Backslashes in prompt content must be escaped for TOML triple-quoted strings (notably `\|` in grep patterns should become `\\|`)

### 3.4 Converter Function

```javascript
/**
 * Convert a Claude Code command (.md) to Gemini CLI format (.toml)
 * @param {string} content - Markdown file content with YAML frontmatter
 * @returns {string} - TOML file content
 */
function convertClaudeToGeminiToml(content) {
  let description = '';
  let body = content;

  // Extract frontmatter
  if (content.startsWith('---')) {
    const endIndex = content.indexOf('---', 3);
    if (endIndex !== -1) {
      const frontmatter = content.substring(3, endIndex).trim();

      // Extract description from frontmatter
      const descMatch = frontmatter.match(/^description:\s*(.+)$/m);
      if (descMatch) {
        description = descMatch[1].trim().replace(/^["']|["']$/g, '');
      }

      body = content.substring(endIndex + 3).trim();
    }
  }

  // Replace argument placeholder
  body = body.replace(/\$ARGUMENTS/g, '{{args}}');

  // Escape backslashes for TOML triple-quoted strings
  // In TOML """, backslash is an escape character
  // Literal backslashes in content (e.g., regex \|) must be doubled
  body = body.replace(/\\/g, '\\\\');

  // Build TOML
  const escapedDesc = description.replace(/"/g, '\\"');
  return `description = "${escapedDesc}"\n\nprompt = """\n${body}\n"""\n`;
}
```

### 3.5 Important: `Task()` References in Prompts

Claude commands contain `Task()` syntax examples like:

```
Task(prompt="...", subagent_type="gsd-executor", model="{executor_model}")
```

For Gemini, these should be converted to natural-language delegation instructions that tell the model to call the agent tool. The current approach of using `### SUBTASK: agent-name` headings works because Gemini's model sees the registered agent tools and delegates accordingly. See [Section 8](#8-subagent-delegation) for details.

Replace `Task()` call examples with the `### SUBTASK:` format:

```
### SUBTASK: gsd-executor

Execute plan at {plan_01_path}

Plan:
{plan_01_content}

Project state:
{state_content}
```

**Critical:** Do NOT let the conversion break text that mentions "Task()" in explanatory prose. The string `The @ syntax does not work across Task() boundaries` should become `The @ syntax does not work across agent boundaries` — not be split into a fake SUBTASK block.

---

## 4. Agent File Format

### 4.1 Good News: Same Format

Gemini CLI loads agents from `agents/*.md` using **Markdown with YAML frontmatter** — the same base format as Claude Code. The body of the file becomes the agent's `system_prompt`.

Source: [`packages/core/src/agents/agentLoader.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/agents/agentLoader.ts) (line 339-344):

```typescript
const files = dirEntries.filter(
  (entry) => entry.isFile() && !entry.name.startsWith('_') && entry.name.endsWith('.md'),
);
```

### 4.2 Frontmatter Field Differences

| Field | Claude Code | Gemini CLI | Notes |
|-------|-------------|------------|-------|
| `name` | Required (any format) | Required, must be slug (`[a-z0-9-_]+`) | Already compatible — GSD uses slugs |
| `description` | Required | Required | Same |
| `allowed-tools` | YAML array of Claude tool names | N/A — use `tools` instead | Must convert |
| `tools` | N/A | **YAML array** of Gemini tool names | See tool mapping |
| `color` | Color name (e.g., `cyan`) | **N/A — must be removed** | Causes validation error if present |
| `kind` | N/A | `'local'` (default) or `'remote'` | Not needed (defaults to local) |
| `model` | N/A | Model name string (optional, inherits if omitted) | Can set per-agent |
| `max_turns` | N/A | Positive integer (optional) | Limit agent turns |
| `timeout_mins` | N/A | Positive integer, default 5 (optional) | Set timeout |

> **IMPORTANT — Validated fix (Jan 2026):** The `tools` field MUST be a YAML array, NOT a comma-separated string. Gemini CLI's agent loader validates the frontmatter schema strictly — a string value for `tools` causes: `tools: Expected array, received string`. The `color` field is an unrecognized key that also causes validation failure. Both issues prevent the agent from loading entirely.

### 4.3 Conversion Function

```javascript
/**
 * Convert Claude Code agent frontmatter to Gemini CLI format
 * @param {string} content - Markdown agent file content
 * @returns {string} - Content with Gemini-compatible frontmatter
 */
function convertClaudeToGeminiAgent(content) {
  if (!content.startsWith('---')) return content;

  const endIndex = content.indexOf('---', 3);
  if (endIndex === -1) return content;

  const frontmatter = content.substring(3, endIndex).trim();
  const body = content.substring(endIndex + 3);

  const lines = frontmatter.split('\n');
  const newLines = [];
  let inAllowedTools = false;
  const tools = [];

  for (const line of lines) {
    const trimmed = line.trim();

    // Convert allowed-tools array to tools list
    if (trimmed.startsWith('allowed-tools:')) {
      inAllowedTools = true;
      continue;
    }
    if (inAllowedTools) {
      if (trimmed.startsWith('- ')) {
        tools.push(convertClaudeToGeminiToolName(trimmed.substring(2).trim()));
        continue;
      } else {
        inAllowedTools = false;
      }
    }

    // Strip color (not supported)
    if (trimmed.startsWith('color:')) continue;

    // Keep name, description
    newLines.push(line);
  }

  // Add tools as YAML array (Gemini requires array, NOT comma-separated string)
  if (tools.length > 0) {
    newLines.push('tools:');
    for (const tool of tools) {
      newLines.push(`  - ${tool}`);
    }
  }

  const newFrontmatter = newLines.join('\n').trim();

  // Also convert tool references in body
  let convertedBody = body;
  convertedBody = convertedBody.replace(/\$ARGUMENTS/g, '{{args}}');
  // Path replacement done separately by copyWithPathReplacement

  return `---\n${newFrontmatter}\n---${convertedBody}`;
}
```

---

## 5. Tool Name Mapping

Gemini CLI built-in tool names (from [`packages/core/src/tools/tool-names.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/tools/tool-names.ts)):

| Claude Code Tool | Gemini CLI Tool | Notes |
|-----------------|-----------------|-------|
| `Read` | `read_file` | |
| `Write` | `write_file` | |
| `Edit` | `replace` | |
| `Bash` | `run_shell_command` | |
| `Glob` | `glob` | |
| `Grep` | `search_file_content` | |
| `WebSearch` | `google_web_search` | |
| `WebFetch` | `web_fetch` | |
| `Task` | _(agent tools auto-registered)_ | See Section 8 |
| `TodoWrite` | `write_todos` | |
| `AskUserQuestion` | `ask_user` | |
| `NotebookEdit` | _(not available)_ | |
| `mcp__*` | _(do not list in agent tools)_ | MCP tools are auto-discovered at runtime — see note below |

Additional Gemini-only tools:
- `read_many_files` — read multiple files at once
- `list_directory` — list directory contents
- `save_memory` — save to persistent memory
- `activate_skill` — activate a skill
- `get_internal_docs` — access CLI internal documentation

> **IMPORTANT — Validated fix (Jan 2026):** MCP tool names like `mcp__context7__*` (wildcard patterns) are **invalid** in Gemini CLI agent `tools` arrays. MCP tools are auto-discovered from `mcpServers` in `settings.json` and made available to all agents at runtime — they do not need to be (and must not be) listed in agent frontmatter. For agents that should use MCP tools (e.g., Context7 for documentation lookups), add a usage hint in the agent's prompt body instead. Example:
>
> ```markdown
> **Research tools:** When researching libraries, frameworks, or APIs, use the Context7 MCP tools
> (`context7_resolve-library-id` then `context7_query-docs`) to fetch up-to-date documentation
> and code examples.
> ```

```javascript
const claudeToGeminiTools = {
  Read: 'read_file',
  Write: 'write_file',
  Edit: 'replace',
  Bash: 'run_shell_command',
  Glob: 'glob',
  Grep: 'search_file_content',
  WebSearch: 'google_web_search',
  WebFetch: 'web_fetch',
  TodoWrite: 'write_todos',
  AskUserQuestion: 'ask_user',
  // Task is not mapped — agents are auto-registered as tools
  // mcp__* tools are NOT mapped — they are auto-discovered from mcpServers config
};

function convertClaudeToGeminiToolName(claudeTool) {
  if (claudeToGeminiTools[claudeTool]) {
    return claudeToGeminiTools[claudeTool];
  }
  // MCP tools: skip entirely — they are auto-discovered at runtime
  // Do NOT include mcp__* tools in agent frontmatter
  if (claudeTool.startsWith('mcp__')) {
    return null; // caller should filter out nulls
  }
  // Fallback: lowercase
  return claudeTool.toLowerCase();
}
```

---

## 6. Model Profile Mapping

### 6.1 Model Equivalents

| Claude Model | Gemini Equivalent | Role |
|-------------|-------------------|------|
| `opus` | `gemini-2.5-pro` | High-reasoning, planning, architecture |
| `sonnet` | `gemini-2.5-flash` | Execution, research, verification |
| `haiku` | `gemini-2.5-flash` | Budget tasks, read-only exploration |

> **Note:** As of Jan 2026, Gemini has a two-tier model hierarchy (Pro/Flash) vs Claude's three-tier (Opus/Sonnet/Haiku). Both Sonnet and Haiku map to Flash. When Gemini releases a third tier, revisit this mapping.

### 6.2 Profile Table (Gemini)

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| gsd-planner | gemini-2.5-pro | gemini-2.5-pro | gemini-2.5-flash |
| gsd-roadmapper | gemini-2.5-pro | gemini-2.5-flash | gemini-2.5-flash |
| gsd-executor | gemini-2.5-pro | gemini-2.5-flash | gemini-2.5-flash |
| gsd-phase-researcher | gemini-2.5-pro | gemini-2.5-flash | gemini-2.5-flash |
| gsd-project-researcher | gemini-2.5-pro | gemini-2.5-flash | gemini-2.5-flash |
| gsd-research-synthesizer | gemini-2.5-flash | gemini-2.5-flash | gemini-2.5-flash |
| gsd-debugger | gemini-2.5-pro | gemini-2.5-flash | gemini-2.5-flash |
| gsd-codebase-mapper | gemini-2.5-flash | gemini-2.5-flash | gemini-2.5-flash |
| gsd-verifier | gemini-2.5-flash | gemini-2.5-flash | gemini-2.5-flash |
| gsd-plan-checker | gemini-2.5-flash | gemini-2.5-flash | gemini-2.5-flash |
| gsd-integration-checker | gemini-2.5-flash | gemini-2.5-flash | gemini-2.5-flash |

### 6.3 Runtime Model References

In commands and workflows, model names appear in lookup tables and `Task()` calls. The installer must replace:

- `opus` → `gemini-2.5-pro`
- `sonnet` → `gemini-2.5-flash`
- `haiku` → `gemini-2.5-flash`

**Be careful with sed replacements**: these are short words that could appear in other contexts. Use word-boundary-aware matching or only replace within known patterns (model lookup tables, Task calls with `model=` parameter).

### 6.4 model-profiles.md

The `get-shit-done/references/model-profiles.md` file contains the full profile table. Install a Gemini-specific version with the mapped model names.

---

## 7. Path and Reference Conventions

### 7.1 Path Replacement

All source files use `~/.claude/` as the path prefix. For Gemini:

| Pattern | Replacement |
|---------|-------------|
| `~/.claude/` | `~/.gemini/` (global) or `./.gemini/` (local) |
| `~/.opencode/` | (ignore for gemini target) |

The existing `copyWithPathReplacement()` function handles this. Extend the regex:

```javascript
function copyWithPathReplacement(srcDir, destDir, pathPrefix, runtime) {
  const isGemini = runtime === 'gemini';
  // ...
  // For Gemini: replace both ~/.claude/ patterns and any stale ~/.opencode/ patterns
  const claudeDirRegex = /~\/\.claude\//g;
  content = content.replace(claudeDirRegex, pathPrefix);
}
```

### 7.2 `@` File References

Both Claude Code and Gemini CLI use `@path` syntax to include file contents in prompts. This works the same way — no conversion needed beyond path prefix replacement.

### 7.3 Command Namespace

Gemini CLI derives command names from the file path the same way as Claude Code:

- `commands/gsd/help.toml` → `/gsd:help`
- Subdirectories create colon-separated namespaces

No structural changes needed (unlike OpenCode which uses flat structure).

---

## 8. Subagent Delegation

### 8.1 How It Works in Each Runtime

**Claude Code:** Uses the `Task()` tool — a structured API call:
```
Task(prompt="...", subagent_type="gsd-executor", model="sonnet")
```
The `Task()` tool spawns a fresh-context agent with the specified type and model.

**Gemini CLI:** Each registered agent becomes a callable tool via `SubagentTool`. The model autonomously decides to delegate based on agent descriptions. When delegation happens, the UI shows:
```
Delegating to agent 'gsd-executor'
```

Source: [`packages/core/src/agents/subagent-tool.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/agents/subagent-tool.ts) — each agent is registered as an individual tool that the model can call.

### 8.2 Conversion Approach

Replace `Task()` call examples in commands with instructional `### SUBTASK:` headings that tell the model what to delegate:

**Claude Code format:**
```
Task(prompt="Execute plan at {path}\n\nPlan:\n{content}", subagent_type="gsd-executor", model="{executor_model}")
```

**Gemini CLI format:**
```
### SUBTASK: gsd-executor

Execute plan at {path}

Plan:
{content}

Project state:
{state_content}
```

This works because:
1. The model sees `### SUBTASK: gsd-executor` as an instruction
2. `gsd-executor` is registered as a tool (from `agents/gsd-executor.md`)
3. The model calls the `gsd-executor` tool, passing the content as the `query` parameter
4. The agent runs with its own system prompt (the body of `gsd-executor.md`) and fresh context

### 8.3 Parallel Execution

Claude Code spawns multiple `Task()` calls in a single message for parallel execution. In Gemini CLI, the model can also call multiple agent tools in a single turn. The `### SUBTASK:` blocks should be grouped together to encourage parallel invocation, following the same wave-based pattern.

### 8.4 Important Caveat

The `### SUBTASK:` pattern is prompt-engineering, not structured API. It relies on the model correctly interpreting the instruction and calling the agent tool. This is inherently less deterministic than Claude Code's `Task()` tool. Testing is needed to verify reliability across different Gemini model versions.

### 8.5 Text Mentioning Task() in Prose

Some commands contain explanatory text like:
```
The `@` syntax does not work across Task() boundaries.
```

For Gemini, this should become:
```
The `@` syntax does not work across agent boundaries.
```

The converter must handle this carefully — do NOT treat prose mentions of `Task()` as SUBTASK blocks.

---

## 9. Settings and Configuration

### 9.1 Gemini CLI `settings.json`

Gemini CLI uses `~/.gemini/settings.json` for configuration. The installer should not heavily modify this file (unlike Claude Code where hooks are registered). The following settings are **required** for GSD agents to function:

```json
{
  "general": {
    "previewFeatures": true
  },
  "experimental": {
    "enableAgents": true
  },
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

> **IMPORTANT — Validated fix (Jan 2026):** The `experimental.enableAgents` setting is **required**. Without it, Gemini CLI only loads its two built-in agents (`codebase_investigator` and `cli_help`) and ignores all custom agent files in `~/.gemini/agents/`. This causes the error: `Agent 'gsd-executor' not found. Available agents are: 'codebase_investigator', 'cli_help'`.

**Required settings explained:**
- `previewFeatures: true` — may be needed for agent support depending on the Gemini CLI version
- `experimental.enableAgents: true` — **required** to load custom agents from `agents/*.md`
- `mcpServers.context7` — provides up-to-date library documentation to research agents (phase-researcher, project-researcher, planner) via Context7 MCP. Without this, the Context7 hints in agent prompts have no effect.

The installer should **merge** these settings into the existing `settings.json` (not overwrite), preserving any user configuration already present.

### 9.2 No Hook Registration

Gemini CLI does not support the same hook system as Claude Code. The `SessionStart` hooks (update check, statusline) should be **skipped** for Gemini installs:

```javascript
// Configure hooks (skip for gemini and opencode - different hook systems)
if (!isOpencode && !isGemini) {
  // ... existing Claude hook registration
}
```

### 9.3 No Statusline

Gemini CLI does not support Claude Code's `statusLine` setting. Skip statusline configuration for Gemini installs.

---

## 10. Hooks and Statusline

### 10.1 Update Check

The `gsd-check-update.js` hook runs on Claude Code's `SessionStart` event. Gemini CLI does not have an equivalent hook system. Options:

1. **Skip entirely** for Gemini installs (simplest)
2. Embed a version check in the `/gsd:progress` or `/gsd:help` command prompts using `!{node ~/.gemini/hooks/gsd-check-update.js}` shell syntax
3. Rely on users running `/gsd:update` manually

Recommended: Option 1 (skip). Add a note in `/gsd:help` output about checking for updates.

### 10.2 Statusline

Not applicable — Gemini CLI does not support custom statuslines.

---

## 11. Verification Checklist

After implementation, verify:

### Installer
- [ ] `npx get-shit-done-cc --gemini --global` installs to `~/.gemini/`
- [ ] `npx get-shit-done-cc --gemini --local` installs to `./.gemini/`
- [ ] Interactive prompt shows Gemini as option 3
- [ ] `--all` installs to all three runtimes
- [ ] `--gemini --uninstall` cleanly removes GSD files
- [ ] Existing Claude/OpenCode installs unaffected

### Commands
- [ ] All 27 commands converted to valid `.toml` files
- [ ] `python3 -c "import tomllib; ..."` validates all TOML files parse
- [ ] `$ARGUMENTS` replaced with `{{args}}` in all prompts
- [ ] Paths updated to `~/.gemini/` (global) or `./.gemini/` (local)
- [ ] No leftover frontmatter (no `---` blocks in TOML prompt content)
- [ ] `Task()` call examples converted to `### SUBTASK:` format
- [ ] No corrupted text from Task() prose mentions
- [ ] Backslashes properly escaped for TOML

### Agents
- [ ] All 11 agent `.md` files installed to `agents/`
- [ ] Frontmatter uses `tools:` as a **YAML array** (not comma-separated string)
- [ ] `color:` field stripped (causes validation error)
- [ ] No `mcp__*` wildcard tool names in `tools` array (causes "Invalid tool name" error)
- [ ] `name:` uses slug format (`[a-z0-9-_]+`)
- [ ] Body (system_prompt) has paths updated
- [ ] No `$ARGUMENTS` references remain
- [ ] Research agents (phase-researcher, project-researcher, planner) have Context7 MCP usage hints in prompt body

### Model Profiles
- [ ] `model-profiles.md` uses Gemini model names
- [ ] All model lookup tables in commands use `gemini-2.5-pro`/`gemini-2.5-flash`
- [ ] No `opus`/`sonnet`/`haiku` references outside of comparison/documentation context

### Workflows, Templates, References
- [ ] All `.md` files have paths updated
- [ ] Model names updated in workflow files
- [ ] `Task()` references converted in workflow files
- [ ] `config.json` template unchanged (runtime-agnostic)

### Settings
- [ ] `settings.json` has `experimental.enableAgents: true` (required for custom agents)
- [ ] `settings.json` has `mcpServers.context7` configured (required for research agents)
- [ ] Settings are merged into existing file, not overwritten

### Integration
- [ ] `/gsd:help` runs and displays command list
- [ ] `/gsd:progress` runs and shows project state
- [ ] Agent delegation works (test with `/gsd:new-project`)
- [ ] `@` file references resolve correctly in prompts
- [ ] Model profile resolution works in commands

### Sweep
- [ ] `grep -r "\.claude" ~/.gemini/` returns only PORTING documentation
- [ ] `grep -r "\$ARGUMENTS" ~/.gemini/commands/` returns nothing
- [ ] `grep -r "opus\|sonnet\|haiku" ~/.gemini/` returns nothing outside docs
- [ ] `grep -r "Anthropic\|anthropic" ~/.gemini/` returns nothing outside docs
- [ ] `grep -r "mcp__" ~/.gemini/agents/` returns nothing (MCP tools must not be in agent frontmatter)
- [ ] `grep -r "^color:" ~/.gemini/agents/` returns nothing (unsupported field)

---

## 12. Source References

### Gemini CLI Source Code (verified against repo)

- **Agent loader**: [`packages/core/src/agents/agentLoader.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/agents/agentLoader.ts) — loads `.md` files from `agents/` directory, parses YAML frontmatter, body becomes `system_prompt`
- **Agent types**: [`packages/core/src/agents/types.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/agents/types.ts) — `AgentDefinition` interface with `name`, `description`, `kind`, `promptConfig`, `modelConfig`, `runConfig`, `toolConfig`
- **Agent registry**: [`packages/core/src/agents/registry.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/agents/registry.ts) — loads from user-level (`~/.gemini/agents/`) and project-level (`.gemini/agents/`)
- **Subagent tool**: [`packages/core/src/agents/subagent-tool.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/agents/subagent-tool.ts) — each agent registered as individual callable tool, UI shows "Delegating to agent 'name'"
- **Tool names**: [`packages/core/src/tools/tool-names.ts`](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/tools/tool-names.ts) — canonical list of all built-in tool names

### Gemini CLI Documentation

- [Custom Commands](https://geminicli.com/docs/cli/custom-commands/) — TOML format, `description` + `prompt` fields, `{{args}}` placeholder, `!{shell}` syntax
- [Configuration](https://geminicli.com/docs/get-started/configuration/) — `settings.json` structure, hierarchy
- [GEMINI.md](https://geminicli.com/docs/cli/gemini-md/) — `@file` imports, hierarchical loading
- [Policy Engine](https://geminicli.com/docs/core/policy-engine/) — `delegate_to_agent` default policy

### GitHub Issues and PRs

- [#14316 — Create delegate_to_agent tool](https://github.com/google-gemini/gemini-cli/issues/14316) (implemented, closed)
- [#14769 — DelegateToAgentTool with discriminated union](https://github.com/google-gemini/gemini-cli/pull/14769) (merged)
- [#17346 — Refactor to one tool per agent](https://github.com/google-gemini/gemini-cli/pull/17346) (merged, current architecture)
- [#3132 — Post V1.0 Agent Work](https://github.com/google-gemini/gemini-cli/issues/3132) (tracking epic)
- [#15535 — Support Markdown-based commands](https://github.com/google-gemini/gemini-cli/issues/15535) (open feature request — if implemented, would eliminate TOML conversion)
