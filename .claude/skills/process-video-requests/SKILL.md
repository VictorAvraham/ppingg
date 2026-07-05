---
name: process-video-requests
description: Process the ppingg TV-ad queue - pick up queued AgentVideoJob records from the Base44 app, generate a 15-second English TV commercial with Higgsfield Seedance 2.0 via MCP, and write the video URL back. Use when asked to process pending ad requests, check the video queue, or run the ppingg TV-ad pipeline.
---

# Process ppingg TV-ad queue (AgentVideoJob)

Constants:
- `appId`: `6910d4f137ee558974131c2c` (app: "PPNIGG - Connected TV to your perfect customer")
- Video model: `seedance_2_0` via Higgsfield MCP ONLY (owner requirement — never substitute)
- Output: 15 seconds, 16:9, native audio ON (English announcer voiceover), 720p std
  (production may upgrade resolution only with owner approval)
- Cost reference: ~67.5 credits per 15s std 720p video. ALWAYS preflight with `get_cost: true`
  and abort + warn the owner if remaining balance < cost.

## How the app side works (already deployed in Base44)

- Admin CRM ("Request New Video") and the customer journey (BuildVideo page) both call the
  `createVideoDirect` backend function.
- `createVideoDirect` runs `generateAdScript` (LLM brain → motion prompt + English
  `voiceover_script` ending with the spoken CTA), generates an opening-frame concept image,
  then **enqueues an `AgentVideoJob` record** (`status: "queued"`) with the full Seedance
  prompt, and returns an `agent-<uuid>` ticket as `video_id`.
- Frontends poll `getVideoStatus` which resolves `agent-` tickets from `AgentVideoJob`.
- `syncPendingVideos` completes `VideoRequest` records with `agent-` tickets app-side
  (sets `done`, updates `User.videoUrls` / `lastVideoUrl`, logs activity).

## Agent loop (this is your job)

1. **Fetch queue**: `mcp__base44__query_entities` on `AgentVideoJob`,
   `query: {"status": "queued"}`, `sort: "created_date"`.
   Skip any job whose `errorMessage` mentions "smoke test".
2. **Claim**: set `status: "generating"` (`$set` via `mcp__base44__update_entities`,
   query by `ticketId`) BEFORE generating, so a parallel run never double-bills.
3. **Media**: `mcp__higgsfield__media_import_url` on `sourceImageUrl` → role `start_image`.
   If `logoUrl` is a real raster logo (not favicon/placeholder), import → role `image_references`.
4. **Preflight cost**: `mcp__higgsfield__generate_video` with `get_cost: true` and the exact
   params below. Check `mcp__higgsfield__balance` covers it. If credits are insufficient:
   - Set the job back to `status: "queued"` (do not consume it).
   - Invoke the app's admin alert so the team gets an email via Resend:
     `POST https://ad-craft-tv-copy-74131c2c.base44.app/api/functions/alertAdminLowCredits`
     with header `api_key: <BASE44_API_KEY>` and body
     `{"creditsRemaining": <balance>, "creditsNeeded": <cost>, "serialNumber": "<job serial>", "businessName": "<name>"}`.
   - Tell the owner in chat, then stop (the job stays queued and resumes automatically after top-up).
5. **Generate**:
   ```json
   {"params": {
     "model": "seedance_2_0",
     "prompt": "<job.prompt — already contains motion + voiceover>",
     "duration": 15,
     "aspect_ratio": "16:9",
     "resolution": "720p",
     "generate_audio": true,
     "count": 1,
     "medias": [
       {"role": "start_image", "value": "<media_id of sourceImageUrl>"},
       {"role": "image_references", "value": "<media_id of logo, if usable>"}
     ]
   }}
   ```
6. **Poll** the returned job until complete (typically several minutes).
7. **Write back** on success (`$set` on the AgentVideoJob by `ticketId`):
   `status: "done"`, `videoUrl`, `higgsfieldJobId`, `creditsSpent`.
   Also update the matching `VideoRequest` (query `{"externalRequestId": "<ticketId>"}`):
   `status: "done"`, `videoUrl` — belt-and-braces alongside `syncPendingVideos`.
   On failure: `status: "error"`, `errorMessage` on both records.
8. **Log**: create a `UserActivityLog` record — `eventType: "VideoReceivedSuccess"` /
   `"VideoReceivedError"`, `relatedEntityId`, `userId`/`userEmail` from the job,
   `details` JSON with higgsfield job id + credits.
9. **Report** to the owner (Hebrew): business name, video URL, credits spent,
   remaining balance.

## Legacy queue

Old `VideoRequest` records in `status: "send"` (created before 2026-07-05, e.g. CandidaFree,
GLOVESLINE ×2, Karavel ×4, John's of Bleecker ×2, Super Duper) have NO AgentVideoJob and NO
prompt. Do NOT process them without explicit owner approval — several are duplicates or have
placeholder logos. If approved: build the prompt yourself per the structure in `job.prompt`
examples (hook → value scenes → end card with logo/name/address + "Search us on Google",
English VO ≤35 words), then follow steps 3-9, updating the VideoRequest directly.

## Watcher

While a session is live, poll every ~2 minutes (in-session cron). In-session crons DIE on
session restart — after any restart, re-arm the watcher and CronList to verify.
