---
name: process-video-requests
description: Poll the ppingg Base44 app for pending VideoRequest records (status "send"), generate a 15-second English TV commercial for each with Higgsfield Seedance 2.0, and write the video URL back. Use when asked to process pending ad requests, check for new video requests, or run the ppingg TV-ad pipeline.
---

# Process ppingg VideoRequests

Constants:
- `appId`: `6910d4f137ee558974131c2c` (app: "PPNIGG - Connected TV to your perfect customer")
- Video model: `seedance_2_0` ONLY (never substitute another model without the owner's approval)
- Output: 15 seconds, 16:9, native audio ON, English voiceover
- Default resolution: `720p` std (production upgrade to `1080p` only after owner approves)

## 1. Fetch pending requests

`mcp__base44__query_entities` on entity `VideoRequest` with
`query: {"status": "send"}`, `sort: "created_date"`.

Skip (and do NOT bill credits for):
- Duplicates: same `businessName` + another pending/processing/done record newer than it —
  mark `status: "error"`, `errorMessage: "duplicate request"`.
- Records the owner explicitly parked (check `adminNotes`).

## 2. Validate each request

Required to proceed: `businessName`. Everything else is optional but shapes the ad:
- `businessPresence` = `physical` or `both` AND `businessAddress` non-empty → end-card shows the address.
- Otherwise → CTA is "Search us on Google" + exact business name.
- `logoUrl`: import with `mcp__higgsfield__media_import_url` and pass as `image_references`.
  If the URL is a generic site-builder placeholder or a tiny favicon (<100px), skip the logo
  rather than shipping a blurry mark.
- Use `summary` + `businessType` + `websiteUrl` for the creative concept.

Mark the record `status: "waiting"` (`$set`) before generating, so a second agent run
never double-bills the same request.

## 3. Write the 15-second script

English only. Structure (Seedance 2.0 takes the whole thing as one prompt):

- 0-3 s — Hook: one vivid scene that captures the business's core promise.
- 3-10 s — 2 quick scenes showing product/service in action, real-world setting.
- 10-15 s — End card: business logo, business name in large text, CTA voiceover.

Voiceover: warm, confident announcer, max ~35 spoken words total, written in
double quotes inside the prompt so Seedance lip-syncs/narrates it.

CTA rules (owner requirement):
- Always spoken: `"<Business Name> — search us on Google."`
- On-screen end card: business name + logo (if usable) + address (only if physical presence).

Prompt skeleton:

```
15-second polished TV commercial for "<BusinessName>", a <businessType>. <one-line summary>.
Scene 1 (0-3s): <hook scene>. Scene 2 (3-7s): <scene>. Scene 3 (7-10s): <scene>.
Scene 4 (10-15s): end card on clean background — the business logo and the name
"<BusinessName>" in bold text<, address "<businessAddress>" below it>.
Warm confident male/female announcer voiceover throughout: "<VO line 1> <VO line 2>
<BusinessName> — search us on Google."
Broadcast quality, bright commercial lighting, smooth camera moves, upbeat background music.
```

## 4. Generate

`mcp__higgsfield__generate_video` with:

```json
{"params": {
  "model": "seedance_2_0",
  "prompt": "<script prompt>",
  "duration": 15,
  "aspect_ratio": "16:9",
  "resolution": "720p",
  "generate_audio": true,
  "count": 1,
  "medias": [{"role": "image_references", "value": "<media_id from media_import_url>"}]
}}
```

- Preflight with `get_cost: true` first; abort and warn the owner if remaining credits < cost.
- Generation is async — poll the returned job until complete (job id also viewable via
  `mcp__higgsfield__job_display`). Typical wait: several minutes.

## 5. Write back to ppingg

On success (`mcp__base44__update_entities`, query by record `id`):

```json
{"$set": {"status": "done", "videoUrl": "<result mp4 url>", "externalRequestId": "<higgsfield job id>"}}
```

On failure:

```json
{"$set": {"status": "error", "errorMessage": "<reason>"}}
```

Then append a `UserActivityLog` record (`mcp__base44__create_entities`):
`eventType: "VideoReceivedSuccess"` (or `VideoReceivedError`), `status: "success"|"failed"`,
`relatedEntityId: <VideoRequest id>`, `userId`/`userEmail` copied from the request,
`details`: JSON string with job id, credits spent, model.

## 6. Report

Tell the owner: which requests were processed, credits spent, remaining balance
(`mcp__higgsfield__balance`), and the video URLs.
