# clip-factory skill for Hermes Agent

Lets [Hermes Agent](https://hermes-agent.nousresearch.com) drive your [clip-factory](https://clip-factory.app) account on your behalf — scrape YouTube channels, stitch clips with your CTA, and schedule them out to social.

## Install

```bash
hermes skills tap add athcagithub/clip-factory-skill
```

Hermes will prompt you for `CLIPFACTORY_KEY` during install. Generate one at [clip-factory.app → Settings → Hermes Agent API key](https://clip-factory.app).

## Use

Inside Hermes:

> *"Make 50 vids of @mrbeast and schedule one every 6 hours starting Monday."*

> *"What's my clip-factory quota?"*

> *"Scrape @whoever every Monday at 9am and queue them out."* (use Hermes' built-in cron for recurring tasks.)

## What it wraps

The skill describes the public `/api/v1/*` surface at `https://api.clip-factory.app/v1`. Hermes calls it directly using its built-in terminal/HTTP tools — no separate runtime, no MCP server, no glue code.

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
