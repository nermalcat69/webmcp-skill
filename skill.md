---
name: webmcp-storefront
description: >-
  Add or maintain WebMCP tools on a React storefront so browser AI agents can
  search the catalogue and drive the cart through typed calls instead of DOM
  scraping. Use when asked to "add WebMCP", "expose tools to the agent", "make
  the store agent-usable", wire up document.modelContext, or extend/debug an
  existing webmcp-tools component. Covers the no-stale-closure bridge pattern,
  mount point, .mcp.json, and Chrome testing.
---

# WebMCP on a storefront

WebMCP lets an AI agent connected to the browser tab call typed tools
(`add_to_cart`, `search_products`, …) that run the same code paths the UI uses,
so the React UI updates live. It makes an agent **effective once it is on the
page** — it is not discovery/SEO.

This skill is stack-agnostic: it assumes a React app with client-side routing
(React Router, TanStack Router, Next.js `useRouter`, etc.) and some cart/store
state held in a provider. Adjust the import paths and hook names below to match
the project you are in.

## Reach — which agents can use this

`document.modelContext` is the W3C WebMCP draft. Today it only exists in
Chrome 149+ behind `chrome://flags/#enable-webmcp-testing` (or
`--enable-features=WebMCP`), reached via `chrome-devtools-mcp`. This is the
right target long-term and the code you write here is spec-aligned.

If you need it working **now** in an arbitrary MCP client (Claude Desktop, etc.)
with no browser flag, that needs the older script-tag + relay-server + token
pairing approach (`jasonjmcghee/WebMCP`, `@jason.today/webmcp`). It is **not**
spec-compliant and the author now points people at the W3C spec — don't build
on it unless the "works today, any client" constraint is hard. This skill does
not cover it.

## The one rule that bites

`use-webmcp-tool`'s `useWebMCP` registers each tool **once** and does **not**
re-register when `execute` changes. So `execute` callbacks must never close
over React state, props, or context values — they will be stale forever.

Read live state through, in order of preference:

1. **Static module data** (product arrays, pure helpers) — never changes, import directly.
2. **A module-level bridge ref** updated every render by a `useEffect` in the provider that owns the state.
3. **A module-level nav ref** set by a `useEffect` in the tools component.

## Procedure to add it to a new storefront

1. **Install:** `npm i use-webmcp-tool@^0.2.0` (add `--legacy-peer-deps` if the
   repo already needs it).

2. **Bridge module** — e.g. `src/lib/webmcp-bridge.ts`. Export one ref per piece
   of live state the tools need, plus one for navigation:

   ```ts
   import type { useCart } from "@/components/cart-provider";      // your cart hook
   export const cartBridge: { current: ReturnType<typeof useCart> | null } = { current: null };
   export const navBridge: { current: ((path: string) => void) | null } = { current: null };
   ```

3. **Feed the bridge** — in the provider that owns the state, build the context
   value as a named `const value` and add, with **no dep array**:

   ```ts
   useEffect(() => { cartBridge.current = value; });
   ```

4. **Tools component** — e.g. `src/components/webmcp-tools.tsx`, renders `null`:
   - Top-of-file `unhandledrejection` handler swallowing `use-webmcp-tool`
     `AbortError`s (expected unmount noise).
   - `useEffect(() => { navBridge.current = navigate; }, [navigate])` where
     `navigate` comes from the project's router.
   - One `useWebMCP({ name, description, inputSchema, annotations, execute })`
     per tool. `annotations: { readOnlyHint: true }` for reads,
     `{ readOnlyHint: false }` for writes. Add `untrustedContentHint: true` on
     any read that returns user-generated text (reviews, seller descriptions,
     Q&A) so the agent treats it as untrusted.
   - `execute` reads `cartBridge.current` / static data only. After a mutation,
     `await new Promise(r => setTimeout(r, 30))` before returning a cart
     snapshot so it reflects the scheduled state update.
   - Throw `Error` with a helpful message for bad input (unknown slug/variant);
     the agent sees it. Optional `onError` for logging, `formatOutput` to shape
     the result before MCP normalization.
   - `useWebMCP` re-registers on `name`/`inputSchema` changes but **not** on
     `execute` changes (hence the bridge). Pass `enabled: false` to
     conditionally drop a tool; `supported`/`registered`/`error` come back for
     diagnostics.

5. **Mount once** — inside the state provider and within the router context,
   render `<WebMCPTools />` (the root layout is usually the right place).

6. **`.mcp.json`** at repo root for local agent testing:

   ```json
   { "mcpServers": { "chrome-devtools": { "command": "npx", "args": ["-y", "chrome-devtools-mcp@latest", "--categoryExperimentalWebmcp", "--autoConnect", "--no-usage-statistics"] } } }
   ```

7. **Guide** — drop a short `docs/webmcp.md` in the repo listing the tool table
   and how to test.

## Tool set (baseline)

`search_products`, `get_product_details`, `get_cart`, `add_to_cart`,
`update_cart_quantity`, `remove_from_cart`, `apply_coupon` (reuse the existing
coupon-validation path; on success write whatever key checkout reads),
`go_to_checkout` (navigate only). Add domain tools (e.g. a recommender) by
wrapping an existing pure engine — never fork the logic.

**Never** add a `place_order` / `checkout` tool that completes payment. Checkout
stays manual (address + payment). `go_to_checkout` just navigates.

## Constraints

- All tools are SSR-safe: `use-webmcp-tool` guards every `document.modelContext`
  access inside `useEffect`. The component renders `null` on the server.
- Tools ship to production but are inert where WebMCP is unsupported
  (`useWebMCP` returns `supported: false`).
- No secrets, no privileged paths — a tool may only do what an anonymous user
  can already do in the UI.
- Tools import the same modules the UI does, so they follow catalogue/logic
  changes automatically. Keep it that way — no forked data.

## Testing

- Chrome 149+ with `chrome://flags/#enable-webmcp-testing` → Enabled → relaunch
  (or `--enable-features=WebMCP`). Verify: `'modelContext' in document` → `true`.
- `npm run dev`, open in flagged Chrome, start Claude Code from repo root,
  approve the `chrome-devtools` MCP server, keep the store tab focused, set
  `execute_webmcp_tool` to always-allow.
- Smoke prompt: something that chains a read and a write, e.g. *"Find a product
  matching X, then add it to the cart."* → the relevant read tool →
  `add_to_cart`; the cart badge updates on its own.
- `npx tsc -b` — confirm zero **new** errors in the added files (existing
  unrelated `tsc` noise in the repo is fine).
