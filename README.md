name: pc-disk-cleanup
description: PC Disk Cleanup — scans and cleans Windows C: and D: drives for junk, caches, temp files, and old versions. Two-phase safe workflow: scan first, clean after user review. Use this skill whenever the user asks about "电脑磁盘清理", "PC disk cleanup", "clean disk", "free up space", "cleanup Windows", "清理磁盘", "释放空间", "scan for junk", "disk full", "what's taking up space", "清理C盘", or wants to reclaim storage on Windows.
---

# Windows Disk Cleanup

Two-phase safe cleanup: scan first, then clean after user review.

## Workflow

The scripts are in this skill's `scripts/` directory. The default install path is `~/.claude/skills/pc-disk-cleanup/`. Before running, verify the path exists.

### Phase 1: Scan

Run `scan.ps1` to analyze disk usage safely (read-only):

```powershell
# Default install location
$scripts = "$env:USERPROFILE\.claude\skills\pc-disk-cleanup\scripts"

# Scan both drives (default)
powershell -ExecutionPolicy Bypass -File "$scripts\scan.ps1" -Drives C,D

# Scan single drive
powershell -ExecutionPolicy Bypass -File "$scripts\scan.ps1" -Drives C

# JSON output (status messages go to stderr, JSON to stdout)
powershell -ExecutionPolicy Bypass -File "$scripts\scan.ps1" -Drives D -OutputFormat JSON

# JSON to file (clean output without status messages mixed in)
powershell -ExecutionPolicy Bypass -File "$scripts\scan.ps1" -Drives D -OutputFormat JSON -OutFile scan_result.json
```

The scan script outputs a categorized report with:
- Item name and path
- Size (human-readable and bytes)
- Category: `cache`, `temp_files`, `old_versions`, `unused_programs`, `user_data`
- Risk level: `low` (safe to delete), `medium` (can reinstall if needed), `high` (user data, review carefully)
- Recommendation: `safe`, `review`, `manual` (requires admin privileges)

Present the scan results to the user as a table with checkboxes. Let them select what to delete. **Never delete anything without user confirmation.**

### Phase 2: Clean

After the user confirms their selections, write a JSON file with the selected items and run:

```powershell
powershell -ExecutionPolicy Bypass -File "$scripts\clean.ps1" -ItemsFile selected_items.json

# Dry run first to verify what will happen
powershell -ExecutionPolicy Bypass -File "$scripts\clean.ps1" -ItemsFile selected_items.json -DryRun
```

The clean script:
- Logs every action to `pc-disk-cleanup.log` in the current directory
- Reports progress with per-item status
- Handles "access denied" by noting the item requires admin elevation
- Never deletes anything not in the items file

## Safety rules

1. **Scan is read-only** — never modifies files, only measures sizes
2. **User must confirm** — always present the report and get explicit confirmation before deleting
3. **Precise path matching** — never use fuzzy glob matching on deletion; only exact paths from the scan report
4. **Log everything** — every deletion is recorded with timestamp, path, and result
5. **Flag admin items** — items needing administrator privileges are marked `manual` and skipped in normal clean
6. **Warn about high-risk items** — user data (Lightroom, screenshots, documents) gets extra warning before deletion

## Categories and risk levels

| Category | Risk | Examples |
|----------|------|----------|
| cache | low | GPU shader cache, browser cache, npm cache, Discord cache, Adobe CameraRaw cache |
| temp_files | low | Windows Temp, updater downloads, WPS addons cache, WeChat temp |
| old_versions | medium | Old WeChat versions, Chrome Canary, editor installs (can reinstall) |
| unused_programs | medium | Apps no longer used, game installers |
| user_data | high | Screenshots, Lightroom exports, chat backups — only delete if user confirms |

## Manual cleanup (admin required)

These items are detected but cannot be auto-cleaned:
- `pagefile.sys` — reduce via `sysdm.cpl` > Advanced > Performance > Virtual Memory
- Installed programs (Lenovo, VS 2022, etc.) — uninstall via Settings > Apps
- Windows.old — use Disk Cleanup > Clean up system files

When scanning detects these, include them in the report with `risk: manual` and explain the manual steps.

## Requirements

- **Windows only** — PowerShell 5.1+ (built into Windows 10/11)
- No admin required for scanning; some clean targets need admin elevation
- Skill installs to `~/.claude/skills/pc-disk-cleanup/` by default
- Scripts use standard Windows environment variables (`$env:LOCALAPPDATA`, `$env:APPDATA`, etc.)
