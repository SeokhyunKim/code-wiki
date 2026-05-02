---
description: Incrementally update the wiki to reflect source changes since the last ingest. Diffs git from `last_ingested_sha` to HEAD and regenerates only affected pages plus their ancestors.
argument-hint: (no arguments)
allowed-tools: Bash(python3 *), Bash(git *), Read, Write, Skill
---

The user wants to sync the wiki with current source. No arguments.

## Step 1: Preconditions

1. **Git repo?** Run `git rev-parse HEAD`. If it fails, abort:
   > "code-wiki:sync requires a git repository. (For non-git workflows, use `/code-wiki:rebuild`.)"

2. **Wiki initialized?** Verify `wiki/config.yaml` exists. If not:
   > "wiki/config.yaml not found. Run `/code-wiki:init` first."

3. **State present?** Check `.code-wiki/state.json`.
   - If missing, run **soft-bootstrap** first:
     ```bash
     python3 "${CLAUDE_PLUGIN_ROOT}/bin/state-bootstrap.py" --project-root "$(pwd)"
     ```
     - If it succeeds (typical path: a fresh clone with committed `wiki/`), continue.
     - If it errors with "wiki/ has not been committed", surface the message and abort (suggest `/code-wiki:build` instead).

## Step 2: Compute dirty work-set

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/bin/scan-changes.py" --project-root "$(pwd)"
```

The output JSON has shape:

```json
{
  "from_sha": "...",
  "to_sha":   "...",
  "pages":      [<work items, leaf or parent, in bottom-up order — same shape as walk-tree>],
  "deletions":  ["wiki/src/old/index.md", ...],
  "dirty_topics": ["wiki/topics/auth.md", ...]
}
```

If `pages`, `deletions`, and `dirty_topics` are all empty, report:
> "Wiki is up to date. (No source changes between `<from_sha>` and `<to_sha>`.)"
…and exit. Do not call state-update for a no-op.

## Step 3: Read configuration

Read `wiki/config.yaml` to get `wiki_language` and `language_hints`. Defaults: `en` / `[]`.

## Step 4: Apply deletions

For each path in `deletions`, delete the file and any now-empty parent directory under `wiki/`:

```bash
rm -f "<wiki_path>"
# After rm, prune empty ancestors up to (but not including) wiki/ itself.
```

Do not delete topic pages here — those are tracked separately in `dirty_topics`.

## Step 5: Regenerate dirty leaves and parents

For each item in `pages`, in order (bottom-up):

1. Ensure the wiki page's parent directory exists.
2. Invoke the appropriate skill — same call shape as `/code-wiki:build`:

   - `kind == "leaf"` → `Skill(skill="code-wiki:generate-leaf-page", args=<JSON inputs>)`
   - `kind == "parent"` → `Skill(skill="code-wiki:generate-parent-page", args=<JSON inputs>)`

3. Verify the wiki file exists post-skill. If not, log and continue (don't fail the whole sync).

The bottom-up ordering ensures parent generation reads freshly-regenerated children.

## Step 6: Handle dirty topics

If `dirty_topics` is non-empty, surface them to the user:

> "These topic pages reference sources that changed and may be stale: `<list>`. Run `/code-wiki:topic <name>` to regenerate each. (Automatic topic regeneration is a v2 feature.)"

Do not regenerate topic pages in v1 sync — that's deferred to T19's `/code-wiki:topic` command. Do still record them in the state-update report so users see them in the log.

## Step 7: Update state.json and log.md

Build the report from the scan-changes output:

```json
{
  "operation": "sync",
  "ingested_sha": "<scan_output.to_sha>",
  "pages": <scan_output.pages, filtered to those whose wiki file actually exists post-skill>,
  "deletions": <scan_output.deletions>
}
```

```bash
echo '<report json>' | python3 "${CLAUDE_PLUGIN_ROOT}/bin/state-update.py" --project-root "$(pwd)"
```

If state-update exits non-zero, surface stderr and warn the user that state may be inconsistent.

## Step 8: Report

```
Synced wiki: <from_sha> .. <to_sha>
- regenerated leaves:  <count>
- regenerated parents: <count>
- deleted pages:       <count>
- dirty topics needing manual regen: <list, or "none">
```

## Failure modes

- `state-bootstrap` errors → surface verbatim. If it's "wiki/ not committed", suggest build.
- `scan-changes` exit 2 (no last_ingested_sha) → suggest `/code-wiki:build`.
- Skill produces no file → record in summary, do not fail the sync.
- `state-update` errors → state may be stale; surface the error and recommend `/code-wiki:lint --fix` after the user fixes the underlying issue.

## What this command does NOT do

- Generate new topic pages (use `/code-wiki:topic`).
- Modify `wiki/CLAUDE.md` or `wiki/config.yaml`.
- Touch source files.
- Commit anything.
