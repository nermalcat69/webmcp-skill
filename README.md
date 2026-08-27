# webmcp-skill

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for adding
[WebMCP](https://github.com/webmachinelearning/webmcp) tools to a React storefront, so a
browser-connected AI agent can search the catalogue and drive the cart through typed
calls instead of scraping the DOM.

## What it does

Walks through exposing tools like `search_products`, `add_to_cart`, and `go_to_checkout`
via `document.modelContext`, wired so they run the same code paths as the UI and the
React view updates live. The core of the skill is the **no-stale-closure bridge
pattern** — `useWebMCP` registers each tool once and never re-registers, so tool
callbacks read live state through module-level refs instead of closing over React state.

It is stack-agnostic: assumes a React app with client-side routing and cart state in a
provider. You adjust import paths and hook names to fit your project.

## Install

Copy `skill.md` into your skills directory:

```
.claude/skills/webmcp-storefront/skill.md
```

(or `~/.claude/skills/webmcp-storefront/skill.md` for all projects).

## Use

Ask Claude Code to "add WebMCP to this store", "expose tools to the agent", or
"wire up document.modelContext". The skill covers install, the bridge module, the
tools component, the mount point, `.mcp.json` for local testing, and testing in
Chrome 149+ with the WebMCP flag.
