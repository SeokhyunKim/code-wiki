# code-wiki — moved

> **This plugin has moved to [robintech-seoul/agent-toolkit](https://github.com/robintech-seoul/agent-toolkit).**
> It now lives at [`claude/skills/code-wiki`](https://github.com/robintech-seoul/agent-toolkit/tree/main/claude/skills/code-wiki)
> and installs from the **`robintech`** marketplace. This repository is archived and will not receive further updates.

## Install from the new location

```
/plugin marketplace add robintech-seoul/agent-toolkit
/plugin install code-wiki@robintech
```

If you previously installed it from the `seokhyunkim` marketplace, uninstall that copy first:

```
/plugin uninstall code-wiki@seokhyunkim
```

Command names are unchanged (`/code-wiki:init`, `/code-wiki:build`, `/code-wiki:sync`, …), and an existing
`wiki/` directory keeps working as-is.

## What it is

A Claude Code plugin that builds and maintains a hierarchical, LLM-generated wiki over a codebase — leaf folders
summarize their files, parents synthesize their children, and topic pages capture cross-cutting concerns.
Full documentation, spec, and source: [agent-toolkit/claude/skills/code-wiki](https://github.com/robintech-seoul/agent-toolkit/tree/main/claude/skills/code-wiki).
