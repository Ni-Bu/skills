# skills

A small collection of [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skills I use day-to-day.

## Skills

| Skill | What it does |
|---|---|
| [`review`](./review) | Senior-engineer-level review of the current branch against its PR base. Goes beyond lint — flags design mistakes, inconsistencies, missing error handling, convention violations, and logic gaps. |
| [`tdd`](./tdd) | Test-driven development loop. Writes behavior-first tests and runs the red/green/refactor cycle, with supporting notes on deep modules, interface design, mocking, and refactoring. |
| [`grill-me`](./grill-me) | Stress-tests a plan by interviewing you relentlessly, walking each branch of the decision tree and offering a recommended answer per question. |
| [`write-a-prd`](./write-a-prd) | Builds a PRD through interview + codebase exploration, then writes it to an Obsidian vault using a structured template. |
| [`prd-to-slices`](./prd-to-slices) | Breaks a PRD into independently-grabbable vertical slices (tracer bullets), saved as linked Obsidian notes with a Mermaid dependency graph. |
| [`improve-codebase-architecture`](./improve-codebase-architecture) | Explores a codebase for architectural friction and writes an Obsidian RFC proposing module-deepening refactors. |
| [`recap`](./recap) | End-of-day recap. Reads Claude Code session transcripts for the day, cross-references git + GitHub PR activity (+ Monday, optionally), and writes an Obsidian daily note. |
| [`brag`](./brag) | Maintains a yearly "brag document" in Obsidian. Runs automatically off the daily recap, with standalone and backfill modes. |

## Install

Pick one.

**Symlink (recommended — stay in sync with `git pull`):**

```bash
git clone https://github.com/Ni-Bu/skills.git ~/SideProjects/skills
mkdir -p ~/.claude/skills
for s in review tdd grill-me write-a-prd prd-to-slices improve-codebase-architecture recap brag; do
  ln -sfn ~/SideProjects/skills/"$s" ~/.claude/skills/"$s"
done
```

**Snapshot copy (fork-and-forget — edit freely, pull your own updates manually):**

```bash
git clone https://github.com/Ni-Bu/skills.git /tmp/skills
mkdir -p ~/.claude/skills
for s in review tdd grill-me write-a-prd prd-to-slices improve-codebase-architecture recap brag; do
  cp -R /tmp/skills/"$s" ~/.claude/skills/
done
```

After either, restart Claude Code and the skills will show up in `/skills`.

## Obsidian-backed skills

Five skills write their output into an [Obsidian](https://obsidian.md) vault:

- `write-a-prd` → `$OBSIDIAN_VAULT/PRDs/<feature>/PRD - <feature>.md`
- `prd-to-slices` → `$OBSIDIAN_VAULT/PRDs/<feature>/<slice>.md` (+ `Dependency Graph.md`)
- `improve-codebase-architecture` → `$OBSIDIAN_VAULT/RFC - <module>.md`
- `recap` → `$OBSIDIAN_VAULT/daily-recaps/YYYY-MM-DD.md`
- `brag` → `$OBSIDIAN_VAULT/brag/<year>.md`

### Setting up Obsidian with Claude Code

1. Install Obsidian and create a vault (or open an existing one).
2. Export the vault path as `OBSIDIAN_VAULT` in your shell rc — Claude Code's subprocess inherits it from there:

   ```bash
   # ~/.zshrc or ~/.bashrc
   export OBSIDIAN_VAULT="$HOME/ObsidianVault"
   ```

3. Restart your terminal (or `source ~/.zshrc`).
4. Verify inside a Claude Code session: ask Claude to run `echo "$OBSIDIAN_VAULT"`. It should print your vault path.

If `$OBSIDIAN_VAULT` is unset when you invoke one of the Obsidian skills, Claude will ask you for a path at first use.

### Obsidian features the templates use

The templates assume you're reading the output in Obsidian, not plain Markdown:

- **YAML frontmatter** — tags, dates, custom fields for search and Dataview queries
- **Callouts** — `> [!tip]`, `> [!warning]`, `> [!success]`, `> [!example]-` (collapsed by default), etc.
- **Wiki-links** — `[[Note name]]` for cross-linking PRDs to slices to recaps
- **Mermaid diagrams** — for the PRD dependency graph and the `brag` yearly timeline

All of these render natively in Obsidian with no plugins. They'll still be readable in VS Code or on GitHub, but the callouts and wiki-links won't look as nice.

## Optional: Monday.com integration

`recap` and `brag` can pull in Monday activity (updates on your items, cases resolved, etc.) if a Monday MCP is configured. **This is optional** — both skills fall back gracefully if Monday isn't connected.

There are two ways to wire it up, in order of ease:

### Option 1 — Claude's built-in Monday connector (recommended, if available)

1. In Claude Code, run `/mcp` and pick **Monday** from the connector list.
2. Authenticate via OAuth in the browser window that opens.
3. Done — tools like `mcp__claude_ai_monday_com__*` will appear in your session.

The connector is gated to certain Claude plans (Max / Team / Enterprise at the time of writing — check your plan in `/status`). If it's available to you, this is by far the easiest path.

### Option 2 — Self-hosted `monday-api-mcp` (works on any plan, needs a token)

Use this if Option 1 isn't available, or if you want the full Monday API surface.

1. Create a Monday API token (Monday → your profile → Developers → My Access Tokens).
2. Export it in your shell rc:

   ```bash
   # ~/.zshrc or ~/.bashrc
   export MONDAY_TOKEN="your-token-here"
   ```

3. Register the MCP server in `~/.claude/settings.json`:

   ```jsonc
   {
     "mcpServers": {
       "monday-api-mcp": {
         "command": "npx",
         "args": ["-y", "@mondaydotcomorg/monday-api-mcp@latest"]
       }
     }
   }
   ```

   The subprocess inherits `MONDAY_TOKEN` from your environment — no `env` block needed.

4. Restart Claude Code. You should see tools like `mcp__monday-api-mcp__search`, `mcp__monday-api-mcp__get_updates` in a session.

**Heads up:** `claude mcp list` only shows locally-configured MCP servers — it may not show API-registered ones. The real test is whether the `mcp__monday-api-mcp__*` tools appear inside a Claude Code session.

## Credits

Five of these skills are derived from [mattpocock/skills](https://github.com/mattpocock/skills), with tweaks to target Obsidian instead of the original's GitHub-issues-based workflow:

- `grill-me`
- `tdd`
- `write-a-prd`
- `prd-to-slices`
- `improve-codebase-architecture`

Thanks Matt for open-sourcing your setup — it's the reason this repo exists in its current shape.

## License

MIT. See [`LICENSE`](./LICENSE).
