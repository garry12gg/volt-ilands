---
name: daily-vlog
description: >-
  Produces a 5–20 scene daily vlog from today's character anchor and
  writes `daily_vlog_result`. Use when the Agent wants a static
  composition workflow for a day-in-the-life story; do not use for
  outfit reveals, selfie monologues, or panel comics.
allowed-tools: Bash(dl generate-video:*) Bash(dl artifact:*) Bash(dl skill:*)
artifact-contract: schemas/artifact_contract.json
metadata:
  ilands:
    applicable-to: [full]
    priority: 2.5
    kind: composition_skill
    recommended-skills:
      - ootd-style-share
      - video-generation
    produces:
      - slot: "daily_vlog_result"
        content_type: "application/json"
---

# Daily Vlog

把今天的角色参考图转成 5–20 个场景的 vlog。这个 skill 只关心叙事脚本、逐场景视频生成和结果归档；没有 publish 步骤，终点是 `daily_vlog_result` 被验证并 promote。

## Artifact CLI Primer

Use the artifact working set through `dl artifact ...`.

- `dl artifact write --slot=<name> --content-type=<mime> --content-file=<path|->`
- `dl artifact read --slot=<name>`
- `dl artifact patch-json --slot=<name> --operations-file=<path|->`
- `dl artifact finalize --slot=<name> --mode=verify|verify_and_promote`

## 工作流

```
ootd_result / ootd-style-share  →  5–20 scene narrative JSON
                              →  sequential scene video generation
                              →  write + finalize daily_vlog_result
```

## Phase 1 - 角色锚点

- 优先读 `ootd_result` 的 `character_url`
- 如果当天没有可用角色图，`load_skill('ootd-style-share')` 跑到生成角色图为止
- 角色图只用作形象锚点，不在这里引入发布或审批语义

## Phase 2 - 叙事计划

- 输出严格 JSON，场景数 5–20
- 统一 `aspect_ratio`，推荐 `9:16`
- 每个场景都要有 `time`、`title`、`description`、`video_prompt`
- `video_prompt` 里不要写漂浮字幕、UI、logo
- 允许在结果里声明 `core_emotion`、`narrative_tone`、`tone_rationale`、`outfit_anchor`

### 退化策略

1. JSON 解析失败或场景数不足 5，重试 1 次，并显式要求只输出 JSON
2. 仍失败时，降级为 5 个最小可用场景，并把 `quality_tier` 设为 `degraded`
3. 完成率偏低时，保留已完成场景并继续归档，不回到用户确认路径

## Phase 3 - 逐场景生视频

- 每场景顺序提交，不并发轮询
- 角色图始终用 Phase 1 的 `character_url`
- 单场景失败先重试 1 次；仍失败就跳过该场景并继续下一个

## Phase 4 - 结果写入

```bash
cat <<'EOF' | dl artifact write --slot=daily_vlog_result --content-type=application/json --content-file=-
{
    "character_url": "<Phase 1.character_url>",
    "core_emotion": "<Phase 2.core_emotion>",
    "narrative_tone": "<Phase 2.narrative_tone>",
    "outfit_anchor": "<Phase 2.outfit_anchor>",
    "aspect_ratio": "<Phase 2.aspect_ratio>",
    "scenes": [...],
    "failed_scenes": [...],
    "quality_tier": "ok | degraded",
    "created_at": "<ISO8601>"
  }
EOF

dl artifact finalize --slot=daily_vlog_result --mode=verify_and_promote
```

## Completion Rules

- `daily_vlog_result` 是唯一 terminal slot
- `failed_scenes` 可以非空，但 `scenes` 里必须保留已完成场景
- 只要结果槽写入并通过 finalize，这个 skill 就结束

## Fallback Ladder

1. 角色锚点缺失时，先补 `ootd_result`，再补 `ootd-style-share`
2. 叙事 JSON 出错时，缩回 5 场景最小方案
3. 单场景视频失败时，跳过该场景并继续
4. 预算不足时，保留已完成场景并以 `degraded` 结束
