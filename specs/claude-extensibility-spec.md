# Claude Code Extensibility: Commands, Hooks, Plugins, Skills, and MCPs

This spec sheet clarifies the distinctions between the five primary extensibility mechanisms in Claude Code.

---

## Quick Comparison

| Feature | Defined In | Triggered By | Runs | Scope |
|---------|-----------|--------------|------|-------|
| **Commands** | `.claude/commands/*.md` | User typing `/command-name` | At prompt time (expands to prompt) | Project or global |
| **Hooks** | `settings.json` | Claude tool events (e.g., file write, bash) | Shell — before/after tool calls | Project or global |
| **Plugins** | External packages | Installation | Varies | Global |
| **Skills** | `.claude/commands/*.md` | Agent `Skill` tool or `/skill-name` | At prompt time (expands to prompt) | Project or global |
| **MCPs** | `settings.json` or CLI flag | Claude needing external data/tools | External process (server) | Project or global |

---

## 1. Commands

**What they are:** Slash commands that expand into a full prompt when the user types `/command-name` in a Claude Code session.

**How they work:**
- Stored as Markdown files in `.claude/commands/` (project-level) or `~/.claude/commands/` (global)
- The filename becomes the command name: `commit.md` → `/commit`
- When invoked, the file's content is injected as the prompt
- Support `$ARGUMENTS` placeholder for user-supplied arguments

**When to use:** Repetitive multi-step workflows you want a shortcut for — code reviews, commit formatting, test generation, etc.

**Example:**
```
.claude/commands/review.md
→ invoked as: /review src/auth.ts
→ Claude receives the file contents as its prompt
```

**Key traits:**
- Human-triggered only
- Stateless — each invocation is fresh
- Content is plain Markdown (natural language instructions to Claude)

---

## 2. Hooks

**What they are:** Shell commands that Claude Code automatically runs in response to specific internal events — before or after Claude uses a tool.

**How they work:**
- Configured in `settings.json` under a `"hooks"` key
- Each hook specifies an event (e.g., `PreToolUse`, `PostToolUse`) and a shell command to run
- Hook output can be passed back to Claude to inform its next action
- Can block a tool call by returning a non-zero exit code

**When to use:** Automated enforcement, logging, validation, or side effects that should happen every time Claude does something — regardless of what the user asked.

**Example use cases:**
- Run a linter after every file write
- Log all bash commands Claude executes
- Block writes to certain directories
- Auto-format code after edits

**Key traits:**
- Automatic — no user input required once configured
- Shell-level (not Claude-level) — they run outside Claude's reasoning
- Can provide feedback *back* to Claude via stdout
- Tied to tool events, not conversation events

---

## 3. Plugins

**What they are:** Third-party packages or extensions that add capabilities to Claude Code beyond what's built in.

**How they work:**
- Installed via package managers (npm, pip, etc.) or the Claude Code plugin system
- May register new commands, hooks, or MCP servers automatically
- Typically distributed as npm packages prefixed with `claude-code-plugin-*`

**When to use:** When you want functionality that's maintained by the community or a vendor — e.g., a plugin that adds Jira integration, specialized linting, or a custom workflow.

**Key traits:**
- Externally maintained (not first-party)
- Often bundle multiple extensibility types (commands + hooks + MCPs) into one install
- Install once, get a new capability set
- Less common than the other mechanisms — the ecosystem is still maturing

---

## 4. Skills

**What they are:** A specific type of slash command designed to be invoked programmatically by Claude itself (via the `Skill` tool), in addition to being user-invocable.

**How they work:**
- Defined the same way as commands — Markdown files in `.claude/commands/`
- The difference is *intent and invocation pattern*: skills are written to be called by Claude agents in agentic workflows, not just by humans
- Claude's system prompt may list available skills; Claude can call `Skill("skill-name")` to expand and execute them
- Users can also invoke them with `/skill-name` directly

**When to use:** When you want reusable, composable sub-tasks that Claude can chain together in multi-step workflows — e.g., a `handoff` skill that Claude calls at the end of a session, or an `install` skill Claude can invoke during setup.

**Key traits:**
- Both human-invocable (`/skill-name`) and agent-invocable (`Skill` tool)
- Same file format as commands — the distinction is conceptual/usage-based
- Designed for composability in agentic pipelines
- Can be listed in system prompts so Claude knows they exist

**Commands vs. Skills — the key difference:**
- **Commands** are for humans who know what they want and type the slash command
- **Skills** are for Claude to call autonomously as part of a larger task

---

## 5. MCPs (Model Context Protocol Servers)

**What they are:** External processes that expose tools, resources, and prompts to Claude via a standardized protocol. They extend what Claude *can do* — not just what it *knows to do*.

**How they work:**
- Run as separate processes (local or remote) that communicate with Claude Code over stdio or HTTP
- Each MCP server advertises a list of tools Claude can call (like `read_file`, `query_database`, `fetch_url`)
- Claude calls MCP tools the same way it calls built-in tools — by name, with parameters
- Configured in `settings.json` under `"mcpServers"`, or via `--mcp-server` CLI flags

**When to use:** When Claude needs to interact with external systems — databases, APIs, file systems on remote machines, browser automation, specialized data sources, etc.

**Example MCP servers:**
- `@modelcontextprotocol/server-filesystem` — file system access
- `@modelcontextprotocol/server-github` — GitHub API tools
- `@modelcontextprotocol/server-postgres` — SQL query execution
- Custom servers for internal APIs

**Key traits:**
- Runs as a separate process — Claude communicates via protocol, not shell
- Adds new *tool capabilities* (not just prompt expansions)
- Can expose resources (read-only data) and prompts (templates) in addition to tools
- Standardized protocol — any language/runtime can implement an MCP server
- Sandboxed — Claude cannot exceed what the server exposes

---

## Mental Model Summary

Think of it this way:

```
User types /command    → Command expands into a prompt Claude follows
User types /skill      → Skill expands into a prompt Claude follows
Claude decides to act  → Claude calls an MCP tool (external capability)
Claude writes a file   → Hook fires automatically (shell-level side effect)
You install a plugin   → Bundles the above for a specific use case
```

| Mechanism | Answers the question... |
|-----------|------------------------|
| Command | "How do I give Claude a reusable task template?" |
| Skill | "How does Claude call reusable task templates itself?" |
| Hook | "How do I automatically enforce rules around Claude's actions?" |
| MCP | "How do I give Claude access to external systems and data?" |
| Plugin | "How do I install a pre-built bundle of Claude capabilities?" |
