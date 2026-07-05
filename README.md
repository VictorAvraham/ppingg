# ppingg TV-Ad Agent

Automation agent that connects the **ppingg** app (built on Base44) to **Higgsfield Seedance 2.0** for automatic TV-commercial generation.

## What it does

1. Watches the `VideoRequest` entity in the ppingg Base44 app (`PPNIGG - Connected TV to your perfect customer`, appId `6910d4f137ee558974131c2c`) for records with `status: "send"`.
2. Validates the business data (name, type, summary, address, logo).
3. Writes a 15-second English TV-commercial script with voiceover and a CTA
   ("Search us on Google" / business name + logo + address).
4. Generates the video with **Seedance 2.0 only** (`seedance_2_0`) via the Higgsfield MCP,
   16:9, 15 s, native audio (voiceover) enabled.
5. Writes the result back to the same `VideoRequest` record:
   `videoUrl`, `externalRequestId`, `status: "done"` (or `status: "error"` + `errorMessage`),
   and appends a `UserActivityLog` record.

## Architecture

Both integrations run as MCP connectors inside a Claude Code session — there is no
standalone server in this repo:

| Side | Connection | Auth |
|------|-----------|------|
| ppingg / Base44 | Base44 MCP connector (`query_entities`, `update_entities`, `create_entities`) | Base44 account OAuth |
| Higgsfield | `https://mcp.higgsfield.ai/mcp` connector (`generate_video`, `media_import_url`) | Higgsfield account OAuth |

The full operating procedure lives in
[`.claude/skills/process-video-requests/SKILL.md`](.claude/skills/process-video-requests/SKILL.md) —
any Claude Code session in this repo can run it with `/process-video-requests`.

## Generation cost (measured 2026-07-05)

15-second, 16:9, 720p, one video:

| Model / mode | Credits |
|---|---|
| `seedance_2_0` std | 67.5 |
| `seedance_2_0` fast | 52.5 |
| `seedance_2_0_mini` | 37.5 |

## Status values on VideoRequest

- `send` — request submitted by user/admin, waiting for the agent
- `waiting` — agent picked it up, generation in progress
- `done` — `videoUrl` populated
- `error` — see `errorMessage`
