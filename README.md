# clip-factory skill

Lets your AI agent — [Claude Code](https://claude.com/claude-code), [Hermes Agent](https://hermes-agent.nousresearch.com), or any of the [50+ agents the `skills` CLI supports](https://github.com/vercel-labs/skills) — drive your [clip-factory](https://clip-factory.app) account on your behalf: scrape YouTube channels, stitch clips with your CTA, and schedule them out to social.

## Install

```bash
npx skills add athcagithub/clip-factory-skill
```

This uses the [`skills`](https://github.com/vercel-labs/skills) CLI, which auto-detects your installed agents. To target one explicitly and skip prompts:

```bash
npx skills add athcagithub/clip-factory-skill -a claude-code -y    # Claude Code  → ~/.claude/skills/
npx skills add athcagithub/clip-factory-skill -a hermes-agent -y   # Hermes Agent → ~/.hermes/skills/
```

**API key.** Generate one at [clip-factory.app → Settings → Agent API key](https://clip-factory.app).
- **Hermes** prompts you for `CLIPFACTORY_KEY` automatically the first time you use the skill.
- **Claude Code** (and other agents without a secret prompt): the skill asks you to paste the key in chat and exports it for the session. To skip that each time, export it yourself first: `export CLIPFACTORY_KEY=cf_live_...` (add to `~/.zshrc` to persist).

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
