# ima-sevio-ai

IMA Sevio video generation skill for OpenClaw / ClawHub style runtimes.

## What it does

- Exposes two user-facing models:
  - `Ima Sevio 1.0`
  - `Ima Sevio 1.0-Fast`
- Supports task types:
  - `text_to_video`
  - `image_to_video`
  - `first_last_frame_to_video`
  - `reference_image_to_video`
- Uses product list as runtime source of truth for parameter matching.
- Supports local input file upload to OSS and direct pass-through for remote URLs.

## Install

```bash
git clone https://github.com/imastuido/ima-sevio-ai.git
```

## Runtime requirement

- `IMA_API_KEY`

## Main files

- `SKILL.md`
- `scripts/ima_video_create.py`
- `scripts/ima_logger.py`
- `_meta.json`

## Notes

- User-facing model names should prefer Sevio naming.
- Internal model IDs are mapped by the script at runtime.
