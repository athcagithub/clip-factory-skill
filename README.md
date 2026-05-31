# clip-factory skill

Lets your AI agent — [Claude Code](https://claude.com/claude-code), [Hermes Agent](https://hermes-agent.nousresearch.com), or any of the [50+ agents the `skills` CLI supports](https://github.com/vercel-labs/skills) — drive your [clip-factory](https://clip-factory.app) account on your behalf: scrape YouTube channels, stitch clips with your CTA, and schedule them out to social.

## Install

Use the [`skills`](https://github.com/vercel-labs/skills) CLI and **name your agent explicitly** — this is the reliable path:

```bash
npx skills add athcagithub/clip-factory-skill -a claude-code -y    # Claude Code  → ~/.claude/skills/
npx skills add athcagithub/clip-factory-skill -a hermes-agent -y   # Hermes Agent → ~/.hermes/skills/
```

> **Don't use the bare `npx skills add athcagithub/clip-factory-skill` for Claude Code.** Without `-a`, the CLI installs to the universal `~/.agents/skills/` directory — which **Claude Code does not read** — so the skill silently won't show up. Always pass `-a claude-code`. (In the interactive picker, the "Universal — always included" line does *not* cover Claude Code; you must tick Claude Code itself.)

Claude Code picks up the new skill within the session, or on next launch if it doesn't appear right away.

**Save your API key once.** Generate one at [clip-factory.app → Settings → Agent API key](https://clip-factory.app), then store it on your computer so no agent ever has to ask for it:

```bash
read -rsp 'clip-factory API key: ' k && mkdir -p ~/.clip-factory && printf '%s' "$k" > ~/.clip-factory/key && chmod 600 ~/.clip-factory/key && echo ' ✓ saved'
```

That's it — the skill reads the key from `~/.clip-factory/key` (or a `CLIPFACTORY_KEY` env var) on every run. If you skip this step, the skill simply asks for the key the first time you use it and saves it to the same place for you. The key stays on your machine — it's never committed or sent anywhere except the clip-factory API.

<details>
<summary>Manual install (no <code>npx</code>)</summary>

```bash
mkdir -p ~/.hermes/skills/clipfactory
curl -fsSL https://raw.githubusercontent.com/athcagithub/clip-factory-skill/main/SKILL.md \
  -o ~/.hermes/skills/clipfactory/SKILL.md
```

</details>

## Use

Just talk to your agent:

> *"Make 50 vids of @mrbeast and schedule one every 6 hours starting Monday."*

> *"What's my clip-factory quota?"*

> *"Scrape @whoever every Monday at 9am and queue them out."* (use your agent's scheduler — e.g. Hermes' built-in cron, or `/loop` / scheduled runs in Claude Code — for recurring tasks.)

## What it wraps

The skill describes the public `/v1/*` surface at `https://api.clip-factory.app/v1`. The agent calls it directly using its built-in terminal/HTTP tools — no separate runtime, no MCP server, no glue code.

Auth is a bearer token (`Authorization: Bearer cf_live_...`). Each key is tied to a single clip-factory user and is gated on an active subscription; revoking a key or letting your subscription lapse cuts the agent off within seconds.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/quota` | Remaining credits + plan info |
| GET | `/projects` | List your clip-factory projects |
| GET | `/projects/:id` | Single project details (CTA presence, caption) |
| GET | `/accounts` | Connected social accounts (Postiz / Post-Bridge) |
| POST | `/scrape` | Scrape N shorts from a channel into a project |
| GET | `/clips` | List clips with status filter |
| POST | `/stitch` | Stitch clips with the project's CTA |
| POST | `/schedule` | Schedule a stitched clip for posting |

Full request/response shapes, error codes, and recipe procedures live in [`SKILL.md`](./SKILL.md).

## Limitations

- **CTA upload is web-only.** Multipart file upload doesn't fit cleanly into an agent loop, and CTA setup is a one-time-per-project task. If the project has no CTA, the API returns `cta_missing` and the agent will instruct the user to upload one on the website.
- **One active key per user.** Generating a new key revokes the previous one. This is enforced server-side; if you want multiple machines, share the same key.

## License

MIT — see [`LICENSE`](./LICENSE).
