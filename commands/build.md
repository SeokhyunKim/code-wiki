---
description: Generate the entire wiki from scratch (bottom-up) — runs once at the start of a project, or after a structural reset. For incremental updates, use `/code-wiki:sync`.
argument-hint: [--force]
allowed-tools: Bash(python3 *), Bash(git *), Read, Write, Skill
---

The user wants to build the wiki from scratch. The arguments may include `--force`.

`$ARGUMENTS`

## Step 1: Preconditions

Run these checks in order. Stop on the first failure with a clear message.

1. **Wiki initialized?** Verify `wiki/config.yaml` exists in the project root. If not, abort:
   > "wiki/config.yaml not found. Run `/code-wiki:init` first."

2. **Git repo?** Run `git rev-parse HEAD` and capture the SHA. If it fails (not a git repo, or no commits), abort:
   > "code-wiki requires a git repository with at least one commit. Initialize git and commit your source first."

3. **Wiki already populated?** Check whether any source root's subdirectory under `wiki/` already has content. The source roots come from `wiki/config.yaml`'s `source_roots:` list — for each, look at `wiki/<source_root_path>/`.
   - If any of these directories exists and contains files, **and** `$ARGUMENTS` does NOT contain `--force`, abort:
     > "wiki/<source-root>/ already has content. Re-run with `--force` to overwrite, or use `/code-wiki:sync` for incremental updates."
   - With `--force`, proceed (existing files will be overwritten file-by-file as we regenerate).

## Step 2: Build the work list

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/bin/walk-tree.py" --project-root "$(pwd)"
```

Capture the JSON output. It's an array of work items, each with:
- `source_root`, `folder_relpath`, `kind` (`leaf` or `parent`), `source_files`, `child_wikis`, `wiki_path`.

The order is **bottom-up**: every leaf comes before its parents. This is the order you must iterate — generating a parent before its children would read stale child wikis.

If the array is empty, abort:
> "No source folders to wikify. Check `source_roots` and `ignore_patterns` in wiki/config.yaml."

## Step 3: Read configuration

Read `wiki/config.yaml` to get `wiki_language` and `language_hints`. You'll pass these to each skill invocation. If `wiki_language` is missing, default to `en`.

## Step 4: Generate pages — iterate the work list in order

For each work item in the JSON, in order:

1. Ensure the wiki page's parent directory exists (`mkdir -p` of `dirname wiki_path`).
2. Invoke the appropriate skill:

   - If `kind == "leaf"`:
     ```
     Skill(skill="code-wiki:generate-leaf-page", args="""
       {
         "folder_abs": "<absolute path to folder = $(pwd)/folder_relpath>",
         "folder_relpath": "<work_item.folder_relpath>",
         "source_root":     "<work_item.source_root>",
         "loose_files":     [<basenames from work_item.source_files>],
         "target_wiki_relpath": "<work_item.wiki_path>",
         "wiki_language":   "<from config>",
         "language_hints":  [<from config>]
       }
     """)
     ```
   - If `kind == "parent"`:
     ```
     Skill(skill="code-wiki:generate-parent-page", args="""
       {
         "folder_abs": "<absolute path>",
         "folder_relpath": "<...>",
         "source_root":     "<...>",
         "loose_files":     [<basenames>],
         "child_wiki_paths": <work_item.child_wikis>,
         "target_wiki_relpath": "<work_item.wiki_path>",
         "wiki_language":   "<from config>",
         "language_hints":  [<from config>]
       }
     """)
     ```

3. After the skill returns, verify the wiki file was created at `wiki_path`. If not, treat as a generation error — log it and continue to the next item (build is best-effort across folders; partial output is acceptable).

**Important**: do not parallelize parent generation. A parent's children must be on disk before the parent is generated. The work list ordering already enforces this; just don't reorder.

## Step 5: Update state.json and log.md

Once all skills have run, build a JSON report:

```json
{
  "operation": "build",
  "ingested_sha": "<HEAD SHA from Step 1>",
  "pages": <the original work-list array, possibly filtered to drop any items where the wiki file was not actually written>,
  "deletions": []
}
```

Pass it to:

```bash
echo '<report json>' | python3 "${CLAUDE_PLUGIN_ROOT}/bin/state-update.py" --project-root "$(pwd)"
```

The script:
- Computes content hashes for each generated wiki page.
- Computes source hashes for each referenced source file.
- Updates `source_to_wiki` and `source_hashes`.
- Sets `last_ingested_sha`.
- Appends a build entry to `.code-wiki/log.md`.

If `state-update.py` exits non-zero, surface its stderr to the user — state has not been written and the wiki may be inconsistent.

## Step 6: Report

Print a concise summary:

```
Built wiki at <project_root>/wiki/
- pages generated: <N>
- source roots:    <list>
- last_ingested_sha: <sha>
Run /code-wiki:lint to check the result, or /code-wiki:query <q> to use it.
```

## Failure modes

- A skill invocation produces an empty or malformed page → note in summary, do not fail the whole build. The user can re-run targeted with `/code-wiki:rebuild <path>` later.
- `walk-tree.py` errors → surface stderr verbatim and abort.
- `state-update.py` errors → surface stderr verbatim. State may be stale; suggest re-running build.

## What this command does NOT do

- Generate topic pages. Those are produced by `/code-wiki:topic` (explicit) or `/code-wiki:query` (lazy).
- Modify `wiki/CLAUDE.md` or `wiki/config.yaml`. Those are user-curated.
- Touch source files.
- Commit anything. `wiki/` and `.code-wiki/` are written but not staged or committed.
