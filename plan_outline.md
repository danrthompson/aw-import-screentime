# Project Plan: Robust, Incremental Screen Time Ingestion for ActivityWatch (Enhanced Fork)

---

## Overview

We will fork and enhance the [ActivityWatch/aw-import-screentime](https://github.com/ActivityWatch/aw-import-screentime) repository to enable reliable, incremental ingestion of Screen Time data from iOS devices into ActivityWatch. The goal is to avoid duplicate imports, allow selective device filtering, support test/trial runs, enable periodic automated runs, and address other usability and robustness issues.

This enhanced version is designed for robust, repeated use—especially in automated or scheduled workflows.

---

## Features & Improvements

### 1. Incremental Import with Deduplication

**Goal:** Prevent duplicate events when the script is run multiple times.

- **Approach:**
  - Store the timestamp of the latest ingested event (in a file, e.g., `.last_imported_timestamp`).
  - Maintain a log of unique event IDs (using the database primary key `ZOBJECT.Z_PK` or a hash of event fields) that have already been imported (e.g., in `.ingested_ids.json` or a small SQLite DB).
  - On each run, pull all events starting from 7 days before the last imported timestamp.
  - Before ingesting, check if each event ID has already been imported; only ingest new ones.
  - After import, update the timestamp and the ingested ID log. Optionally, prune the log to avoid unbounded growth.

### 2. Device Filtering

**Goal:** Only ingest Screen Time data from specific devices (e.g., iPhones, skip Macs).

- **Approach:**
  - Add a configuration option (e.g., in a config file or as CLI argument) to specify allowed device IDs or device names.
  - When extracting events, filter out events not belonging to the specified devices.
  - Device IDs/names can be discovered via the script (e.g., a `--list-devices` mode).

### 3. Test/Trial Mode: Limit Ingested Date Range

**Goal:** Allow quick testing by ingesting only a small window of data.

- **Approach:**
  - Add a CLI flag (e.g., `--days=N`) to only process the last N days of data, regardless of incremental tracking.
  - Useful for debugging, testing new deduplication logic, or dry runs.

### 4. Automated Scheduling Support

**Goal:** Enable reliable, periodic ingestion (e.g., via cron).

- **Approach:**
  - Provide an example shell script or instructions for setting up a cron job (or systemd timer/Windows Task Scheduler).
  - Ensure the main script is idempotent and robust to repeated runs.

### 5. Additional Logging and Error Handling

- Improve logging so the user can see what’s ingested, skipped, or filtered.
- Handle common errors gracefully (e.g., missing DB, permission errors, malformed data).

### 6. Timezone Handling

- Ensure all times are normalized to UTC, or allow a config/CLI parameter for preferred timezone.
- Optionally store/display device local time as additional metadata.

---

## Steps To Accomplish

1. **Fork the repository and clone it locally.**
2. **Implement incremental import and deduplication**:
    - Store last timestamp in `.last_imported_timestamp`.
    - Store ingested IDs in `.ingested_ids.json` or a small SQLite DB.
    - Modify SQL query and import logic.
3. **Add device filtering**:
    - Accept a list of allowed device IDs/names via config or CLI.
    - Provide a way to list all discovered devices.
4. **Add trial mode/date range limiting**:
    - Allow user to specify how many days of data to process (default: all).
5. **Provide a scheduling script/instructions**:
    - E.g., a `run_periodic.sh` script and a crontab example.
6. **Improve documentation**:
    - README: Use, configure, interpret logs, find device IDs, set up periodic runs.
    - Document all new CLI flags/settings.
7. **Test the script** with different scenarios (first run, repeated run, missing DB, new device, etc.).
8. **Optional: Add automated tests** for the new behaviors.

---

## Confirmed Tasks

:white_check_mark: **To be implemented as part of the initial upgrade:**

1. **Incremental/Deduplicated Import** (see above)
2. **Device Filtering** (see above)
3. **Timezone Handling** (see above)
4. **Error Reporting & Resilience**
   - Improve error messages for missing/corrupt DB, permissions, etc.
   - Handle exceptions gracefully and log errors appropriately.

---

## Maybe/To-Do (Optional Enhancements)

:large_orange_diamond: **Enhancements under consideration:**

- **App Name/ID Normalization**
  - Normalize app names or IDs (e.g., lowercase, strip whitespace, map bundle IDs to friendly names) for better aggregation.
  - May require a mapping table.

- **Dry Run/Simulation Mode**
  - Add a CLI flag to simulate the import process, printing a summary of what would be ingested without writing to ActivityWatch.

- **Config File Support**
  - Allow configuration via YAML/JSON file as an alternative or supplement to CLI arguments.

- **Verbose/Quiet Logging**
  - Add CLI flags to adjust logging verbosity, supporting both detailed logs and minimal/quiet operation.

- **Additional Metadata Extraction**
  - Extract and store extra fields if available (e.g., app category, bundle ID, app version).

---

## Skipped/Lower Priority for Now

- **WAL Handling**
  - Handling SQLite's Write-Ahead Log (WAL) for the freshest data is skipped unless it proves easy and cross-platform.
  - *Workaround:* Manually checkpoint the DB with `sqlite3` if needed.

- **Automated Testing**
  - Unit/integration tests may be added later as the project matures.

---

## Example Usage

- **List available devices:**
  `python3 main.py --list-devices`

- **Ingest only iPhone data, last 2 days:**
  `python3 main.py --device-id ABCDE12345 --days 2`

- **Run periodically (e.g., via cron):**
  `0 * * * * /path/to/venv/bin/python3 /path/to/main.py --device-id ABCDE12345`

---

## Reference: Key Limitations of Screen Time Data

- Data may not always distinguish between "active" vs. "running" apps (esp. on macOS).
- The database may not store very long history (observed: 2-3 months).
- Most recent data may lag due to Apple’s WAL (write-ahead log) mechanism.
- Events may occasionally appear late (e.g., after a sync).
- Device IDs can be cryptic; mapping them to friendly names may require manual effort.

---

## Context & Rationale

- **Why not just run the original script periodically?**
  Because it re-imports all data every time, causing duplicates.
- **Why filter by device?**
  To avoid double-counting usage from Macs, which may be captured elsewhere.
- **Why deduplicate by ID and use a look-back window?**
  To catch late-arriving events and guarantee no duplicates, even if timestamps overlap.

---

## Implementation Suggestions

- **Database Path and Permissions**
  - The script must have read access to the iOS/macOS `Knowledge.db` file and any associated WAL files.
  - If running on macOS, you may need to grant Full Disk Access to Terminal or your Python environment.
  - Add a CLI/config option for custom database path, and document the default expected paths for macOS/iOS backups.

- **How to Obtain Knowledge.db from iOS Devices**
  - For iPhones/iPads, you typically need to make a local encrypted backup using Finder (macOS) or iTunes (Windows), then extract the `KnowledgeC.db` file from the backup directory.
  - Reference:
    - `~/Library/Application Support/MobileSync/Backup/` on macOS.
    - Consider providing a step-by-step or a reference for extracting the file.

- **ActivityWatch Bucket/Module Integration**
  - Specify which bucket or module in ActivityWatch the imported data will appear in (e.g., `aw-watcher-screentime`).
  - If you want to keep iOS data separate from macOS, allow per-device, per-bucket mapping (future enhancement).
  - Ensure event format matches ActivityWatch's expectations for proper analysis.

- **Edge Case Handling**
  - Some events may appear in the database with timestamps far in the past or future due to system clock issues or sync delays. Consider filtering or flagging these.
  - If you travel or change device timezones, double-check event times for consistency.

- **Performance Optimization**
  - If the database is large, use batched queries (e.g., `LIMIT` and `OFFSET`) to avoid loading everything into memory at once.
  - Log progress for long-running imports.

- **Security & Privacy**
  - Remind users that app usage data is sensitive; store logs and export files securely.
  - Consider adding a retention policy or a cleanup script for old logs and backup files.

- **CLI/UX Documentation**
  - Include example commands for every major feature and flag.
  - Document expected output (e.g., what does a successful run look like?).
  - Add troubleshooting tips for common errors (e.g., "database locked", "no devices found").

- **Versioning & Migration**
  - If you change the format of the deduplication log or timestamp file, document how to migrate or flush old data.

- **Contribution Guidelines (Optional)**
  - If you plan to open-source your fork, add a section with coding style, PR process, and issue reporting advice.

- **Contact / Support**
  - Add your contact info or preferred support channel if users/contributors have questions.

---

## Quick Start

1. **Install dependencies:**
   `poetry install`

2. **Extract `KnowledgeC.db` from your iPhone backup**
   (see [How to Obtain Knowledge.db from iOS Devices](#how-to-obtain-knowledgedb-from-ios-devices))

3. **List devices:**
   `python3 main.py --list-devices`

4. **Run import for your iPhone (last 7 days):**
   `python3 main.py --device-id <YOUR_DEVICE_ID> --days 7`

5. **(Optional) Schedule periodic runs:**
   See [Automated Scheduling Support](#4-automated-scheduling-support)

---

## Resources

- Original repo: https://github.com/ActivityWatch/aw-import-screentime
- Reference implementation for InfluxDB: https://github.com/FelixKohlhas/ScreenFlux
- Example for parsing Knowledge.db: https://www.r-bloggers.com/2019/10/spelunking-macos-screentime-app-usage-with-r/
- ActivityWatch: https://activitywatch.net/

---

## Next Steps

1. Fork and clone repository.
2. Implement and test the above features.
3. Document the process so it’s repeatable and robust.
4. Use the maybe/to-do section as a backlog for potential future improvements.
5. See the project plan for context, rationale, and references.

---

*This document serves as the comprehensive design, action plan, and technical reference for upgrading the ActivityWatch/aw-import-screentime tool to support robust, deduplicated, and configurable ingestion of iOS Screen Time data.*