---
name: ootd-style-share
description: "Produces today's OOTD as a full-body character image plus a short showcase video. The skill ends after finalizing `ootd_result`."
allowed-tools: Bash(dl knowledge search:*) Bash(dl fetch:*) Bash(dl generate-image:*) Bash(dl generate-video:*) Bash(dl artifact:*)
metadata:
  ilands:
    applicable-to: [full]
    priority: 3.0
    kind: composition_skill
    recommended-skills:
      - image-generation
      - video-generation
    produces:
      - slot: "ootd_result"
        content_type: "application/json"
artifact-contract: schemas/artifact_contract.json
---

# OOTD Style Share

OOTD skill for a single daily outfit drop. It produces one character image and one short cinematic showcase video, then stops after `ootd_result` is finalized. Downstream skills such as `selfie-vlog`, `daily-vlog`, `daily-comic`, `host-broadcast`, and `trending-dance` may reuse the same character anchor later in the day.

## Artifact CLI Primer

Use the artifact working set through `dl artifact ...`.

- `dl artifact write --slot=<name> --content-type=<mime> --content-file=<path|->`
- `dl artifact read --slot=<name>`
- `dl artifact patch-json --slot=<name> --operations-file=<path|->`
- `dl artifact finalize --slot=<name> --mode=verify|verify_and_promote`

## Workflow

### 1. Find one or two outfit references

Use the current season, the agent persona, `SELF.md`, and `memory.md` to choose a reference that is not too similar to recent OOTDs. Search social or image sources and keep the best one or two references.

### 2. Synthesize the persona look

Turn the reference into a concrete image prompt and a short showcase video prompt. Preserve immutable persona traits and keep the outfit physically plausible.

### 3. Generate the full-body image

Generate one watermark-free full-body image at the chosen aspect ratio. If the first provider fails, switch once before degrading.

### 4. Generate the showcase video

Generate one 10-15 second showcase video from the image and prompt. If the video fails, keep the image and record the degradation instead of inventing a fake video URL.

### 5. Finalize `ootd_result`

Write the terminal artifact and finalize it.

```bash
cat <<'EOF' | dl artifact write --slot=ootd_result --content-type=application/json --content-file=-
{
    "reference_urls":["<reference_1>","<reference_2>"],
    "character_url":"<character_url>",
    "video_url":"<video_url>",
    "aspect_ratio":"<aspect_ratio>",
    "image_prompt":"<image_prompt>",
    "video_prompt":"<video_prompt>",
    "quality_tier":"<normal|degraded>",
    "created_at":"<iso8601>"
  }
EOF

dl artifact finalize --slot=ootd_result --mode=verify_and_promote
```

Completion checks:

- `ootd_result` is promoted
- `character_url` is non-empty
- `aspect_ratio` matches the chosen render plan

Recovery:

- if reference search fails, widen the search once and then continue from season + persona defaults
- if the image or video generation fails, keep the best partial output and mark the tier as degraded

## Constraints

- Do not publish from this skill
- Do not ask the user for confirmation
- Do not refresh the character anchor multiple times in the same day unless the agent intentionally starts a new outfit run
