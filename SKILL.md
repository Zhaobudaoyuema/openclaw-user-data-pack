---
name: openclaw-user-data-pack
version: 1.1.0
description: "Pack OpenClaw user data into a zip with manifest; apply that zip to a new OpenClaw home/workspace. Optional layers gated (managed skills, sessions, config)."
trigger: "OpenClaw backup, export user data, pack workspace, migration zip, 打包 openclaw, 迁移记忆, openclaw 一键导出, 一键应用, 导入 zip, restore openclaw from zip, 新机器恢复 openclaw"
---

# OpenClaw user data: pack and apply

Audience: you are the OpenClaw agent. Follow this to help the user export or import their data using the scripts in this skill.

Respond in the same language the user uses.

## What you do

When the user wants to back up or migrate out: run `scripts/pack_openclaw.py` to build a local zip that includes `EXPORT_MANIFEST.txt`. Default pack includes only the workspace tree under `workspace/` in the archive. Add optional flags only after explicit user confirmation for each optional layer.

When the user wants to restore on another machine or a fresh `~/.openclaw`: run `scripts/apply_openclaw.py` with their zip. Default apply extracts only `workspace/` from the zip into the target workspace. Add optional apply flags only after explicit user confirmation for each optional layer.

## Safety (state before pack or apply)

The zip may contain highly sensitive material: persona files, MEMORY.md, memory logs, workspace skills; if optional layers were packed, also full session JSONL and openclaw.json (keys, tokens, channel settings).

Never pack `~/.openclaw/credentials/`. The apply script never writes to credentials. Tell the user they must re-login and re-pair channels on the new machine; do not copy credentials from the old host unless they fully accept the risk.

Warn against uploading the zip to untrusted or public storage.

Apply overwrites existing files with the same path. If `--apply-config` is used, the script backs up the existing `openclaw.json` to `openclaw.json.bak.<timestamp>` before replacing it.

## Pack: default vs optional

| Content | Path inside zip | In default pack? |
|---------|-------------------|------------------|
| Workspace (persona, memory, workspace skills, canvas, etc.) | `workspace/` | yes |
| Managed skills | `managed-skills/` | no, needs `--managed-skills` |
| Sessions | `sessions/<agentId>/sessions/` | no, needs session flag plus acknowledgement flag |
| Config snapshot | `config/openclaw.json` | no, needs config flags plus acknowledgement flag |
| Credentials | n/a | never |

## Apply: default vs optional

Match flags to what is actually in the zip. If a layer exists in the zip but the user did not pass the matching flags, the script prints warnings and skips that layer.

| Content | Action | Default apply? |
|---------|--------|----------------|
| Workspace | Extract `workspace/*` into the target workspace directory | yes, unless `--no-apply-workspace` |
| Managed skills | Extract into `<openclaw-home>/skills/` | no, needs `--apply-managed-skills` |
| Sessions | Extract into `<openclaw-home>/agents/<id>/sessions/` | no, needs `--apply-sessions` and `--i-know-restoring-sessions-overwrites` |
| Config | Write to `<openclaw-home>/openclaw.json` (backup first) | no, needs `--apply-config` and `--i-know-config-overwrites-secrets` |

## Paths

OpenClaw home: `$OPENCLAW_HOME` or `~/.openclaw`. On Windows: `%USERPROFILE%\.openclaw`.

Workspace resolution: pack script reads config when `--workspace` is omitted. For apply, if the user passes `--workspace`, the directory is created if missing. If omitted, resolve from the current environment (openclaw.json must parse). On a new machine, prefer `openclaw onboard` first or pass `--workspace` explicitly.

Run pack and apply in the same kind of environment (e.g. both in WSL) so paths refer to the same files.

If `openclaw.json` is JSON5 and parsing fails, run `pip install -r requirements.txt` (includes json5) or pass `--workspace` explicitly.

## Pack workflow

1. `pip install -r requirements.txt`
2. `python scripts/pack_openclaw.py --dry-run` and confirm the list
3. Explain default vs optional layers. Do not add `--managed-skills`, session flags, or config flags without separate user approval for each
4. `python scripts/pack_openclaw.py` with only approved flags
5. Tell the user the output zip path and that `EXPORT_MANIFEST.txt` is inside the archive

## Apply workflow

1. Confirm the zip is from a trusted source and was produced by `pack_openclaw.py` in this skill (expected layout: `workspace/`, `EXPORT_MANIFEST.txt`, etc.)
2. `pip install -r requirements.txt`
3. Run apply with `--dry-run` first, e.g. `python scripts/apply_openclaw.py --zip <path.zip> --dry-run` plus `--openclaw-home` and `--workspace` as needed
4. Explain what paths will be written. Do not add `--apply-managed-skills`, session flags, or config flags without separate user approval
5. Run without `--dry-run`. If restoring config, remind the user they still need valid model and channel auth on the new machine; paths inside an old openclaw.json may not match this machine
6. Optionally suggest the user run `openclaw doctor` locally after restore (they run it in their environment)

Example: workspace only, explicit home and workspace

```bash
python scripts/apply_openclaw.py --zip ./openclaw-user-export-xxx.zip \
  --openclaw-home ~/.openclaw \
  --workspace ~/.openclaw/workspace \
  --dry-run
python scripts/apply_openclaw.py --zip ./openclaw-user-export-xxx.zip \
  --openclaw-home ~/.openclaw \
  --workspace ~/.openclaw/workspace
```

Example: all optional layers after user approved each

```bash
python scripts/apply_openclaw.py --zip ./export.zip \
  --openclaw-home ~/.openclaw --workspace ~/.openclaw/workspace \
  --apply-managed-skills \
  --apply-sessions --i-know-restoring-sessions-overwrites \
  --apply-config --i-know-config-overwrites-secrets
```

## CLI reference

Pack:

```text
python scripts/pack_openclaw.py [--workspace PATH] [--openclaw-home PATH] [--config PATH]
  [-o FILE.zip] [--exclude-git | --no-exclude-git] [--managed-skills]
  [--sessions --i-know-sessions-are-large-and-sensitive]
  [--config-snapshot --i-know-config-may-contain-secrets]
  [--dry-run] [--manifest-sha256] [--sha256-max-mb N]
```

Apply:

```text
python scripts/apply_openclaw.py --zip FILE.zip [--openclaw-home PATH] [--workspace PATH] [--config PATH]
  [--no-apply-workspace] [--apply-managed-skills]
  [--apply-sessions --i-know-restoring-sessions-overwrites]
  [--apply-config --i-know-config-overwrites-secrets]
  [--dry-run]
```

## Trigger hints

| Intent | Example user phrases |
|--------|----------------------|
| Export | backup workspace, export memory, pack openclaw |
| Import | new PC restore, import zip, apply backup, restore openclaw |
| Chinese | 一键打包, 一键应用, 导入 zip, 迁移 |

## Quick commands

| Goal | Command |
|------|---------|
| Pack preview | `python scripts/pack_openclaw.py --dry-run` |
| Pack | `python scripts/pack_openclaw.py` |
| Apply preview | `python scripts/apply_openclaw.py --zip x.zip --dry-run` |
| Apply | `python scripts/apply_openclaw.py --zip x.zip` |
| Dependencies | `pip install -r requirements.txt` |
