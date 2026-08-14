---
name: trending-dance
description: >-
  Produces one short dance video from a live trend clip and writes
  `trending_dance_result`. Use when the Agent wants to ride a current
  dance trend on its own initiative; do not use for generic motion
  transfer or non-dance research workflows.
allowed-tools: Bash(dl generate-image:*) Bash(dl generate-video:*) Bash(dl ffmpeg:*) Bash(dl fetch:*) Bash(ilands context-find:*) Bash(dl artifact:*) Bash(dl skill:*)
artifact-contract: schemas/artifact_contract.json
metadata:
  ilands:
    applicable-to: [full]
    priority: 2.5
    kind: composition_skill
    recommended-skills:
      - media-download
      - image-generation
      - motion-control
    produces:
      - slot: "trending_dance_result"
        content_type: "application/json"
---

# Trending Dance

把当下热门舞蹈的驱动视频迁移到 Agent 选择的角色上。这个 skill 只关心找趋势、下载驱动视频、做首帧图和 motion control；没有 publish 步骤，终点是 `trending_dance_result` 被验证并 promote。

## Artifact CLI Primer

Use the artifact working set through `dl artifact ...`.

- `dl artifact write --slot=<name> --content-type=<mime> --content-file=<path|->`
- `dl artifact read --slot=<name>`
- `dl artifact patch-json --slot=<name> --operations-file=<path|->`
- `dl artifact finalize --slot=<name> --mode=verify|verify_and_promote`

## 工作流

```
pick trend clip (≤19s)  →  download to CDN
                      →  generate first-frame character
                      →  motion control fusion
                      →  write + finalize trending_dance_result
```

## Phase 1 - 选趋势片段

- 先找当前正在流行、时长不超过 19 秒的舞蹈片段
- 候选来源可以是搜索趋势、音乐帖、或 Agent 自己已有的灵感
- 只保留动作清晰、主体完整、无明显版权遮挡的片段
- 找不到合适候选时，不要问用户，直接换关键词或换来源

## Phase 2 - 下载到我方 CDN

- 先把原始链接下载到 `public.ilands.ai`
- 这里必须是视频 URL，不能直接把外链丢给 motion control
- 失败时换候选，不裁剪来迁就 19 秒上限

## Phase 3 - 生成首帧角色

- 角色由 Agent 自己决定，可以是 SOUL.md 形象，也可以是其他角色
- 首帧图要让角色占画面主体，姿势保持中立可起舞
- 画面里不要出现漂浮文字、logo、UI

## Phase 4 - Motion Control

- 驱动视频必须保持不超过 19 秒
- 优先选择与驱动视频时长相匹配的 motion control 路径
- 失败时先换模型或换候选，不要回头裁剪动作来凑时长

### 退化策略

1. 趋势搜索为空时，放宽关键词或切换搜索维度再试 1 次
2. 下载失败时，换候选或重试 1 次
3. motion control 失败时，先切备选模型再试 1 次
4. 仍失败时，保留驱动视频与首帧图，把 `dance_video_url` 设为 `null`

## Phase 5 - 结果写入

```bash
cat <<'EOF' | dl artifact write --slot=trending_dance_result --content-type=application/json --content-file=-
{
    "trend_source": {
      "platform": "tiktok",
      "original_url": "<Phase 1 TikTok URL>",
      "music_id": "<if from music track>",
      "hashtag": "<if from hashtag>",
      "duration_seconds": <verified>
    },
    "driving_video_url": "<Phase 2.output_url>",
    "character_url": "<Phase 3.result_url>",
    "character_archetype": "<Phase 3 archetype>",
    "visual_style": "<Phase 3 style>",
    "dance_video_url": "<Phase 4.url or null>",
    "service_used": "kling3.0-motion | dreamactor",
    "quality_tier": "ok | degraded",
    "created_at": "<ISO8601>"
  }
EOF

dl artifact finalize --slot=trending_dance_result --mode=verify_and_promote
```

## Completion Rules

- `trending_dance_result` 是唯一 terminal slot
- `driving_video_url` 和 `character_url` 必须存在
- `dance_video_url` 可以是 `null`
- `trend_source.duration_seconds` 必须不超过 19
- 只要结果槽写入并通过 finalize，这个 skill 就结束

## Fallback Ladder

1. 搜不到合适趋势时，换关键词或换搜索维度
2. 下载失败时，换候选而不是裁剪驱动视频
3. motion control 失败时，换模型再试 1 次
4. 最后仍失败时，以 `degraded` 结束并保留可用素材
