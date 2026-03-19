Title: ChatVault MCP App

project name - chat-vault-part1

This project uses the **generic MCP App prompts** defined in:

- `prompts/part-mcp-app/common.md`

Use that file for:

- **Prompt1**: Detach from GitHub repository
- **Prompt2**: Scaffold  MCP App project
- **Prompt3**: Implement HTTP JSON-RPC MCP server with ext-apps
- **Prompt4**: Build the MCP App widget as a single-file HTML bundle (Vite + React + Tailwind + vite-plugin-singlefile)
- **Prompt5**: Wire MCP App UI resources and a test tool (registerAppTool, registerAppResource)
- **Prompt6**: Deployment, logging, and end-to-end verification

This file defines the **ChatVault-specific MCP App behavior** starting from Prompt7.

## Engineering Principles (ChatVault-specific)

- **Reuse old-depricated-part 1 behavior, not their plumbing**: The new Part1 MCP App is a port to MCP App of an older ChatGPT App Part 1 (widget), Part 2 (backend) is an MCP server, stays unchanged, implemented with the MCP Apps SDK (ext-apps) as in the generic Part MCP App prompts. Use `registerAppTool` and `registerAppResource` for correct tool/resource shape and widget iframe behavior.
- **Simple, explicit tools**: Treat `browseMySavedChats` as an MCP App tool whose job is to open the ChatVault widget and hand it enough context to know which user/team to show. Use a flat input schema (raw Zod shape: optional `shortAnonId`, `portalLink`, `loginLink`, `isAnon`) and a handler that returns text plus `_meta.ui.resourceUri`.
- **Responsive UX**: The widget is for humans, not test harnesses. Prioritize fast feedback (loading states, clear empty/error messages, obvious "open portal" actions) over protocol cleverness.
- **Traceability across layers**: From the client down to the MCP server and widget, you should be able to trace a single tool call via logs and UI states. When in doubt, add a small, well-scoped log with a unique tag and keep the log volume bounded.


Prompt4 (ChatVault-specific app/widget prompt): --- Note: this is where the user defines the actual widget component(s) and MCP actions for ChatVault.

This is the MCP server we want to build: actions: saveChat, loadChats, searchChat and browseSavedChats.

A Chat is { title, timestamp, turns[{ prompt, response }] }.
browseSavedChats returns the widget we are creating in this project.
loadChats is a paged fetch, hardcoded with example data for this project.
saveChat and searchChat are dummy functions for this project.
The widget is like a Chrome history browser, internally calling loadChats via skybridge (window.openai.toolCall). It is a list of chats.When user clicks on a chat, it opens up, showing all the saved prompts and responses. The prompts and responses are truncated with ellipses by default, but when clicked, they show in full. Each has copy to clipboard button which changes to green checkmark for 5 secs when clicked.
loadChats should have userId as a parameter, but we are not setting it inside the widget.
at the bottom of the widget, add collapsible debug panel. Add logging to widget and show in the debug panel. Load widget initi and calling loadChats thoroughly.
Note: The widget must detect and adapt to dark mode (use data-theme attribute, CSS variables, and dark: Tailwind classes).

After you implement these ChatVault-specific tools, resources, and widget behavior for Prompt4, go back to Prompt3 in openai-AppsSDK-prompt.md and update the Jest + e2e tests so they cover the actual live MCP server (including the new browseSavedChats behavior) end-to-end via /mcp. When doing so, take into account the isolated widget behavior and failure modes you validated in Prompt5 so tests exercise both protocol-level correctness and real widget behavior.

Prompt5 (Isolated ChatVault widget test on port 4444):

Set up a simple static server to serve the built widget assets from the ChatVault project root:
From the tutorial root:
cd /home/nick/chatvault-tutorial/chat-vault-part1
npx serve assets -l 4444
Verify that the ChatVault widget can be loaded in isolation in a regular browser (outside ChatGPT) by opening:
http://localhost:4444/chat-vault.html
Use this isolated page to:
Confirm that the widget HTML, JS, and CSS are valid and that React mounts successfully.
Exercise basic UI interactions (header rendering, empty-history state, expand/collapse of turns, debug panel toggle) without requiring window.openai.callTool.
Diagnose widget-only issues (for example, runtime errors, host-API absence, or layout/styling problems) independently of MCP transport and ChatGPT’s hosting behavior.
Verify that the widget handles missing or delayed host APIs (for example, window.openai) in a bounded, observable way (clear error or retry message, no infinite retries or unbounded logging), and that purely local interactions (such as debug panel toggling) remain responsive.

---

Prompt7: Define ChatVault MCP App inputs and wiring for `browseMySavedChats`

This is the MCP App server we want to build: actions: saveChat, loadChats, searchChat and browseSavedChats.

A Chat is { title, timestamp, turns[{ prompt, response }] }.
browseSavedChats returns the MCP App widget we are creating in this project.
loadChats is a paged fetch, hardcoded with example data for this project.
saveChat and searchChat are dummy functions for this project.
The widget is like a Chrome history browser, internally calling loadChats via skybridge (window.openai.toolCall). It is a list of chats.When user clicks on a chat, it opens up, showing all the saved prompts and responses. The prompts and responses are truncated with ellipses by default, but when clicked, they show in full. Each has copy to clipboard button which changes to green checkmark for 5 secs when clicked.
loadChats should have userId as a parameter, but we are not setting it inside the widget.
at the bottom of the widget, add collapsible debug panel. Add logging to widget and show in the debug panel. Load widget initi and calling loadChats thoroughly.
Note: The widget must detect and adapt to dark mode (use data-theme attribute, CSS variables, and dark: Tailwind classes).


Goal: Specify how the Part MCP App receives context from the client (e.g. ChatGPT) and how that flows into the widget.


browseMySavedChats:
Requirements:

- Tool contract:
  - `name: "browseMySavedChats"`.
  - `inputSchema`: use a **raw Zod shape** (plain object of optional fields), e.g. `shortAnonId`, `portalLink`, `loginLink`, `isAnon`, as in the old-depricated-part1 prompt. Avoid `z.object({...}).passthrough()` for ext-apps type compatibility.
  - Treat this as a routing tool that opens the widget; keep validation minimal.
- Expected inputs:
  - A short anon/user ID (for example, `shortAnonId` or equivalent) to identify the viewer inside ChatGPT.
  - A `portalLink` URL where the full ChatVault web app lives. Will be injected in the future (part3).
  - A `loginLink` URL to sign in or upgrade if needed. Will be injected in the future.
  - Booleans for `isAnon` / `isAnonymousSubscription` as needed. Will be injected in the future. 
- Behavior:
  - On `tools/call` for `browseMySavedChats`, the server should:
    - Log the incoming arguments (keys only; avoid dumping secrets).
    - Return a result that:
      - Contains friendly text explaining what's happening (for example, "Opened ChatVault! Use the widget to browse your saved chats.").
      - Includes `_meta` or structured content pointing the client at the widget resource URI (for example, `ui://chat-vault/mcp-app.html`) and, if appropriate, echoing the `portalLink`/`loginLink` in a safe, clear way that the widget can read via `result.meta`.
- The focus here is on **shape clarity**: the tool should be trivially understandable by "vibe engineers" looking only at the JSON schema and a couple of log lines.

---

Prompt8: ChatVault MCP App widget behavior

Goal: Implement the Part MCP App widget UI so it can be used as an MCP App iframe inside ChatGPT/Claude and as a regular page (for local testing).

Requirements:

- Visual behavior:
  - Reuse the high-level layout from old-depricated-part1 (a chat history browser), but simplify wherever it makes sense for "portal-style" usage.
  - Show a **header** with the user/team name (if available) and a clear title like "ChatVault – Saved Chats".
  - Provide obvious actions:
    - "Open full app" button linking to `portalLink` (in a new tab) when available.
    - "Sign in / create account" button linking to `loginLink` when appropriate.
- Data behavior:
  - Treat the widget as **read-only** with respect to saved chats:
    - The widget talks to the client via the MCP Apps SDK (e.g. `app.callServerTool("loadMyChats", ...)` or `window.openai.callTool`), so it can load chats through the same MCP server that served the widget (or via client-provided `_meta`).
  - When no chats are available, show a friendly empty state with a link to the portal.
- Debug panel:
  - Include a collapsible debug panel (similar to Part 1) that shows:
    - The raw `result.meta` (or `_meta`) from the client.
    - Any tool calls the widget makes (method, tool name, result shape).
    - Environment info (light/dark mode, client type, any MCP/App-specific flags).
- Theme:
  - Respect light/dark mode via `data-theme` and/or CSS variables, as in Part 1.

---

Prompt9: End-to-end validation from ChatGPT/Claude

Goal: Prove that the full chain works:

- ChatGPT/Claude → Part MCP App on Vercel → widget.

Requirements:

- Add **structured logging** in the Part MCP App to capture:
  - Incoming JSON-RPC request summary (method, id, `params` keys).
  - Tool dispatch events (`tools/call` → `browseMySavedChats`).
  - Outgoing JSON-RPC responses (status, result vs. error, response byte length).
- Ensure there is:
  - A single place where `shortAnonId`, `portalLink`, and `loginLink` are assembled and passed to the `browseMySavedChats` tool.
  - A log line that shows the exact tool args used for each call (without leaking secrets).
- Manual verification steps (to be run by a human or scripted later):
  - From ChatGPT, call the tool and confirm:
    - The widget opens and shows expected user/team.
    - The debug panel logs the same IDs/URLs that were logged server-side.
    - Error paths (missing portal link, invalid token, rate limiting) are visible in both the widget and the MCP App logs.

The outcome should be a **boring, predictable MCP App**: it uses the MCP Apps SDK (ext-apps) for tools and resources, a single-file widget build (Vite + vite-plugin-singlefile), and feels delightful at the surface while remaining easy for vibe engineers to debug and extend.

---

Prompt10 (optional): Display mode and layout

Goal: Support display mode (pip / inline / fullscreen) and safe area so the widget adapts to the client's layout.

Requirements:

- Use `window.openai.requestDisplayMode` (or equivalent) if the client exposes it; otherwise degrade gracefully.
- Respect safe area insets from the client when laying out the widget (e.g. avoid notches, system UI).
- Keep the widget usable in all display modes (pip, inline, fullscreen) with appropriate min/max height and scrolling.
