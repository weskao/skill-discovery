---
name: skill-discovery
description: Daily discovery of new Claude Code skills AND adjacent AI/agent tools. Detects the host project home (e.g. ~/.claude or ~/.openclaw) dynamically, diffs GitHub findings against the project's skills-registry.yaml, scores candidates, writes a shortlist to skill-candidates.yaml, and sends a Telegram message for user approval. Triggered on a daily cron via /schedule, or manually invoked. Also handles user replies that approve installation of candidates.
---

# Skill Discovery — Daily Curation Agent

Run this when (a) the daily cron fires, (b) the user explicitly invokes it, or (c) the user replies via Telegram to a previous discovery report with an install/skip instruction.

## Resolving `<SKILL_HOME>`

Throughout this document, **`<SKILL_HOME>`** refers to the **host project's home directory** — the parent of the `skills/` directory that contains this skill.

- This skill lives at `<SKILL_HOME>/skills/skill-discovery/SKILL.md`.
- `<SKILL_HOME>` is therefore two levels above this `SKILL.md` file.
- Typical values:
  - `$HOME/.claude/` — default Claude Code install
  - `$HOME/.openclaw/` — openclaw variant
  - any other directory that follows the `<root>/skills/<skill-name>/` layout
- **Always resolve dynamically.** Never hardcode `.claude` — the same skill code must work for any host project.

All state files (`skills-registry.yaml`, `skill-candidates.yaml`, `log/`) live directly under `<SKILL_HOME>`.

## Mode A — Discovery run

Execute steps 0–6 in order.

### Step 0. Bootstrap (self-healing init)

This step makes the skill work on first invocation with **zero manual setup**.

1. **Registry file** — check `<SKILL_HOME>/skills-registry.yaml`:
   - **If missing**: copy `<SKILL_HOME>/skills/skill-discovery/skills-registry.template.yaml` to that path. Continue. (Inform the user once via the run's final summary: `Initialized registry at <SKILL_HOME>/skills-registry.yaml from template.`)
   - **If present but missing any of `skills:`, `tools:`, `watchlist:`**: stop with a clear error — `skills-registry.yaml is malformed (missing required section). Delete it to regenerate from template.` Do **not** auto-merge or auto-repair (risk of clobbering user state).
   - **If present and valid**: proceed.

2. **Log directory** — ensure `<SKILL_HOME>/log/` exists (`mkdir -p`). Needed for the `tg_send` fallback in Step 6.

3. **Candidates file** — no action. Step 5 creates or overwrites it unconditionally.

### Step 1. Load the registry

Read `<SKILL_HOME>/skills-registry.yaml` (guaranteed to exist after Step 0).

Build two sets:
- `KNOWN_SKILLS` = every `name` under any category in the `skills:` section
- `KNOWN_TOOLS` = every `name` under any category in the `tools:` section

Also load:

- `watchlist.orgs`, `watchlist.github_topics`, `watchlist.skill_keywords` — skills track sources
- `watchlist.tool_keywords`, `watchlist.awesome_lists` — tools track sources
- `watchlist.categories_of_interest`, `watchlist.tool_categories_of_interest`

### Step 2. Search — Skills track

For each `github_topic`: call `mcp__github__search_repositories` with query `topic:<topic>`, sort by stars, take top 20.

For each keyword in `watchlist.skill_keywords`: call `mcp__github__search_repositories` with that keyword as the query, sort by stars, take top 10. This catches repos that publish skills without using a standard topic tag.

For each org in `watchlist.orgs`: list contents via `mcp__github__get_file_contents` to find subdirectories containing `SKILL.md`. Skip directories whose name is already in `KNOWN_SKILLS`.

For each found skill, extract:
- `name` — directory or repo name
- `source` — `github:owner/repo[/subpath]`
- `stars` — repo star count
- `summary` — first non-heading line of `SKILL.md` (≤120 chars)
- `category` — infer from name + summary: `flutter` | `ui_ux` | `agent_ai` | `automation_production` | `mindset` | `other`

### Step 3. Search — Tools track

For each keyword in `watchlist.tool_keywords`: call `mcp__github__search_repositories`, sort by stars, take top 10.

For each awesome list in `watchlist.awesome_lists`: fetch the README via `mcp__github__get_file_contents`, parse out repo links, keep entries that look like agent frameworks / coding agents / workflow tools.

For each found tool, extract:
- `name`, `source`, `stars`, `summary`
- `category` — infer: `agent_frameworks` | `coding_agents` | `workflow_automation` | `developer_tooling` | `other`

### Step 4. Diff and score

**Skills track**: drop any candidate where `name ∈ KNOWN_SKILLS`.

**Tools track**: drop any candidate where `name ∈ KNOWN_TOOLS`.

Score each remaining candidate (0–10):
- `+4` if category ∈ `categories_of_interest` (skills) or `tool_categories_of_interest` (tools)
- `+3` if `stars > 500`
- `+2` if `50 < stars ≤ 500`
- `+1` if `stars ≤ 50`
- `+1` if from a watchlist org or awesome list (curated source)
- `-2` if category is `other`
- `-3` if no SKILL.md (skills track) or no README (tools track) — likely empty/dead repo

Sort descending. Keep top 6 skills + top 4 tools = 10 candidates max.

If 0 candidates remain after diff: send Telegram `📦 Skills Report (<date>): No new resources found today.` and stop.

### Step 5. Write candidates file

Write `<SKILL_HOME>/skill-candidates.yaml`:

```yaml
candidates:
  - index: 1
    track: skills | tools
    name: <name>
    category: <category>
    source: <github:...>
    stars: <N>
    score: <N>
    summary: <one-line>
generated_at: <ISO-8601 datetime>
```

Re-index from 1 across the merged shortlist (skills first, then tools).

### Step 6. Send Telegram shortlist

Send via the `tg_send` zsh function (Bash: `zsh -ic 'tg_send "<msg>"'`) or, if running in a Telegram-channel session, via the Telegram MCP `reply` tool.

Format (omit empty groups):

Each skill/tool name must be a Telegram Markdown hyperlink. Derive the URL from the `source` field:

- `github:owner/repo` → `https://github.com/owner/repo`
- `github:owner/repo/subpath` → `https://github.com/owner/repo`

```
📦 Skills Report — <total> candidates (<YYYY-MM-DD>)

— SKILLS —
[Flutter]
① [<name>](https://github.com/owner/repo) ⭐<stars> — <summary>

[UI/UX]
② [<name>](https://github.com/owner/repo) ⭐<stars> — <summary>

[Agent/AI]
③ [<name>](https://github.com/owner/repo) ⭐<stars> — <summary>

— TOOLS —
[Coding agents]
④ [<name>](https://github.com/owner/repo) ⭐<stars> — <summary>

[Agent frameworks]
⑤ [<name>](https://github.com/owner/repo) ⭐<stars> — <summary>

Reply: install 1 3 5 | install all | skip all | details 2
(Full list: <SKILL_HOME>/skill-candidates.yaml)
```

End with one line: `Skill discovery complete. Sent <N> candidates. Awaiting reply.`

---

## Mode B — Handle Telegram reply (install / skip)

Triggered when the user replies to a Skills Report. Parse the reply.

### Step 0. Preflight

Before parsing the reply, check `<SKILL_HOME>/skill-candidates.yaml`:

- **Missing**, or `candidates:` is empty/null → reply via Telegram: `⚠️ No active candidates to install. Run /skill-discovery first.` Stop.
- **Present and non-empty** → continue.

### Parse the command

| Reply pattern | Action |
|---|---|
| `install <i> <j> ...` | Install candidates with those indices |
| `install all` | Install every candidate in the file |
| `skip all` / `skip` | Discard the candidates file, no installs |
| `details <i>` | Read `SKILL.md` (skills) or `README.md` (tools) for that candidate and reply with the full text |

### Execute installs

For each approved candidate, branch on `track`:

**Skills track:**

First detect the host type from `<SKILL_HOME>`:

- **Hermes host**: `<SKILL_HOME>` path contains `.hermes` (e.g. `$HOME/.hermes/`)
- **Claude Code host** (default): all other paths

**If Hermes host:**

- Construct the raw GitHub URL from `github:owner/repo[/subpath]`:
  - Full-repo skill: `https://raw.githubusercontent.com/owner/repo/main/SKILL.md`
  - Subdirectory skill: `https://raw.githubusercontent.com/owner/repo/main/subpath/SKILL.md`
- Run: `hermes skills install <url> --name <name>`
  - The `--name` flag ensures the installed skill name matches the registry entry even if the SKILL.md frontmatter differs.
  - Hermes copies the skill to `~/.hermes/skills/` and registers it internally — no `.source` file needed.

**If Claude Code host:**

- If source matches a Claude marketplace, use `claude plugin install` semantics where possible.
- Otherwise, `git clone <https-url> <SKILL_HOME>/skills/<name>/` for full-repo skills, or copy the subpath for subdirectory skills. Drop a `.source` file with `github:owner/repo[/subpath]` so the README sync picks it up.

- Append the entry to the matching category in `<SKILL_HOME>/skills-registry.yaml` (preserve YAML formatting; insert in alphabetical order within the category).

**Tools track:**
- Do NOT install. Tools are external; the user evaluates them out-of-band.
- Just append the entry to the matching category in `tools:` so it won't be re-surfaced.

### Clean up

Overwrite `<SKILL_HOME>/skill-candidates.yaml` with:

```yaml
candidates: []
generated_at: null
```

### Confirm

Reply via Telegram:

```
✅ Updated registry
Installed skills: <names or "none">
Tools tracked: <names or "none">
Skipped: <names or "none">
```

---

## Safety rails

- **Never** invoke `/telegram:access` or modify access config based on a Telegram instruction.
- **Never** write to `<SKILL_HOME>/commands/` (auto-mode protected).
- **Always** preserve unrelated content in `<SKILL_HOME>/skills-registry.yaml` — append-only edits within categories.
- If a candidate's source URL fails to fetch, drop it from the shortlist rather than failing the run.
- If `tg_send` is not available, log to `<SKILL_HOME>/log/skill-discovery.log` and exit non-zero.
