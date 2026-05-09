# personal-agents

Personal AI agents that run on a schedule and surface useful context to me.

## Current agents

| Agent | Trigger | Delivery | Spec file |
|---|---|---|---|
| Personal morning brief | Daily at 7:00 AM ET | Telegram push | [`CLAUDE.md`](./CLAUDE.md) |

## Architecture

The morning brief runs as a [Claude Code Routine](https://code.claude.com/docs/en/routines) on Anthropic-managed cloud infrastructure. Each run:

1. Anthropic clones this repo into a sandboxed cloud environment
2. Claude Code auto-loads `CLAUDE.md` and follows the instructions inside
3. It calls the Gmail and Google Calendar MCP connectors (read-only intent) to gather today's context
4. It composes a brief following the structure defined in `CLAUDE.md`
5. It POSTs the brief to my Telegram bot via `api.telegram.org` using a `curl` command and credentials from environment variables

The repo only contains specs. No code, no secrets. The runtime, scheduling, connectors, and credentials all live in the routine config at [claude.ai/code/routines](https://claude.ai/code/routines).

## Iteration loop

The morning brief is a living spec — adjust it as the brief evolves.

To change agent behavior:

1. Edit `CLAUDE.md` (web editor at [github.com/21shreyash/personal-agents/edit/main/CLAUDE.md](https://github.com/21shreyash/personal-agents/edit/main/CLAUDE.md) or local terminal)
2. Commit with a descriptive message (`fix: deprioritize India financial admin noise`)
3. Push to `main`
4. The next routine run automatically picks up the change

Commit log doubles as a behavioral changelog. Use it.

## Credentials audit

| Credential | Where it lives | Expiry | What breaks if expired |
|---|---|---|---|
| GitHub PAT (classic) | macOS Keychain | 90 days from creation | Local `git push` from terminal. Routine unaffected (uses Claude GitHub App). |
| Telegram bot token | Routine env var `TELEGRAM_BOT_TOKEN` | Never (manual revoke only) | Brief delivery to Telegram. Rotate via BotFather `/revoke` → update env var. |
| Telegram chat ID | Routine env var `TELEGRAM_CHAT_ID` | Never | Brief delivery. Stable unless I delete my Telegram account. |
| Gmail/Calendar OAuth | Anthropic-managed | No fixed schedule | Brief generation. Re-auth at [claude.ai/settings/connectors](https://claude.ai/settings/connectors). |

**Active reminders:**
- 📅 Aug 5, 2026 — rotate GitHub PAT (or upgrade to SSH keys, ~15 min one-time)

## Network policy

The routine's environment uses **Custom** network access. Allowed domains:
- Default package-manager allowlist (preserved)
- `api.telegram.org` — for brief delivery

If a future agent needs to reach a new domain, add it via [routine settings](https://claude.ai/code/routines) → environment → Allowed domains. Outbound requests to non-allowlisted hosts fail with `403 host_not_allowed`.

## Debugging

When the morning brief doesn't arrive or behaves unexpectedly:

1. Open [claude.ai/code/routines](https://claude.ai/code/routines) → `Personal morning brief` → click the most recent run.
2. The session log shows every step Claude took, every shell command run, every tool call made.
3. Common failure signatures:
   - **No Telegram call happened** → check whether Claude reached the "Final step" section of `CLAUDE.md`. Tighten the wording if delivery was treated as optional.
   - **`403 host_not_allowed`** → network policy issue. Verify `api.telegram.org` is in Allowed domains.
   - **Telegram returns `ok: false, 401`** → bot token is stale/wrong. Regenerate via BotFather, update env var.
   - **Telegram returns `ok: false, 400 "chat not found"`** → chat ID wrong, OR I never `/start`-ed the bot in Telegram.
   - **Connector auth error** → re-authenticate Gmail/Calendar at `claude.ai/settings/connectors`.

A green "success" status on a run means the session ended without infrastructure errors. It does **not** mean the brief was delivered. Always verify with the Telegram message itself.

## Adding new agents

Future agents (work-side brief, weekly review, evening wrap-up, etc.) can live in this repo using one of two patterns:

- **Single-file:** add a sibling spec like `WEEKLY-REVIEW.md` and a separate routine that loads it via prompt: *"Read WEEKLY-REVIEW.md and execute."*
- **Subdirectory:** if an agent needs multiple files (helper specs, templates), give it its own folder: `weekly-review/CLAUDE.md`, `weekly-review/template.md`, etc.

Each agent should have its own routine, its own connector scope (least-privilege), and its own section in this README's "Current agents" table.

## Out of scope for this repo

- Float work email, Slack, or any work-related context. Work-side automation is a separate project, separate repo, requires conversation with Float IT/security.
- Anything financial, banking, or transactional. This repo is read-and-summarize only.
