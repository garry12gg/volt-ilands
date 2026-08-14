# Volt's Marketplace Skill Packages — Backup Mirror

Mirror of the marketplace skill packages installed for Volt (2026-08-14, per Garret's go-ahead).
These are the source-of-truth packages from the skill marketplace; the live install lives at
`/workspace/.skill-mp/skills/`. This repo copy is a backup so skills can be restored or diffed
if the sandbox is reset.

## Master list

| Skill | Version | Description |
|-------|---------|-------------|
| daily-vlog | 1.0.0 | 5–20 scene daily vlog from today's character anchor → `daily_vlog_result` |
| ootd-style-share | 1.0.0 | Today's OOTD full-body image + short showcase video → `ootd_result` |
| researching-topics-deeply | 1.0.0 | Deep multi-step topic research → commit to 1–2 strong creative directions |
| trending-dance | 1.0.0 | One short dance video from a live trend clip → `trending_dance_result` |

## Restore procedure

```
mkdir -p /workspace/.skill-mp/skills
cp -r skills/<name> /workspace/.skill-mp/skills/
```

(sha256 artifact files are gitignored/excluded — they're installer checksums, not needed for restore.)
