# Deploy and Host MCPJam Inspector with Railway

MCPJam Inspector is the open-source testing and evaluation platform for MCP
server developers. This template compiles it from the official source tree and
runs it as one Railway service — a Hono/Node API plus a compiled React client —
behind a public HTTPS URL. You get tools, resources, prompts, the OAuth
debugger, a widget emulator and the LLM playground, with no `node`, no `docker`
and no tunnel on your machine. Access is gated by the Inspector's own session
token, which Railway generates for you.

## About Hosting MCPJam Inspector

The Inspector normally runs locally (`npx @mcpjam/inspector`, or a desktop
app). Hosting it buys you a URL: test an MCP server from a phone or a second
machine, hand a colleague a live server configuration instead of a stale JSON
snippet, and keep a standing rig pointed at staging. The app is stateless, so
there is no database, no volume and no migration — one service, one public URL.

**Works hosted:** any public HTTPS MCP server (Streamable HTTP and SSE),
including OAuth-protected ones; tools, resources, prompts, elicitation,
logging, conformance checks; the LLM playground with your own key; the
ChatGPT-apps / MCP-apps widget emulator; the OAuth debugger.

**Does not work hosted:** STDIO servers (a container has no `npx` of yours),
`localhost` servers, reading your machine's `~/.mcp` or skills, and
tab-to-tab continuations designed for `127.0.0.1`. The terminal and browser
tools are disabled by this template — see Security.

**Getting in:** the Inspector is gated by a URL fragment, not a login. After the
deployment succeeds, open the service's deploy log and copy the printed link:

```text
✔ MCPJam Inspector is ready
  ➜ Open MCPJam
  https://YOUR-SERVICE.up.railway.app/#token=YOUR-TOKEN
```

Open that whole link in a browser; the `#token=` fragment signs the browser in
and is stripped from the address bar.

## Common Use Cases

*   Reproduce an MCP client's failing call end to end, with a URL a teammate can
    open instead of a log file.
*   Debug OAuth message by message — `/authorize` → `/token` across protocol
    versions, including DCR and CIMD.
*   Render a ChatGPT-apps or MCP-apps widget in the built-in emulator, without
    ChatGPT, ngrok or a local dev server.
*   Share one live MCP server configuration with a teammate, an agency client
    or a QA reviewer, and keep a public demo of your server for prospects.

## Dependencies for MCPJam Inspector Hosting

Nothing. This template is a single service with no database, no volume, no
sidecar and no third-party account to sign up for. The build itself pulls Node
24, Chromium and Playwright from their public upstreams inside Railway's
builder.

### Deployment Dependencies

*   [MCPJam Inspector (upstream source)](https://github.com/MCPJam/inspector) —
    vendored here at a pinned commit.
*   [MCPJam documentation](https://docs.mcpjam.com) · [mcpjam.com](https://www.mcpjam.com)
*   Build-time network egress to `registry.npmjs.org`, `cdn.playwright.dev` and
    the Debian apt mirrors. A build failure there is an egress problem, not a
    code problem — redeploy.

### Implementation Details

| Variable | Value | Why |
|---|---|---|
| `MCPJAM_SESSION_TOKEN` | generated, 32 chars | The credential in the access link |
| `MCPJAM_ALLOWED_HOSTS` | `${{RAILWAY_PUBLIC_DOMAIN}}` | Host allowlist for the token gate and the browser Origin check |
| `ALLOWED_ORIGINS` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Exact origins allowed to call the API |
| `MCPJAM_INSPECTOR_FRONTEND_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | The address the app prints and builds callbacks from |
| `PORT` | `6274` | Keeps the app and Railway's health check on one port |
| `MCPJAM_LOCAL_COMPUTER_ENABLED` | `false` | No shell on a public URL |
| `MCPJAM_LOCAL_BROWSER_ENABLED` | `false` | Same |
| `RAILWAY_HEALTHCHECK_TIMEOUT_SEC` | `600` | Room for the first, slower start |

Because these reference `RAILWAY_PUBLIC_DOMAIN`, they follow your domain
automatically. If you attach a custom domain, update the two allowlist
variables by hand. Health check is `GET /health` — public and unauthenticated by
design, which is what lets Railway probe it.

**On the deploy screen, accept the template's values.** Railway asks you to
confirm the four whose values are literal: `PORT`,
`MCPJAM_LOCAL_COMPUTER_ENABLED`, `MCPJAM_LOCAL_BROWSER_ENABLED` and
`RAILWAY_HEALTHCHECK_TIMEOUT_SEC`. Do not blank the two `false` flags: the app
reads them as `MCPJAM_LOCAL_COMPUTER_ENABLED !== "false"`, so an empty value does
not mean "off" — it *enables* the terminal tool. That is the one way to end up
with a public shell on a public URL.

### Why Deploy MCPJam Inspector on Railway?

*   **One click to a working URL** — no local toolchain, no tunnel, no firewall
    rule, no second machine.
*   **A reproducible build** — the source is pinned and built with upstream's
    own Dockerfile, so a deployment is a known-good artifact.
*   **HTTPS by default** — Railway terminates TLS. That matters here: browsers
    will not grant a widget iframe the permissions it needs over plain HTTP.
*   **No lock-in** — the same Inspector runs from `npx`, a desktop app or a
    container. Railway hosts it; it does not own it.
*   **Cheap and disposable** — one stateless service, no state to migrate, and
    sleeping or deleting it is a complete off switch.

**Cost: ~US$5–6 per month** always-on — measured at 0.46 GB resident memory and
0.02 % of one vCPU while idle, billed at Railway's usage rates of US$10 per
GB-month of memory and US$20 per vCPU-month. Limits are ceilings, not
reservations, so a generous cap costs nothing until it is used. Turn on **App
Sleeping** to cut the idle cost; the token is pinned in a variable, so the access
link survives a sleep (in-memory state does not).

## Security

*   **URL + token is the entire security model.** No accounts, no login, no rate
    limit. Treat the access link like an SSH deploy key, and rotate the token
    when someone leaves.
*   **Terminal and browser tools are off by default.** Re-enabling
    `MCPJAM_LOCAL_COMPUTER_ENABLED` hands an arbitrary shell to anyone with the
    link. Inside a container it is not your laptop's shell, but it is still a
    shell.
*   **`*.up.railway.app` hostnames are guessable** from your project name; a
    custom domain you control can be pointed elsewhere or taken down.
*   **The origin allowlist is not a firewall** — it stops cross-origin browsers
    and DNS rebinding. The token is what authorises access.
*   **Anyone signed in can spend your LLM credits** and make the Inspector call
    tools on any server they connect. This is a development tool, not a
    production front end.

## Troubleshooting

**Every request returns `403 {"error":"Forbidden"}`.** The origin check. Set
`MCPJAM_ALLOWED_HOSTS` to your exact hostname (no scheme, no path) and
`ALLOWED_ORIGINS` to `https://your-hostname`, then redeploy. This is the usual
cause after adding a custom domain.

**"Session token required" / "Invalid session token".** The link lost its
fragment. Copy the whole link from the deploy log, including `#token=…`, or set
your own `MCPJAM_SESSION_TOKEN` and redeploy.

**A plain URL shows instructions instead of the app.** By design — open it
through the access link so it can sign the browser in.

**Widget rendering says the browser is unavailable.** Chromium is baked into the
image, but a sandbox can deny it some syscalls. Core inspection, logging, OAuth
and the playground are unaffected.

**The health check fails on the first deploy.** Raise
`RAILWAY_HEALTHCHECK_TIMEOUT_SEC` further (the template sets `600`) and
redeploy; the first container start initialises the Inspector's services before
it serves.

## About this template

This is a **community deployment packaging**, not the upstream product. It
vendors the [MCPJam Inspector source](https://github.com/MCPJam/inspector) at a
pinned commit and changes no application code, no Dockerfile and no dependency.

*   Apache License 2.0, © MCPJam. Not endorsed by, affiliated with, or supported
    by MCPJam.
*   Full deployment guide: [github.com/BURNI80/mcpjam-inspector-railway](https://github.com/BURNI80/mcpjam-inspector-railway)
