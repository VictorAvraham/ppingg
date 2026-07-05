# ppingg TV-Ad Agent

Automation agent that connects the **ppingg** app (built on Base44) to **Higgsfield Seedance 2.0** for automatic TV-commercial generation.

## What it does

Every "create ad" action in ppingg — the admin CRM button or the customer journey — flows
through one pipeline (deployed in the Base44 app on 2026-07-05):

1. `createVideoDirect` (Base44 backend function) builds the creative: `generateAdScript`
   produces a cinematic motion prompt + an **English announcer voiceover ending with the
   spoken CTA** ("Search us on Google" / brand + address), and an AI opening-frame image
   is generated for the business.
2. The function **enqueues an `AgentVideoJob`** record (`status: "queued"`) and returns an
   `agent-<uuid>` ticket to the frontend — which keeps polling `getVideoStatus` unchanged.
3. **This agent** picks up queued jobs, generates the video with **Seedance 2.0 only**
   (`seedance_2_0` via the Higgsfield MCP — 15 s, 16:9, native audio, 720p), and writes
   `videoUrl` + `status: "done"` back to the job.
4. `getVideoStatus` / `syncPendingVideos` resolve `agent-` tickets from the queue, complete
   the `VideoRequest`, update the user's video list, and log activity.

The legacy path (Kling 3.0, 5 s, no voiceover, via `platform.higgsfield.ai`) was replaced:
the old platform API only exposes Seedance 1.x (max 12 s), so Seedance 2.0 runs through the
Higgsfield MCP on the agent side.

## Architecture

Both integrations run as MCP connectors inside a Claude Code session — there is no
standalone server in this repo:

| Side | Connection | Auth |
|------|-----------|------|
| ppingg / Base44 | Base44 MCP connector (`query_entities`, `update_entities`, file/deploy tools) | Base44 account OAuth |
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
