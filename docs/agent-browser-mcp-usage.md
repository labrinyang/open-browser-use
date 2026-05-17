# Agent Browser Use over MCP

This document is the agent-facing contract for driving open-browser-use through
the MCP `js` tool. It exists to keep LLMs on the intended path and to make token
costs visible before they turn into failed tool calls.

Reference points checked against the MCP specification:

- Tool results may include `structuredContent`, and tools that advertise an
  `outputSchema` should return structured data matching that schema:
  https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- Long-running tools can stream progress with `notifications/progress` when the
  client supplies a progress token:
  https://modelcontextprotocol.io/specification/2025-06-18/basic/utilities/progress
- Large artifacts should be exposed as resources or resource links instead of
  inlining the full payload in a tool result:
  https://modelcontextprotocol.io/specification/2025-06-18/server/resources

## Call Chain

1. The MCP client calls `tools/list` and receives `js`, `js_reset`, and
   `js_add_module_dir`, each with an input schema and output schema.
2. The client calls `tools/call` for `js` with JavaScript `source` and optional
   `timeout_ms`.
3. `crates/obu-node-repl/src/mcp_server.rs` validates arguments. If the request
   has a progress token, it installs a temporary progress sink.
4. `JsRuntimeManager` sends a JSONL `exec` frame to the Node kernel with request
   metadata, including session and turn identifiers.
5. `kernel.js` runs each cell as a fresh ES module in the same Node child. It
   installs locked globals: `agent`, `display`, `nodeRepl`, and `obuRepl`.
6. `await agent.browsers.get("chrome" | "cdp")` asks
   `obuRepl.discoverBackends()` for backend descriptors, opens the native pipe
   through the Rust broker, and sends `getInfo` to `obu-host`.
7. Browser commands flow through SDK guards, add session metadata, cross the
   native pipe broker, hit `obu-host` dispatcher policy, and then run in the CDP
   or WebExtension backend.
8. `display()` frames can stream as MCP progress for text/JSON. The final MCP
   result returns concise text `content` plus structured `stdout`, `stderr`,
   `result`, `duration_ms`, `truncated`, `displays`, and `error`.

## Canonical Agent Flow

```js
const browser = await agent.browsers.get("chrome");
const tab = await browser.tabs.create({ url: "http://127.0.0.1:8000/index.html" });
await tab.attach();

const heading = await tab.getByRole("heading", { name: /dashboard/i }).textContent();
display({ heading });

const shot = await tab.screenshot({
  type: "jpeg",
  quality: 60,
  fullPage: false,
  clip: { x: 0, y: 0, width: 900, height: 700, scale: 0.5 }
});
display({ __obuImage: true, mime_type: shot.mime_type, data: shot.data_base64 });
```

Use this order when possible:

1. Inspect state with locators, `textContent()`, `count()`, `readAll()`, URL, and
   title.
2. Use a clipped/compressed screenshot only when visual inspection is needed.
3. Use `tab.dev.cdp(...)` for missing Playwright APIs such as page evaluation.
4. Return small summaries as the last expression; send progress with small
   `display()` calls.

## Repeated Agent Mistakes to Avoid

- `agent.browsers.get(...)` is async. Always `await` it before reading `.tabs`.
- `browser.tabs.create()` accepts a URL string or `{ url }`. With no URL it uses
  `about:blank`.
- WebExtension sessions cannot drive `file://` pages. Serve local files over
  HTTP, for example `python3 -m http.server 8000`.
- There is no first-class `tab.evaluate()`. Use:

  ```js
  const result = await tab.dev.cdp("Runtime.evaluate", {
    expression: "document.title",
    returnByValue: true,
    awaitPromise: true
  });
  ```

- `process` and `node:process` are unavailable in the kernel. Use
  `nodeRepl.cwd`, `nodeRepl.homeDir`, `nodeRepl.tmpDir`, and
  `nodeRepl.requestMeta`.
- Default filesystem writes are blocked by Node permissions. Do not rely on
  `screenshot({ path })` or `writeFile` inside the MCP JavaScript cell.
- If a WebExtension service worker restart leaves a stale native socket path,
  call `js_reset`; reset respawns the kernel and refreshes runtime descriptors.

## Token Budget Hotspots

Current large-payload sources:

- `tab.screenshot()` and `tab.content.export({ format: "png" | "pdf" })`
  return base64 inline.
- `tab.content.export({ format: "html" })` base64-encodes the full document.
- `display({ __obuImage, data })` and `nodeRepl.emitImage(...)` store the image
  payload in the final `displays` array. Images are not progress-streamed.
- Text and JSON `display()` frames can stream as progress, but they are also
  included in final `displays`.
- `console.log`, `nodeRepl.write`, and the final expression all flow back in the
  final structured result.
- `tab.dev.cdp("Runtime.evaluate", { returnByValue: true })` can return very
  large objects if the expression serializes DOM, storage, or app state.
- DOM snapshots and locator `readAll()` results can be large on dense pages.

Use these defaults to keep results small:

- Never `console.log` or return raw screenshot/base64/HTML/PDF payloads.
- Do not write `display(await tab.screenshot())`; wrap the image explicitly as
  `{ __obuImage: true, mime_type, data }`.
- Prefer `type: "jpeg"`, `quality: 50..70`, `fullPage: false`, and a `clip`
  with `scale < 1` for screenshots.
- Return counts, selected text, dimensions, short arrays, or explicit summaries
  instead of whole documents or page state.
- When using CDP evaluation, project to a small shape in the page expression.

Known implementation gaps that still deserve follow-up:

- Large `stdout`, `result`, and `displays` are not truncated or spilled to MCP
  resources yet.
- Image displays still inline data in the final result instead of returning a
  resource link.
- HTML/PDF/content exports do not have a resource-backed path yet.
