---
name: IMA Sevio AI Generation
version: 1.1.0
category: file-generation
author: IMA Studio (imastudio.com)
keywords: imastudio, video generation, text-to-video, image-to-video, IMA, Ima Sevio, Sevio, IMA Video Pro, IMA Video Pro Fast
argument-hint: "[text prompt or media URL]"
description: >
  IMA model generation with exactly two Sevio models: Ima Sevio 1.0 and Ima Sevio 1.0-Fast.
  Supports text-to-video, image-to-video, first-last-frame, and reference-image workflows.
  Keeps the same API flow, reflection retry mechanism, and interface contract as ima-video-ai.
  Requires IMA API key.
---

# IMA Sevio AI Creation

## 🎯 Skill Capabilities

本技能面向 **IMA 视频生成**，提供统一入口，能力范围如下：

- **模型能力**：仅对用户暴露 **Ima Sevio 1.0**、**Ima Sevio 1.0-Fast** 两个模型（内部自动映射模型 ID）。
- **任务能力**：支持 `text_to_video`、`image_to_video`、`first_last_frame_to_video`、`reference_image_to_video`。
- **输入能力**：
  - `prompt`：文本描述。
  - `--input-images`：统一承载引用资源（image/video/audio URL 或本地文件路径）。
  - 本地资源自动上传 OSS，远程 HTTP(S) 链接直接透传。
- **参数能力**：自动读取产品配置（`credit_rules`、`form_config`）并匹配 `attribute_id / credit / parameters`。
- **执行能力**：内置反省重试机制（参数降级、规则重选、超时建议）与任务轮询。
- **输出能力**：返回任务结果 URL（用于直接回传用户端渲染/播放）。

---

## ✨ Expected Outcomes & Boundaries (Outcomes, Timing, Scope Limits)

### Expected Outcomes

- **文生视频**：可快速得到可播放视频结果，适合创意验证、脚本草稿、素材生成。
- **图生视频 / 参考视频**：可提升主体连续性与风格一致性（相比纯文本）。

### Timing Expectations

| 模型（用户展示） | 典型耗时 |
|---|---:|
| Ima Sevio 1.0（IMA Video Pro） | 120~300s |
| Ima Sevio 1.0-Fast（IMA Video Pro Fast） | 60~120s |

轮询超时上限：`40 分钟（2400s）`。

### Capability Boundaries (Avoid Misunderstanding)

- 本技能只做 **视频生成链路**，不负责后期剪辑、自动分镜编排、成片包装。
- 结果质量受提示词、参考素材质量、当次产品规则与积分策略影响，不保证每次一致。
- 仅支持本技能白名单模型；其他模型名会被拦截或映射后执行。

---

## ⚠️ 内部调用：模型 ID 参考（不对用户展示）

**User-facing rule:** In user messages, always use **Ima Sevio 1.0 / Ima Sevio 1.0-Fast** names.  
Do not expose raw `model_id` unless the user explicitly asks for technical details.

**CRITICAL:** When calling the script, you MUST use exact `model_id` values. For `ima-sevio-ai`, only these two are allowed:

| Friendly Name | model_id | Notes |
|---|---|---|
| IMA Pro | `ima-pro` | Default quality model |
| IMA Pro Fast | `ima-pro-fast` | Faster / lower-latency model |
| Ima Sevio 1.0 | `ima-pro` | Display-name alias |
| Ima Sevio 1.0-Fast | `ima-pro-fast` | Display-name alias |

### 模型中文介绍（可公开口径）

**IMA Video Pro（Ima Sevio 1.0）**

视频生成，从此不用等。  
自研 IMA Video Pro——对标行业同类最强视频模型，在画质一致性、镜头控制、多模态输入等核心维度达到同等水准，同时将生成时长压缩至秒级响应。  
不是更快的妥协，是更强的起点。IMA Video Pro，现已上线 IMA Studio。

**核心优势（公开可查）**
- 高帧率时序一致性
- 精准镜头语言控制
- 图像 / 音频 / 文本多模态输入
- 2K 级输出画质

**IMA Video Pro Fast（Ima Sevio 1.0-Fast）**

面向“快速出片与高频迭代”场景的加速模型版本。  
在保持视频质量与可控性的前提下，进一步优化生成时延，适合批量试风格、快速打样和实时创作流程。

**Rules:**
- Do NOT infer model IDs from other IMA skills.
- Do NOT use any model outside this allowlist.
- If user asks for other models, map to one of the two allowed models with explanation.
- Alias input `Ima Sevio 1.0` is auto-mapped to `ima-pro`.
- Alias input `Ima Sevio 1.0-Fast` is auto-mapped to `ima-pro-fast`.

---

## ⚠️ MANDATORY PRE-CHECK: Read Knowledge Base First!

**If ima-knowledge-ai is not installed:** Skip all "Read …" steps below; use only this SKILL's default parsing tables.

**BEFORE executing ANY video generation task, you MUST:**

1. **Understand video modes** — Read `ima-knowledge-ai/references/video-modes.md`:
- `image_to_video` = input image becomes frame 1
- `reference_image_to_video` = input image is visual reference, not frame 1

2. **Check visual consistency needs** — Read `ima-knowledge-ai/references/visual-consistency.md` if user mentions:
- "系列"、"分镜"、"同一个"、"角色"、"续"、"多个镜头"
- multi-shot continuity, character consistency, repeated subject

3. **Check workflow/model/parameters** — Read related references when unsure about mode or parameters.

**Why this matters:**
- AI generation is independent by default.
- Text-only generation cannot preserve visual continuity reliably.
- Wrong mode choice causes wrong results.

---

## 📥 User Input Parsing (Model & Parameter Recognition)

### 1) User phrasing → `task_type`

| User intent | task_type |
|---|---|
| Only text | `text_to_video` |
| One image as first frame | `image_to_video` |
| One image as reference | `reference_image_to_video` |
| Two images as first+last frame | `first_last_frame_to_video` |

### 2) User phrasing → `model_id`

Normalize case-insensitively and ignore spaces:

| User says | model_id |
|---|---|
| `ima-pro`, `pro`, `专业版`, `高质量` | `ima-pro` |
| `ima-pro-fast`, `fast`, `极速`, `快速` | `ima-pro-fast` |
| `Ima Sevio 1.0` | `ima-pro` |
| `Ima Sevio 1.0-Fast` | `ima-pro-fast` |
| "默认" / "推荐" / "自动" | `ima-pro` |

If user explicitly asks "faster", prefer `ima-pro-fast`.
If user explicitly asks "best quality", prefer `ima-pro`.

### 3) User phrasing → duration / resolution / aspect_ratio

| User says | Parameter | Normalized value |
|---|---|---|
| 5秒 / 5s | duration | 5 |
| 10秒 / 10s | duration | 10 |
| 15秒 / 15s | duration | 15 |
| 横屏 / 16:9 | aspect_ratio | 16:9 |
| 竖屏 / 9:16 | aspect_ratio | 9:16 |
| 方形 / 1:1 | aspect_ratio | 1:1 |
| 720P / 720p | resolution | 720P |
| 1080P / 1080p | resolution | 1080P |
| 4K / 4k | resolution | 4K (only if model/rule supports) |

If unspecified, use product `form_config` defaults.

---

## ⚙️ How This Skill Works

This skill uses bundled script `scripts/ima_video_create.py` and keeps original API workflow:
- product list query
- parameter resolution
- create task
- poll task detail
- return video URL

### 🌐 Network Endpoints Used

| Domain | Purpose | What's Sent |
|---|---|---|
| `api.imastudio.com` | task create + status polling | prompt, model params, task IDs, API key |
| `imapi.liveme.com` | image upload (when image input exists) | image bytes, API key |

**Privacy notes:**
- API key is sent to both domains for auth.
- `--user-id` is local-only and not sent to IMA servers.
- Local files: preferences and logs in `~/.openclaw`.

---

## Agent Execution (Internal)

```bash
# Text to video
python3 {baseDir}/scripts/ima_video_create.py \
  --api-key $IMA_API_KEY \
  --task-type text_to_video \
  --model-id ima-pro \
  --prompt "a puppy runs across a sunny meadow, cinematic" \
  --user-id {user_id} \
  --output-json

# Image to video
python3 {baseDir}/scripts/ima_video_create.py \
  --api-key $IMA_API_KEY \
  --task-type image_to_video \
  --model-id ima-pro-fast \
  --prompt "camera slowly zooms in" \
  --input-images https://example.com/photo.jpg \
  --user-id {user_id} \
  --output-json

# First-last frame to video
python3 {baseDir}/scripts/ima_video_create.py \
  --api-key $IMA_API_KEY \
  --task-type first_last_frame_to_video \
  --model-id ima-pro \
  --prompt "smooth transition" \
  --input-images https://example.com/first.jpg https://example.com/last.jpg \
  --user-id {user_id} \
  --output-json

```

`--input-images` accepts remote HTTP(S) links and local file paths.
Local image files are uploaded to OSS first; non-local HTTP(S) links are assigned directly.
CLI form is space-separated arguments; equivalent JSON form is:
`["https://example.com/ref1.jpg","https://example.com/ref2.jpg"]`.

---

## 🚨 CRITICAL: How to send video to user

Always send remote URL directly:

```python
video_url = json_output["url"]
message(action="send", media=video_url, caption="✅ 视频生成成功")
```

Do NOT download to local file before sending.

---

## 🧠 User Preference Memory

Storage: `~/.openclaw/memory/ima_prefs.json`

```json
{
  "user_{user_id}": {
    "text_to_video": {"model_id": "ima-pro", "model_name": "IMA Pro", "credit": 0, "last_used": "..."},
    "image_to_video": {"model_id": "ima-pro-fast", "model_name": "IMA Pro Fast", "credit": 0, "last_used": "..."},
    "first_last_frame_to_video": {"model_id": "ima-pro", "model_name": "IMA Pro", "credit": 0, "last_used": "..."},
    "reference_image_to_video": {"model_id": "ima-pro", "model_name": "IMA Pro", "credit": 0, "last_used": "..."}
  }
}
```

Model selection priority:
1. user preference
2. knowledge-ai recommendation
3. fallback default (`ima-pro`)

### Defaults

| Task | Default | Alt (fast) |
|---|---|---|
| text_to_video | `ima-pro` | `ima-pro-fast` |
| image_to_video | `ima-pro` | `ima-pro-fast` |
| first_last_frame_to_video | `ima-pro` | `ima-pro-fast` |
| reference_image_to_video | `ima-pro` | `ima-pro-fast` |

---

## 💬 User Experience Protocol (IM / Feishu / Discord)

### Estimated Generation Time

| Model | Estimated Time | Poll Every | Send Progress Every |
|---|---:|---:|---:|
| ima-pro | 120~300s | 8s | 45s |
| ima-pro-fast | 60~120s | 8s | 30s |

Polling timeout upper bound: **40 minutes** (`2400s`).

Use:
- Step 1: pre-generation notice (model/time/credits)
- Step 2: progress updates
- Step 3: success push (video first, then shareable link)
- Step 4: failure message with actionable retry options

Progress formula:

```text
P = min(95, floor(elapsed_seconds / estimated_max_seconds * 100))
```

---

## Step 4 — Failure Notification

Translate technical errors to user language. For 401/4008 include links:
- API key: https://www.imaclaw.ai/imaclaw/apikey
- credits: https://www.imaclaw.ai/imaclaw/subscription

### Enhanced Error Handling (Reflection)

The script keeps the same reflection mechanism (up to 3 retries):
- `500` → parameter degradation
- `6009` → auto-complete missing params from matched rules
- `6010` → reselect matching credit rule
- timeout → actionable guidance

### Fallback suggestion table

| Failed model | First alt | Second alt |
|---|---|---|
| `ima-pro` | `ima-pro-fast` | `ima-pro` (retry with downgraded params) |
| `ima-pro-fast` | `ima-pro` | `ima-pro-fast` (retry with defaults) |
| unknown | `ima-pro` | `ima-pro-fast` |

---

## Supported Models

Only two models are exposed by this skill:
- `ima-pro`
- `ima-pro-fast`

Supported categories:
- `text_to_video`
- `image_to_video`
- `first_last_frame_to_video`
- `reference_image_to_video`

> Attribute rules, points, and exact parameter combinations must be queried at runtime from product list.

---

## Environment

Base URL: `https://api.imastudio.com`

Required headers:
- `Authorization: Bearer ima_your_api_key_here`
- `x-app-source: ima_skills`
- `x_app_language: en` (or `zh`)

---

## ⚠️ MANDATORY: Always Query Product List First

You MUST call `/open/v1/product/list` before creating tasks.
`attribute_id` and `credit` must match current rule set.

Common failures if skipped:
- invalid product attribute
- insufficient points
- `6006`, `6010`

---

## Core Flow

```text
1) GET /open/v1/product/list
2) (if image input) upload image(s) -> HTTPS CDN URL(s)
3) POST /open/v1/tasks/create
4) POST /open/v1/tasks/detail (poll every 8s)
```

---

## Image Upload

For image tasks, source images must resolve to public HTTPS URLs.
Bundled script supports local file path and uploads automatically.

---

## API 1: Product List

`GET /open/v1/product/list?app=ima&platform=web&category=<task_type>`

Use type=3 leaf nodes to read:
- `model_id`
- `id` (`model_version`)
- `credit_rules[]`
- `form_config[]`

---

## API 2: Create Task

`POST /open/v1/tasks/create`

### text_to_video (example)

```json
{
  "task_type": "text_to_video",
  "enable_multi_model": false,
  "src_img_url": [],
  "parameters": [
    {
      "attribute_id": 1234,
      "model_id": "ima-pro",
      "model_name": "IMA Pro",
      "model_version": "ima-pro",
      "app": "ima",
      "platform": "web",
      "category": "text_to_video",
      "credit": 25,
      "parameters": {
        "prompt": "a puppy dancing happily",
        "duration": 5,
        "resolution": "1080P",
        "aspect_ratio": "16:9",
        "n": 1,
        "input_images": [],
        "cast": {"points": 25, "attribute_id": 1234}
      }
    }
  ]
}
```

For image tasks, keep top-level `src_img_url` and nested `input_images` consistent.

---

## API 3: Task Detail

`POST /open/v1/tasks/detail` with `{ "task_id": "..." }`

Status interpretation:
- `resource_status`: `0/null` processing, `1` ready, `2` failed, `3` deleted
- Stop only when all medias have `resource_status == 1` and none failed

---

## Common Mistakes

- Polling too fast (use 8s)
- Missing required nested fields (`prompt`, `cast`, `n`)
- Credit/attribute mismatch (`6006` / `6010`)
- Inconsistent `src_img_url` and `input_images`
- Wrong mode choice (`image_to_video` vs `reference_image_to_video`)

---

## Python Example

```python
import time
import requests

BASE_URL = "https://api.imastudio.com"
API_KEY = "ima_your_key_here"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
    "x-app-source": "ima_skills",
    "x_app_language": "en",
}

ALLOWED = {"ima-pro", "ima-pro-fast"}


def get_products(category: str) -> list:
    r = requests.get(
        f"{BASE_URL}/open/v1/product/list",
        headers=HEADERS,
        params={"app": "ima", "platform": "web", "category": category},
    )
    r.raise_for_status()
    nodes = r.json().get("data", [])
    leaves = []

    def walk(items):
        for n in items:
            if n.get("type") == "3" and n.get("model_id") in ALLOWED:
                leaves.append(n)
            walk(n.get("children") or [])

    walk(nodes)
    return leaves


def create_video_task(task_type: str, prompt: str, product: dict, src_img_url=None, **extra) -> str:
    src_img_url = src_img_url or []
    rule = product["credit_rules"][0]
    defaults = {f["field"]: f["value"] for f in product.get("form_config", []) if f.get("value") is not None}

    params = {
        "prompt": prompt,
        "n": 1,
        "input_images": src_img_url,
        "cast": {"points": rule["points"], "attribute_id": rule["attribute_id"]},
        **defaults,
    }
    params.update(extra)

    payload = {
        "task_type": task_type,
        "enable_multi_model": False,
        "src_img_url": src_img_url,
        "parameters": [{
            "attribute_id": rule["attribute_id"],
            "model_id": product["model_id"],
            "model_name": product["name"],
            "model_version": product["id"],
            "app": "ima",
            "platform": "web",
            "category": task_type,
            "credit": rule["points"],
            "parameters": params,
        }],
    }

    r = requests.post(f"{BASE_URL}/open/v1/tasks/create", headers=HEADERS, json=payload)
    r.raise_for_status()
    return r.json()["data"]["id"]


def poll(task_id: str, interval: int = 8, timeout: int = 600) -> dict:
    deadline = time.time() + timeout
    while time.time() < deadline:
        r = requests.post(f"{BASE_URL}/open/v1/tasks/detail", headers=HEADERS, json={"task_id": task_id})
        r.raise_for_status()
        task = r.json().get("data", {})
        medias = task.get("medias", [])
        if medias:
            rs = lambda m: m.get("resource_status") if m.get("resource_status") is not None else 0
            if any(rs(m) in (2, 3) or (m.get("status") == "failed") for m in medias):
                raise RuntimeError(f"Task failed: {task_id}")
            if all(rs(m) == 1 for m in medias):
                return task
        time.sleep(interval)
    raise TimeoutError(f"Task timed out: {task_id}")
```
