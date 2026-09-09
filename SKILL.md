---
name: photo-library-organizer
description: "Organize a flat camera uploads dump (multiple years of photos, videos, and screenshots mixed together) into a structured photo library with event/trip subfolders. Generates safe Python + bash scripts with dry-run mode and SHA-256-verified deletes. Use when: organize my photos, clean up camera uploads, sort my photo library, deduplicate photos."
---

# Photo & Video Library Organizer

Helps users organize a flat camera uploads dump — a single folder containing multiple years of photos, videos, and screenshots all mixed together — into a structured photo library with event/trip subfolders.

## Overview

This skill guides Claude through generating Python and bash scripts that:
1. Deduplicate camera uploads against an existing organized photo library (SHA-256)
2. Classify screenshots vs. real photos using EXIF metadata
3. Cluster remaining photos by date and match to existing event/trip folders
4. Route videos to a separate video library (with dedup)
5. Generate safe, verified copy + delete scripts with dry-run mode

**Safety model**: all destructive operations use two-pass SHA-256 verification (verify ALL destination copies → then delete ALL originals). Every script has `--dry-run` before `--copy`/`--move`/`--delete`. If the user has cloud sync or a backup solution (iCloud, Google Photos, Time Machine, etc.), that provides an additional recovery layer — but do not rely on it as the only safety net.

---

## Phase 0 — Discovery

Ask the user all at once as a numbered list:

1. Path to their **camera uploads dump** (flat folder, all years mixed — this is the source to clean out). Use an absolute path; if it's on an external or network drive, confirm it's mounted first.
2. Path to their **organized photo library** (has year/event subfolders — this is the destination). Ask them to describe the folder structure (e.g. "year folders directly contain event folders" vs. something deeper) rather than assuming — Phase 3 needs this to know how deep to look.
3. Path to their **video library** — optional; a separate folder for longer family/home videos. If left blank and camera uploads turns out to contain video files, Phase 4 will ask again at that point rather than inventing a destination.
4. What OS they're on: **macOS** or **Linux** (Windows not supported by bash scripts)
5. Whether they want **local AI classification** via Ollama (advanced, optional — improves event matching, requires ~1.8 GB RAM for the model)

Store these as named constants at the top of every generated script — assign each path to a shell variable once and reference it everywhere after; never re-type a literal path. State up front that all generated scripts, reports, logs, and checkpoints will live in `{organized_library}/_photo_org_scripts/`.

**Before generating any script:**
- Confirm `exiftool` and `python3` (3.8+) are installed — `exiftool -ver` and `python3 --version`. If either is missing, give the install command (`brew install exiftool` on macOS) and stop; don't generate scripts against a broken environment.
- Confirm the user has a working backup or cloud sync covering the camera uploads folder. This skill permanently deletes files — without a backup, a deletion cannot be undone regardless of the two-pass verification.

**Key OS differences to apply throughout:**
- macOS: `shasum -a 256 "$file" | awk '{print $1}'`, use `pip3`, no `--break-system-packages`
- Linux: `sha256sum "$file" | awk '{print $1}'`, use `pip --break-system-packages`

---

## Phase 1 — Deduplication Against Existing Library

Generate `find_dupes.py`:
- SHA-256 index every file in the organized library (recursive)
- SHA-256 index every file in camera uploads
- Match: camera uploads files whose SHA-256 exists anywhere in the organized library
- Output: `dupes_report.txt` (human-readable summary by destination folder) + `delete_dupes.sh`
- **Progress output**: print a running count every 1000 files indexed (e.g. `Indexed 1000 files...`) so a long run doesn't look hung

**`delete_dupes.sh` — mandatory two-pass structure:**
```
Pass 1: for every duplicate, verify the destination copy exists AND SHA-256 matches the source
        If ANY fail → print all failures, ABORT, exit 1 (delete nothing)
Pass 2: only runs if Pass 1 fully passed → delete all camera uploads originals
```

Never touch the organized library. Only delete from camera uploads.

---

## Phase 2 — Screenshot Classification

Generate `classify_files.py`:
- Run `exiftool -json` in batch mode on all remaining camera uploads images
- Real photo signal: any of these EXIF fields present → FNumber, ISO, ExposureTime, ShutterSpeedValue, ApertureValue, FocalLength, LensModel, LensInfo
- Screenshot signal (secondary): image dimensions match a known phone screenshot resolution
- **If exiftool can't read a file** (corrupt, unsupported format, permission error): don't crash the batch — treat it as having no EXIF data, so it falls through to the screenshot-signal check below, and log it separately to `exif_errors.txt` for the user to review

Known screenshot dimensions (check both portrait and landscape):
```
750×1334   iPhone 6/7/8
1125×2436  iPhone X/XS
1170×2532  iPhone 12/13
1179×2556  iPhone 14/15 Pro
1284×2778  iPhone 12/13 Pro Max
1290×2796  iPhone 14/15 Pro Max
828×1792   iPhone XR/11
1206×2622  iPhone 16 Pro
1320×2868  iPhone 16 Pro Max
1080×1920  Android common
1080×2340  Android common
2048×2732  iPad Pro 12.9"
1668×2388  iPad Pro 11"
```

This list isn't exhaustive and won't cover every future device — that's fine by design, because it's only ever a secondary signal. The classification priority below already falls back to "no camera EXIF" alone (lower confidence, still flagged for review) when a resolution doesn't match, so an unlisted phone doesn't get silently missed.

Classification priority:
1. Has camera EXIF → real photo (goes to `real_photos.txt`)
2. No camera EXIF + matches screen resolution → screenshot with high confidence
3. No camera EXIF + no screen resolution match → screenshot with lower confidence (flag separately)

Output: `real_photos.txt`, `likely_screenshots.txt` (with confidence reason per file), `move_screenshots.sh`

`move_screenshots.sh` moves files to `{camera_uploads}/_screenshots/` — NOT deleted. User reviews before any permanent action.

---

## Phase 3 — Date Clustering + Folder Assignment

Generate `cluster_photos.py`:

**Step 1 — Extract dates from real photos**
- Run `exiftool -json -r -DateTimeOriginal -CreateDate -FileName` on real photos list
- Date priority: DateTimeOriginal → CreateDate → parse `YYYY-MM-DD` prefix from filename
- Files with no recoverable date → `undated_photos.txt` for manual review

**Step 2 — Build date index from organized library**
- Walk the organized library recursively (however deep the user described in Phase 0) and run exiftool on every existing photo, treating each leaf folder that directly contains photos as a candidate destination
- Build: `date → [folder_name]` index, where `folder_name` is that leaf folder's path relative to the library root
- Skip folders whose name matches a catch-all pattern (e.g., contains "misc", "travels", "unsorted")
- **If this index comes back empty** (no photos with readable dates found anywhere in the library): stop and tell the user — don't silently fall through to catch-all-only behavior for the entire library. Ask them to confirm the library path is correct, or confirm it's genuinely a fresh/empty library (in which case every cluster will route to catch-all, which is expected).

**Step 3 — Group photos into date clusters**
- Sort photos by date
- Start a new cluster when the gap between one photo's date and the *next* photo's date (i.e., between consecutive dates once sorted) exceeds `DATE_GAP_DAYS` (default: 3)
- Example: dates Jan 1, Jan 2, Jan 4, Jan 9 → consecutive gaps are 1, 2, 5 days. The 2-day gap (Jan 2 → Jan 4) is ≤ 3, so they stay together; the 5-day gap (Jan 4 → Jan 9) exceeds 3, so it splits into two clusters: `[Jan 1, Jan 2, Jan 4]` and `[Jan 9]`.

**Step 4 — Score each cluster against existing folders**
- Primary: for each indexed folder, count how many of its indexed dates (from Step 2) also appear among the cluster's own photo dates — that's "shared dates" (+1 per shared date, not per photo)
- Example: a cluster spans Jan 1–4. Folder "2024 - New Year Trip" is indexed with photos dated Jan 2 and Jan 3. Shared dates = {Jan 2, Jan 3} → score = 2, regardless of how many photos fall on each date.
- Secondary (if moondream enabled): description token overlap (+2 per shared meaningful token)
- Assign to highest-scoring folder with score > 0

**Step 5 — Fallback assignment**
Only runs when Step 4 found zero folders with any date overlap (score 0 everywhere):
1. Check event keywords (see below) against moondream description (if available)
2. Check seasonal heuristics by cluster date
3. Route to year catch-all folder

**Catch-all folder**: look for a folder in the organized library matching `{YEAR}.*[Tt]ravel` or `{YEAR}.*[Mm]isc` or `{YEAR}.*[Uu]nsorted` — this is a substring/regex match, so `2024-Misc`, `2024 Misc`, and `2024_Misc` all match regardless of the separator. If none exists, suggest the user create `{YEAR} - Misc`.

**Event keywords (secular):**
```
birthday, wedding, graduation, anniversary, engagement,
halloween, thanksgiving, new year, fourth of july, independence day,
valentine, memorial day, labor day,
beach, pool, snow, hiking, ski, skiing, camping,
restaurant, park, concert, game, sports,
baby shower, baby, newborn, portrait, photoshoot,
reunion, party, barbecue, bbq, picnic, parade
```

**Matching rule**: this is a low-bar heuristic, not a scored threshold — it only runs in Step 5, after Step 4 already found zero date overlap anywhere. A single case-insensitive substring match, against either the candidate folder's name or the moondream description, is enough to route the cluster there. E.g. a folder named "My Birthday Hike" matches on the "birthday" keyword. If multiple candidate folders match, prefer the one in the same year; if still tied, fall through to seasonal heuristics.

**Seasonal heuristics (by cluster date):**
- Oct 15 – Nov 5 → Halloween
- Nov 20 – Dec 5 → Thanksgiving  
- Dec 26 – Jan 3 → New Year

Output: `clusters_report.txt` (grouped by destination, with per-cluster confidence) + `copy_clusters.sh` (`--dry-run` / `--copy`)

After `--copy` succeeds → user runs `gen_delete_clusters.py` which reads the copy log and generates `delete_clusters.sh` (two-pass verify + delete).

---

## Phase 4 — Video Pipeline (optional)

Trigger this phase if user provided a video library path OR if camera uploads contains video files (.mp4, .mov, .m4v, .avi, .mkv, .wmv, .3gp).

**If videos are found but Phase 0 didn't collect a video library path**, stop and ask the user now, before generating any script — don't invent a destination folder. They may want to point this phase at a new or existing folder, or skip video handling for this run and leave videos in camera uploads untouched.

Generate `check_video_dupes.py`:
- SHA-256 index all videos in video library (read-only — **never modify this folder**)
- For each camera uploads video: check SHA-256 against index
- Duplicates → `delete_video_dupes.sh` (delete camera uploads copy only)
- Unique videos → `copy_unique_videos.sh` (copy to `{video_library}/{YYYY}/` using year from `YYYY-MM-DD` filename prefix; prompt user for year if filename has no date)
- **Progress output**: print a running count every 10 videos indexed (e.g. `Indexed 10 videos...`) — videos are large enough that even small counts take a while to hash

After copy succeeds → generate `delete_unique_videos.sh` (two-pass verify: check video library copy exists + SHA-256 matches → then delete camera uploads originals).

---

## Phase 5 — Local AI Classification with Ollama (Advanced)

For users who opted in during Phase 0. Requires [Ollama](https://ollama.com) installed locally.

**Setup:**
```bash
# 1. Install the Ollama app from https://ollama.com (one-time, not a CLI step)
# 2. Pull the moondream model:
ollama pull moondream
# 3. Install the Python client the generated scripts use to talk to Ollama:
pip3 install ollama   # macOS
pip install ollama --break-system-packages   # Linux
```

Model: `moondream` (~1.8 GB download). While running, Ollama keeps roughly 1.8–2 GB of RAM allocated to the model *in addition to* whatever the OS and other apps are already using — this is in-use RAM, not total system RAM. On an 8 GB machine that's workable but tight: close memory-heavy apps (browser with many tabs, IDE) first. On 16 GB+ it's a non-issue. The model runs entirely locally — no photos leave the machine.

**Integration into `cluster_photos.py`:**
- After EXIF date extraction, loop through real photos and call:
  ```python
  import ollama, base64
  with open(path, "rb") as f:
      img_b64 = base64.b64encode(f.read()).decode()
  resp = ollama.chat(model="moondream", messages=[{
      "role": "user",
      "content": "Describe this photo in one sentence. Focus on people, setting, and activity.",
      "images": [img_b64]
  }])
  description = resp["message"]["content"]
  ```
- Wrap each `ollama.chat` call in a timeout (e.g. 30s) and a try/except — on timeout, connection error, or an unresponsive Ollama server, log the file to `moondream_errors.txt`, skip it, and continue. A single stuck call must never abort the whole run; a file that errored still goes through date clustering, it just lacks the AI description signal.
- Save to checkpoint JSON after each file — format `{"relative/path/to/file.jpg": "description text", ...}`, keyed by the file's path relative to camera uploads. On startup, load the checkpoint first and skip any file whose key is already present, so a killed/resumed run doesn't redo work or re-call Ollama for files it already described.
- Use description tokens as secondary scoring signal during folder assignment
- Include description excerpt in `clusters_report.txt` for review

When Ollama is **not** available: date clustering alone handles trip/event photos well. The main gap is photos with no date EXIF and no date in the filename — these go to catch-all folders without AI assistance.

---

## Script Conventions (apply to every generated script)

1. **Dry-run mode first** — `--dry-run` prints what would happen; `--copy`/`--move`/`--delete` executes
2. **Two-pass delete** — Pass 1 verifies ALL copies (abort on any failure), Pass 2 deletes
3. **Log file for every copy** — `{script}.log`, one line per file: `PREFIX|absolute_path` where PREFIX is `OK`, `SKIP`, `FAIL`, or `MISSING`. Only a real `--copy`/`--move` run writes this log — `--dry-run` prints a preview but writes nothing, so a delete-log-generator script run before any real copy has no log to read yet. Have it detect that and say so plainly, rather than failing with a raw file-not-found error. When parsing, split on the *first* `|` only (not every `|` in the line), in case a filename itself contains one.
4. **Delete scripts read copy logs** — only delete files logged as `OK|`
5. **Double-quoted bash strings** for all paths — escape only `\` and `"` inside double quotes (handles spaces and apostrophes; do NOT use `'\''` escaping inside double-quoted strings). `$` variables and `$(...)` command substitution ARE expanded inside double quotes — that's the point of using them over single quotes, and is intentional, not something to escape.
6. **Store user-provided paths as variables, once** — at the top of every script, assign each Phase 0 path to a shell variable (e.g. `CAMERA_UPLOADS="/Volumes/MyDrive/Photos"`) and reference it everywhere after as `"$CAMERA_UPLOADS/..."`. Never re-type or re-hardcode the literal path elsewhere in the script — this is what makes convention 7 (`$HOME` substitution) actually achievable.
7. **`$HOME` substitution** — never hardcode `/Users/username/` or `/home/username/`
8. **Skip `_`-prefixed subdirs** when walking source — avoids reprocessing `_screenshots/`, `_scripts/`, etc.
9. **Missing source in Pass 1** — `continue` (don't treat as mismatch — file may already be deleted)
10. **macOS vs Linux SHA-256**:
    - macOS: `hashfn() { shasum -a 256 "$1" 2>/dev/null | awk '{print $1}'; }`
    - Linux: `hashfn() { sha256sum "$1" 2>/dev/null | awk '{print $1}'; }`
11. **Bash arithmetic counters — use `if/then/else`, not `&&/||` chains:**
    ```bash
    # WRONG — ((DELETED++)) post-increments: when DELETED=0, evaluates to 0 (falsy), triggers || branch
    rm "$src" && echo "DELETED: $fname" && ((DELETED++)) || { echo "ERROR: $fname"; ((ERR++)); }
    
    # CORRECT — use if/then/else so rm's exit code is the only condition
    if rm "$src"; then
      echo "  DELETED: $fname"; ((++DELETED))
    else
      echo "  ERROR: $fname"; ((++ERR))
    fi
    ```
12. **Delete script footer** — end every delete script with a neutral recovery note:
    ```bash
    echo "If your backup solution (cloud sync, Time Machine, external drive, etc.) is active, deleted files may be recoverable there."
    ```
    Do NOT hardcode Dropbox or any specific cloud provider.
13. **Large libraries (100K+ files)** — warn the user up front that a full pass can take well beyond 10 minutes, and suggest running long scripts as `nohup python3 script.py > script.out 2>&1 &` (or inside `screen`/`tmux`) so the process survives a closed terminal or dropped SSH session.
14. **No concurrent runs against the same folders** — don't suggest running two scripts that touch the same source or destination at once (e.g. two `find_dupes.py` passes, or dedup and clustering simultaneously). There's no file locking, and concurrent writes to the same log file can interleave and corrupt it.

---

## Scripts Folder

Create `{organized_library}/_photo_org_scripts/` for all generated Python scripts, bash scripts, reports, logs, and checkpoints. Keeps the library root clean.

---

## Suggested Run Order (present as a checklist)

```
□ Phase 1 — Dedup
    python3 find_dupes.py
    Review: dupes_report.txt
    bash delete_dupes.sh --dry-run
    bash delete_dupes.sh --delete

□ Phase 2 — Screenshot Classification
    python3 classify_files.py
    Review: likely_screenshots.txt (spot-check before moving)
    bash move_screenshots.sh --dry-run
    bash move_screenshots.sh --move

□ Phase 3 — Photo Clustering
    python3 cluster_photos.py        (add --ollama flag if using Ollama)
    Review: clusters_report.txt
    bash copy_clusters.sh --dry-run
    bash copy_clusters.sh --copy
    python3 gen_delete_clusters.py   (reads copy log)
    bash delete_clusters.sh

□ Phase 4 — Videos (if applicable)
    python3 check_video_dupes.py
    bash delete_video_dupes.sh       (camera uploads dupes)
    bash copy_unique_videos.sh --dry-run
    bash copy_unique_videos.sh --copy
    python3 gen_delete_unique_videos.py
    bash delete_unique_videos.sh
```

---

## Conversation Flow

1. Ask all 5 discovery questions (Phase 0) in one message
2. Generate Phase 1 scripts immediately after receiving answers — dedup is always the right first step
3. Wait for user to report results before generating the next phase's scripts
4. If user reports errors: diagnose and fix before proceeding to the next phase
5. After the final phase: summarize what's left in the camera uploads folder (undated photos, low-confidence screenshots, anything that errored) and explicitly ask what they want to do with the camera uploads folder itself now that it's been cleaned out — leave it as-is, archive/rename it, or delete it once they're confident everything of value was moved. Don't assume; let the user decide.

Always confirm each destructive step completed successfully before moving to the next phase.
