# Legacy Plugin Workflow

This document primarily describes the retired plugin-mirror architecture that existed before the plugin tree retirement in change `207956`.

## Current State

Current Celest metadata is no longer centered on `tools/celest/plugins` or workspace `.plugins` mirrors.

The live model in `main` is:

- canonical metadata under `tools/celest/agents`, `skills`, `profiles`, `bin`, and `lib`
- workspace-local overrides under `.celest`
- runtime-managed Perforce submit workspace under `${OPENCODE_WORKSPACE_PERFORCE_ROOT:-/perforce}`
- runtime-injected P4 environment such as `P4PORT`, `P4USER`, and `OPENCODE_WORKSPACE_ID`

At workspace startup, `workspace-init.ts` syncs the canonical metadata roots into `.celest` and prepares the submit client `celest-submit-${OPENCODE_WORKSPACE_ID}`. The old `p4-workflow.ts` helper and `.plugins` mirror are no longer the primary workflow.

Use this document only as historical reference when investigating old behavior, migrations, or legacy docs that still mention `/plugins` or `.plugins`.

Practical reference for:

- understanding the legacy sync pipeline end-to-end
- understanding when and how the old plugin refresh path worked
- debugging old `.plugins` or `/plugins` references during migration

## Terms

- **Shared plugin root**: `tools/celest/plugins`, mounted read-only as `/plugins` in workspace containers
- **Local plugin mirror**: `<workspace>/.plugins`, a per-workspace Perforce-backed copy for local edits
- **Plugin-sync sidecar**: Docker container (`celest-plugin-sync`) that polls Perforce and syncs `/plugins`
- **State file**: `/plugins/.plugin-sync-state` — JSON with the latest synced changelist and timestamp
- **Mirror state file**: `.plugins/.plugin-management-sync-state.json` — per-workspace sync metadata (version, baseCl, per-file hashes)
- **Instance**: an in-memory context inside a workspace container, created on first HTTP request, cached until disposed or container restart
- **Sync client**: per-workspace read-only Perforce client for the local `.plugins` mirror
- **Submit client**: per-workspace Perforce stream client used by `/plugin-management:submit`

## Architecture Overview

Three-layer pipeline:

```
Layer 1: Plugin-sync sidecar (docker/plugin-sync.ts)
  Polls Perforce every 15s → p4 sync /plugins → writes .plugin-sync-state

Layer 2: Workspace mirror sync (shared-mirror-sync.ts)
  Triggered by freshness checks → p4 sync .plugins → protects local edits → cleans empty dirs

Layer 3: Command reload (command/index.ts, refresh.ts)
  Triggered after mirror sync → clears caches → re-scans plugins → registers commands/skills
```

### Layer 1: Plugin-sync Sidecar

The `celest-plugin-sync` container runs continuously and:

1. Authenticates with Perforce (`ensureLogin`, `ensureClient`)
2. Polls `p4 changes` every 15s (configurable via `PLUGIN_SYNC_INTERVAL_SECONDS`) for new changelists
3. Runs `p4 sync -n` (preview) to detect pending changes, even when CLs match the marker
4. Runs `p4 sync` when changes are detected or content is missing
5. Falls back to `p4 sync -f` (forced) when normal sync does not materialize content (e.g., stale have table)
6. Validates no pending changes remain after sync; throws if the tree is still inconsistent
7. Prunes top-level directories for plugins fully deleted from depot
8. Removes empty subdirectories left behind by file deletions
9. Chmod 0755 all files under `bin/` trees for shell executability
10. Writes `.plugin-sync-state` with the synced changelist (atomic write via temp + rename)
11. Cleans up legacy marker files (`.plugin-sync-ready`, `.plugin-sync-last_cl`)
12. Writes per-plugin metadata to `.claude-plugin/.plugin-sync-meta.json`

State file format:
```json
{"changelist":"205353","timestamp":"2026-03-25T16:34:43.600Z"}
```

Environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `P4PORT`, `P4USER`, `P4HOST`, `P4PASSWD` | — | Perforce credentials |
| `CELEST_P4_PLUGIN_DEPOT_ROOT` | `//entropia/main/main/tools/celest/plugins` | Depot path to sync |
| `PLUGIN_SYNC_INTERVAL_SECONDS` | `15` | Polling interval |
| `PLUGIN_ROOT` | `/plugins` | Local sync target directory |
| `PLUGIN_STATE_FILE` | `/plugins/.plugin-sync-state` | State file path |
| `PLUGIN_SYNC_CLIENT` | `celest-plugin-sync` | P4 client name |

### Layer 2: Workspace Mirror Sync

`shared-mirror-sync.ts` runs inside workspace containers when triggered. It:

1. Acquires a file lock (`.sync.lock`) to prevent concurrent syncs
2. Reverts any stale opened files in the sync client (leftover from old shared-client era)
3. Detects locally modified files by comparing content hashes against stored baselines
4. Runs `p4 sync`, protecting local modifications via backup-sync-restore (not `p4 edit`)
5. Detects stale have table entries and force-syncs if needed
6. Prunes deleted plugin directories (three-layer detection)
7. Removes empty subdirectories
8. Updates mirror state file with version, baseCl, and per-file hashes
9. Releases the lock

The sync client is **read-only** — it never runs `p4 edit`, `p4 add`, or `p4 delete`. Local modifications are protected by backing up the file content before sync, then restoring it after. This prevents the sync client from accumulating opened file state that could block future syncs.

The mirror state file has a version number (`STATE_VERSION`). When sync logic changes, the version is bumped. On the next sync, `loadState()` detects the version mismatch, discards the old state, and forces a full re-sync.

### Layer 3: Command Reload

After mirror sync completes, `PluginRefresh.run()` orchestrates:

1. `Command.reload()` → `refreshNow()` flushes caches in parallel:
   - `Config.refresh()` — re-scans config from all plugin directories
   - `Skill.refresh()` — re-discovers `SKILL.md` files
   - `ToolRegistry.state.clear()` — clears MCP tool cache
   - `Command.state.clear()` — clears command map
2. `Agent.refresh()` runs after the parallel flush (depends on config)
3. Plugins are re-scanned from both `/plugins` and `.plugins`
4. Commands and skills are re-registered
5. `Command.Event.Updated` is published (triggers frontend command list update)
6. `PluginRefresh.Event.Completed` (`plugin.refresh.completed`) is published
7. Frontend receives both events and updates the file tree and command list

**Dependency install guard**: All directories within `/plugins` or `.plugins` unconditionally skip dependency installation (`isPluginMirrorDirectory()` in `config.ts`). This prevents the config loader from running `bun install` for every mirrored plugin directory, which previously caused I/O storms during refresh fan-out. Read-only directories (`/plugins:ro`) also skip installation as a fallback check.

## Perforce Client Model

Each workspace has two isolated P4 clients, named using `OPENCODE_WORKSPACE_ID` (e.g., `wks_yinhua`). This prevents cross-user interference — one user's P4 state never blocks another user's operations.

### Sync client (read-only)

- Name: `celest-plugins-sync-{hash(projectRoot + workspaceId)}`
- Root: `<workspace>/.plugins`
- View: `//entropia/.../plugins/... → //<sync-client>/...`
- Purpose: `p4 sync` only. Never `p4 edit`, `p4 add`, or `p4 delete`.
- Used by: `shared-mirror-sync.ts` (background refresh) and `/plugin-management:sync`

### Submit client (edit + submit)

- Name: `celest-submit-{workspaceId}`
- Root: `/perforce` (Docker) or `D:/Perforce/main` (local)
- Model: Perforce Stream (`//entropia/main/main`)
- Purpose: all user-initiated P4 write operations (`p4 edit`, `p4 add`, `p4 reconcile`, `p4 submit`)
- Used by: `/plugin-management:submit`

### Why two clients

- The sync client uses a View mapping (narrow, plugin-files only).
- The submit client uses a Perforce Stream (depot-wide, needed for `p4 reconcile` + `p4 submit`).
- These are fundamentally different workspace models and cannot be merged.
- Both are per-workspace so one user's state never blocks another user's operations.

### Previous shared client issue

Before the per-workspace fix, all workspaces shared a single sync client (`celest-plugins-sync-{hash}`). When user A ran `/plugin-management:clone`, it opened files for `p4 add` in the shared client. User B's background sync then failed because `p4 sync` refused to replace files opened for add. After user A submitted via the separate submit client, the sync client's opened state was never cleared — blocking sync permanently.

## Loading Priority

Plugin discovery is handled by `resolveClaudePluginDirectories()` in `shell/path-env.ts`. The default order is **local-first**: local plugin directories are scanned before shared ones.

### Plugin root candidates

**Default (shared) roots**, checked in order:

1. `OPENCODE_CLAUDE_PLUGIN_ROOT` env var (set to `/plugins` in Docker)
2. `<configDir>/../../plugins` (relative to OpenCode config)
3. `<cwd>/plugins`
4. `<cwd>/tools/celest/plugins`
5. `/plugins` (container root constant)

**Local (workspace) roots**, checked in order:

1. `OPENCODE_WORKSPACE_PLUGIN_ROOT` env var
2. `<cwd>/.plugins`
3. `<cwd>/.plugins/plugins`
4. `<workspaceRoot>/.plugins`
5. `<workspaceRoot>/.plugins/plugins`

### Plugin detection

A directory is recognized as a plugin if it has either:
- `.claude-plugin/plugin.json` manifest, **or**
- Any of: `skills/`, `agents/`, `commands/`, `hooks/` subdirectories

### Deduplication

When the same plugin name (directory basename) exists in both local and shared roots:

- **local-first order** (default): `.plugins/<plugin>` wins, `/plugins/<plugin>` is ignored
- **default-first order**: `/plugins/<plugin>` wins

This means local development can override the shared version without changing the shared root.

### Bin path resolution

Each discovered plugin's `bin/` directory is added to the workspace `PATH`. The sidecar runs `chmod 0755` on all files in `bin/` trees after sync to ensure executability.

## Refresh Triggers

### What triggers a plugin refresh

| User Action | Frontend Trigger | Backend Path | How it Works |
|---|---|---|---|
| Open workspace | `global-sync.tsx` bootstrap `onMount` | `POST /global/plugin/check-freshness` | Compares shared CL vs local CL, syncs if stale |
| Sign in | `auth.tsx` after OAuth exchange | `POST /global/plugin/check-freshness` | Same CL comparison |
| Start new session | `submit.ts` after session create | `POST /global/plugin/check-freshness` | Same CL comparison |
| Enter directory | `global-sync.tsx` directory bootstrap | `POST /global/plugin/check-freshness` | Same CL comparison |
| Refresh page | `global-sync.tsx` `onMount` re-fires | `POST /global/plugin/check-freshness` | Same CL comparison |
| Plugin-sync updates (active) | File watcher detects `.plugin-sync-state` change | `triggerSharedRefresh(observedCL?)` → `runSharedRefreshCheck()` → `PluginRefresh.run()` | Watcher fires with optional CL from event; shared check compares against lastCompletedSharedCL |
| Container restart | Full Instance bootstrap | `ensureSharedPluginsFresh()` on first request | Fresh Instance, full scan |

### The freshness check chain

```
Frontend: ensureSharedPluginsFresh()
  → POST /global/plugin/check-freshness { directory }
    → Backend: PluginRefresh.ensureSharedPluginsFresh()
      → readSharedPluginChangelist()  → reads /plugins/.plugin-sync-state
        (falls back to legacy /plugins/.plugin-sync-last_cl)
      → readLocalMirrorBaseChangelist() → reads .plugins/.plugin-management-sync-state.json
      → if localCL < sharedCL → run mirror sync + Command.reload()
      → if localCL >= sharedCL → skip (already current)
```

### File watcher refresh chain

When plugin-sync writes a new `.plugin-sync-state`:

```
File watcher detects /plugins/.plugin-sync-state change
  → isSharedPluginMarkerFile() → true
  → triggerSharedRefresh(observedCL?)
    → updates lastObservedSharedCL
    → runSharedRefreshCheck()
      → while(true):
        → readSharedPluginChangelist() → file CL
        → effectiveCL = max(lastObservedSharedCL, fileCL)
        → if lastCompletedSharedCL >= effectiveCL → skip (done)
        → if inFlightSharedCL === effectiveCL → skip (already running)
        → PluginRefresh.run() → mirror sync + Command.reload()
        → lastCompletedSharedCL = effectiveCL
        → if lastObservedSharedCL advanced during refresh → loop again
        → otherwise → break
```

The retry loop catches the case where the sidecar pushes a new CL while the previous refresh is still running. Without it, the second CL would be missed until the next frontend trigger.

The `sharedCheckInFlight` field prevents duplicate concurrent refreshes. It is stored as the actual promise reference so the cleanup comparison works correctly (`current.sharedCheckInFlight === check`).

`PluginRefresh.run()` also deduplicates via an `inFlight` map keyed by `source:root:changelist`. Concurrent callers with the same key join the existing promise instead of launching a second refresh.

### File watcher filtering

The file watcher applies `FileIgnore.PATTERNS` (including `node_modules`, `dist`, `.cache`, etc.) to plugin root watchers. This prevents install-generated files from flooding the event bus.

`shouldRefreshForFile()` in `command/index.ts` determines whether a file change triggers a command refresh. Files that **trigger** refresh:

- `/plugins/.plugin-sync-state` (and legacy `.plugin-sync-ready`, `.plugin-sync-last_cl`)
- `.claude-plugin/plugin.json` — plugin manifest changes
- `opencode.json` / `opencode.jsonc` — config changes
- `*.md` files in `command/` or `commands/` directories
- `skill.md` files in `skill/` or `skills/` directories (including `.claude/skills/`, `.agents/skills/`)
- Any file under `.plugins/` (unless it matches the generated-path filter below)

Files under `.plugins/` that are **filtered out** as generated paths:

- `node_modules/**`
- `package.json`, `bun.lock`, `.gitignore`
- `dist/**`, `coverage/**`, `.cache/**`

This breaks the feedback loop where dependency install → file watcher → reload → more installs.

## Instance Lifecycle

An Instance is an in-memory cache for one project directory inside a workspace container.

| Event | Instance State | Refresh Behavior |
|---|---|---|
| First request to workspace | Created + bootstrap | Full plugin scan |
| Subsequent requests | Cached, reused | No re-scan (uses cached state) |
| User idle, comes back | Cached | Next frontend trigger checks freshness |
| Browser tab closed | Stays cached on server | No change |
| Browser reopened | Reused if alive | Frontend `onMount` triggers freshness check |
| Container restart | Destroyed | Fresh bootstrap on next request |
| Config update | Disposed + recreated | Full re-bootstrap |

Key: the Instance cache persists across browser tab closures. Refresh happens when the frontend triggers a freshness check on reconnect (via `onMount` in `global-sync.tsx`).

## Deletion Handling

### Whole plugin deletion

When all files in a plugin are deleted from Perforce:

**In `/plugins` (sidecar):**
1. `p4 sync` deletes tracked files
2. `pluginsWithDeletedFilesFromSyncOutput()` detects "deleted as" lines
3. `pluginStillTracked()` confirms no active files remain in depot
4. `pruneDeletedPluginDirectories()` removes the top-level directory

**In `.plugins` (workspace mirror):**
1. `p4 sync` deletes tracked files
2. Three-layer detection: sync output + state comparison + filesystem scan
3. `pruneDeletedPluginDirectories()` removes the directory

**Known limitation:** If `.claude-plugin/.plugin-sync-meta.json` (untracked, written by `writePluginMetadata()`) exists inside a deleted plugin directory, it prevents cleanup because the directory is not empty. This affects `/plugins` only. Planned fix: move metadata to a consolidated file at the plugin root.

**Known limitation:** Deletion is only detected from `p4 sync` output on the first sync after the Perforce deletion. If that sync runs before the fix is deployed, the directory becomes permanently orphaned. No orphan scan exists in `plugin-sync.ts` (unlike `shared-mirror-sync.ts` which has `localPluginDirectoriesMissingAfterSync()`).

### Single file deletion within a plugin

When individual files are deleted (e.g., a skill removed from a plugin that still exists):

1. `p4 sync` removes the file
2. Perforce does NOT remove empty parent directories (standard behavior)
3. `removeEmptyPluginDirs()` / `removeEmptyDirsSync()` recursively clean up empty directories in both layers

### Protecting local changes during sync

The sync client is read-only. Local modifications are protected via backup-sync-restore:

1. For each tracked file, compare current content hash against the stored baseline
2. If content diverged → back up the file content in memory
3. Run `p4 sync` (may overwrite the file)
4. Restore the backed-up content after sync
5. Protected files survive the sync unchanged

This approach avoids opening files for `p4 edit` in the sync client, which would accumulate opened file state and block future syncs.

**Caveat:** When an entire plugin is deleted from Perforce, `pruneDeletedPluginDirectories()` uses `rmSync(recursive: true)` which removes everything including locally modified files.

## Mirror State Versioning

The mirror state file includes a version number:

```json
{
  "version": 3,
  "targets": { "//entropia/.../plugins/...": { "baseCl": "205353" } },
  "files": { "my-plugin/SKILL.md": { "baseHash": "abc...", "baseCl": "205353" } }
}
```

When `loadState()` reads a state file with an older version, it discards the entire state and returns an empty state. This forces a full re-sync with the new sync logic.

Bump `STATE_VERSION` in `shared-mirror-sync.ts` whenever the sync logic changes in a way that requires existing mirrors to be re-synced.

| Version | Change |
|---|---|
| 1 | Original |
| 2 | Added empty directory cleanup |
| 3 | Per-workspace client names, read-only sync client |

## File Locking

`shared-mirror-sync.ts` uses a file lock to prevent concurrent sync processes:

- Lock file: `.plugins/.sync.lock`
- Implementation: atomic exclusive file creation (`writeFileSync` with `{ flag: "wx" }`)
- Lock contains PID + timestamp
- Stale lock detection: locks older than 5 minutes with dead process are auto-broken
- If lock cannot be acquired, the process exits cleanly (another sync is running)

## User Flows

### Use an existing shared plugin

1. `plugin-sync` updates `/plugins`
2. File watcher detects state file change (or frontend triggers freshness check)
3. Mirror sync runs, commands reload
4. Plugin available immediately

### Edit an existing plugin locally

1. Run `/plugin-management:clone` or `/plugin-management:sync`
2. Edit files under `.plugins/<plugin>`
3. Local `.plugins` version overrides shared `/plugins` at runtime
4. Submit changes with `/plugin-management:submit`

### Add a new plugin

1. Create plugin in `.plugins/<new-plugin>` for local testing
2. Plugin available immediately (local override)
3. Submit to Perforce when ready
4. `plugin-sync` picks it up for all workspaces

### Delete a plugin

1. Delete all files from Perforce and submit
2. `plugin-sync` syncs the deletion, prunes the directory
3. Mirror sync removes from `.plugins`, empty dirs cleaned
4. Command reload removes the plugin's commands/skills

## Debugging

### Check current state

```bash
# Shared state (what CL has plugin-sync reached?)
cat /plugins/.plugin-sync-state

# Local mirror state (what CL has this workspace synced to?)
cat /workspace/.plugins/.plugin-management-sync-state.json | head -10

# Compare — if shared CL > local CL, next freshness check will sync
```

### Check P4 client isolation

```bash
# Which sync client is this workspace using?
grep P4CLIENT /workspace/.plugins/.p4config

# Any opened files in the sync client? (should always be empty)
cd /workspace/.plugins && p4 opened 2>&1

# If files are opened → stale state from old shared-client era
# The sync will revert them automatically on next run
```

### Check if refresh triggered

```bash
# Workspace container logs
docker logs <workspace-container> 2>&1 | grep "plugin.refresh\|check-freshness\|freshness\|Command.*reload"

# Look for:
# - "shared plugin freshness check queued refresh" → sync will run
# - "shared plugin freshness check skipped" → CLs match, no sync
# - "plugin refresh completed" → sync finished, commands reloaded
# - "POST /global/plugin/check-freshness" → frontend triggered check
# - "skipping dependency install" → mirror dir correctly skipped install
```

### Check file watcher

```bash
docker logs <workspace-container> 2>&1 | grep "file.watcher.*plugin-sync-state"

# Look for:
# - event=add or event=change for .plugin-sync-state → watcher fired
# - No entries → watcher may not be detecting changes (Docker volume issue)
```

### Check plugin-sync sidecar

```bash
docker logs celest-plugin-sync --tail 20

# Look for:
# - "published plugins at changelist XXXX" → sync completed
# - "no plugin changes detected" → polling, nothing new
# - "sync failed" → check P4 connectivity
```

### Check for I/O storm symptoms

```bash
# Look for excessive bun install activity
docker logs <workspace-container> 2>&1 | grep "bun.*install\|installing package" | tail -20

# Should NOT see installs for any mirror plugin directory (/plugins/* or .plugins/*)
# The dependency install guard skips all plugin mirror directories unconditionally
```

### Force a re-sync

```bash
# Delete the mirror state to force full re-sync on next freshness check
rm /workspace/.plugins/.plugin-management-sync-state.json

# Then trigger from browser: refresh page or start new session
```

## Known Caveats

### Untracked files prevent deletion cleanup

Perforce sync only manages tracked files. Untracked content (node_modules, dist, caches, `.plugin-sync-meta.json`) keeps directories alive after tracked files are deleted.

### baseCl may not advance on deletion-only changelists

`updateState()` computes baseCl from `headChange` of remaining tracked files. A deletion changelist doesn't change `headChange` on surviving files, so baseCl may not advance. This can cause repeated freshness checks to think the mirror is stale.

### File watcher reliability across Docker volumes

The file watcher (Parcel watcher with inotify backend) may not reliably detect changes on Docker bind-mount volumes. The frontend freshness check triggers serve as a fallback for when the watcher misses events.

### Review panel refresh after sync

The review panel reads `session_diff`, not the live filesystem. After plugin refresh, the frontend clears the session diff cache and refetches, but if the backend's diff generation lags the sync, stale content may briefly appear.

## Recommendations

### For users

- Edit plugins under `.plugins`, not shared `/plugins`
- Submit real additions to Perforce when possible
- Avoid checking in `node_modules` or generated output
- If `.plugins` doesn't update, check for files opened in the sync client
- Refresh the page or start a new session to trigger a freshness check

### For plugin authors

- Keep plugin folders free of generated content
- Prefer reproducible setup over checked-in dependencies
- Only include `package.json` if the plugin has real dependencies
- Use `.p4ignore` to exclude build artifacts

### Future improvements

1. **Sidecar webhook**: POST to `/global/plugin/refresh` after sync for instant active workspace refresh
2. **Orphan scan in plugin-sync.ts**: Detect directories on disk that are no longer tracked in Perforce (the mirror layer has this via `localPluginDirectoriesMissingAfterSync()`, but the sidecar does not)
3. **Consolidated metadata file**: Move `.plugin-sync-meta.json` out of individual plugin directories to prevent deletion interference
4. **Batch `p4 files`**: Single depot query for all deletion candidates instead of per-plugin

## Source Reference

Key source files for the plugin-sync system:

| Component | File |
|---|---|
| Plugin-sync sidecar | `docker/plugin-sync.ts` |
| Sidecar Dockerfile | `docker/Dockerfile.plugin-sync` |
| Shared mirror sync | `opencode/packages/opencode/src/plugin/shared-mirror-sync.ts` |
| Plugin refresh orchestrator | `opencode/packages/opencode/src/plugin/refresh.ts` |
| Command watcher + reload | `opencode/packages/opencode/src/command/index.ts` |
| Plugin root resolution | `opencode/packages/opencode/src/shell/path-env.ts` |
| Dependency install guard | `opencode/packages/opencode/src/config/config.ts` |
| Frontend freshness check | `opencode/packages/app/src/utils/plugin-freshness.ts` |
| Frontend bootstrap trigger | `opencode/packages/app/src/context/global-sync.tsx` |
| Auth trigger | `opencode/packages/app/src/context/auth.tsx` |
| Session create trigger | `opencode/packages/app/src/components/prompt-input/submit.ts` |
| Global API routes | `opencode/packages/opencode/src/server/routes/global.ts` |
| Docker compose | `docker/docker-compose.yaml` |
