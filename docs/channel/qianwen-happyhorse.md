# Qianwen HappyHorse Channel

This document records the maintained fork behavior for routing QIANWEN-WEB-01 through NewAPI.

## Channel

Configure QIANWEN-WEB-01 as an OpenAI-compatible channel:

| Field | Value |
|---|---|
| Channel type | OpenAI-compatible |
| Base URL | `http://<qianwen-web-host>:18002` |
| API key | QIANWEN-WEB-01 service API key |
| Models | `tongyi-qwen3-max-model`, `tongyi-qwen3-max-thinking`, `Qwen-Image-2.0`, `HappyHorse 1.0` |
| Video model | `HappyHorse 1.0` |

The QIANWEN-WEB-01 provider owns all qianwen.com Web details: login cookies, browser runtime upload, ACS signing, material ids, task polling, SQLite task records, and prompt encoding checks. NewAPI should treat it as an OpenAI-compatible upstream and should not implement qianwen.com private upload/signature logic.

## Supported Video Routes

The tested NewAPI routes are:

```text
POST /v1/video/generations
GET  /v1/video/generations/{task_id}
```

The current upstream also accepts OpenAI-style `/v1/videos`, but the deployment test was performed through `/v1/video/generations` because that is the route used by the existing provider-side video adapters.

## Two-Image Request Example

```json
{
  "model": "HappyHorse 1.0",
  "prompt": "Use the first image as the opening frame. Use the second image as the visual reference and ending target. Create a five second 720P cinematic transition video.",
  "duration": 5,
  "size": "16:9",
  "aspect_ratio": "16:9",
  "resolution": "720P",
  "first_frame_image": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
  "reference_images": [
    "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..."
  ]
}
```

QIANWEN-WEB-01 converts `first_frame_image`, `image_url`, `image`, `file_id`, and `reference_images` into qianwen.com official `attachments[].materialId` entries. Public URLs, data URIs, and raw base64 images are uploaded through the qianwen.com Web runtime before the task is submitted.

## Validation

Validation performed on 2026-06-21:

| Item | Result |
|---|---|
| NewAPI task | `task_GW8rRGXBlR1xNW8tNhcBpUBy22BaD2X9` |
| Upstream QIANWEN-WEB-01 task | `4fa38a69-326d-4abc-aa35-c45ca85f6067` |
| Channel | QianWen, channel id `6` in the tested SQLite deployment |
| Model | `HappyHorse 1.0` |
| Parameters | `duration=5`, `resolution=720P`, `aspect_ratio=16:9` |
| Input | 2 PNG images, sent as `first_frame_image` and `reference_images[0]` data URI |
| NewAPI status | `SUCCESS` |
| QIANWEN-WEB-01 status | `succeeded` |
| Upstream payload | 2 official `attachments[].materialId` values |
| Prompt encoding | No `????` corruption in request or upstream payload |
| Output | 2 reachable `video/mp4` URLs |

Validated upstream material ids:

```json
[
  {"materialId": "348539b51d4546779763c2bee9a0146d", "type": "image"},
  {"materialId": "db8fe86fdcb64f258617640e0ff144c6", "type": "image"}
]
```

## Current Caveat

Task polling returns the CDN video URLs correctly. The optional NewAPI content proxy endpoint:

```text
GET /v1/videos/{task_id}/content
```

depends on NewAPI fetch/SSRF settings and may return an allowed-port or domain-filter error if the resolved provider URL or intermediate provider endpoint is not whitelisted. This does not affect task submission or polling; clients can use the returned `url` / `video_url` values directly.

