---
description: Force regeneration of the wiki — entire tree (no args) or a sub-tree (path arg). Use after schema changes or when the wiki is structurally broken; for normal incremental updates, prefer `/code-wiki:sync`.
argument-hint: [<path>]
allowed-tools: Bash(python3 *), Bash(git *), Read, Write, Skill
---

The user wants to force-regenerate part or all of the wiki. The argument may be a path scoped under one of the configured source roots.

`$ARGUMENTS`

## Step 1: Preconditions

Same as `/code-wiki:build`:

1. `wiki/config.yaml` must exist (else suggest `/code-wiki:init`).
2. The project must be a git repo (capture HEAD SHA).

If `$ARGUMENTS` is empty: this is a **full rebuild** — equivalent to `/code-wiki:build --force`. Skip the path-validation step and proceed.

If `$ARGUMENTS` contains a path: this is a **scoped rebuild**. The path must be relative to the project root and must lie inside one of the configured source roots; the next step (walk-tree) validates this and surfaces a clear error if it doesn't.

## Step 2: Compute the work list

Run walk-tree with optional scoping:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/bin/walk-tree.py" \
    --project-root "$(pwd)" \
    ${SCOPE:+--scope-path "$SCOPE"}
```

Where `$SCOPE` is the path argument if provided, empty otherwise.

The output is the same JSON array as `/code-wiki:build` produces, optionally narrowed:
- **No arg**: every folder under every source root, bottom-up.
- **With arg**: the scoped sub-tree's folders (leaves and parents), plus the ancestor chain of the scope up to the source root, bottom-up.

If walk-tree errors with "not inside any configured source root", surface verbatim.

## Step 3: Read configuration

Read `wiki/config.yaml` to get `wiki_language` and `language_hints`.

## Step 4: Regenerate pages

For each work item, in order:

- `kind == "leaf"` → `Skill(skill="code-wiki:generate-leaf-page", args=<inputs>)`
- `kind == "parent"` → `Skill(skill="code-wiki:generate-parent-page", args=<inputs>)`

This unconditionally overwrites the existing page; that is the point of rebuild.

For full rebuild only: also clean up wiki pages that no longer correspond to any current source folder. After all skills run, compare:
- The set of `wiki_path` values produced by walk-tree.
- The set of existing wiki page files under each source root's wiki sub-directory.

Any wiki file that exists on disk but is NOT in the walk-tree output corresponds to a folder that has been removed. Add these to a `deletions` list and `rm -f` them. (For scoped rebuild, do not perform this cleanup — the scope may not cover the whole tree.)

## Step 5: Update state.json and log.md

Build the report:

```json
{
  "operation": "rebuild",
  "ingested_sha": "<HEAD>",
  "pages": <work-list output>,
  "deletions": <list from Step 4 cleanup, or [] for scoped rebuild>
}
```

```bash
echo '<json>' | python3 "${CLAUDE_PLUGIN_ROOT}/bin/state-update.py" --project-root "$(pwd)"
```

For a scoped rebuild, state-update will preserve untouched pages' state entries unchanged — only the regenerated pages get new content_hash values and refreshed source_to_wiki entries.

## Step 6: Topic regeneration (scoped)

If a topic page's recorded `source_files` intersects the regenerated paths, surface a notice that the topic may now be stale and suggest the user run `/code-wiki:topic <name>` for affected topics. (Automatic topic regeneration during rebuild is deferred to v2.)

## Step 7: Report

```
Rebuilt wiki:
- scope:               <path or "(full)">
- pages regenerated:   <count>
- pages deleted:       <count>
- last_ingested_sha:   <sha>
```

## Failure modes

- Invalid scope (not inside any source root) → walk-tree surfaces the error; relay to user.
- Skill produces no file → log and continue (do not fail the whole rebuild).
- state-update fails → surface stderr; rebuild output may be inconsistent.

## When to use this vs. sync vs. build

- **`/code-wiki:build`**: first-time generation; aborts if `wiki/<source-root>/` already has content.
- **`/code-wiki:sync`**: incremental, git-diff based; the everyday command.
- **`/code-wiki:rebuild`**: ignore state, force regeneration. Use after schema/template changes, when wiki is structurally broken, or to refresh prose quality.
- **`/code-wiki:rebuild <path>`**: same, but only that subtree + ancestors. Useful for iterating on prose for a specific module without re-running the whole repo.
