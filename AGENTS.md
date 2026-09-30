# AGENTS.md — instructions for coding agents in this repository

## What this repository is

`mcpjam-inspector-railway` is a **deployment packaging**, not a fork under
active development. It vendors the [MCPJam Inspector](https://github.com/MCPJam/inspector)
source tree at one pinned commit (recorded in `UPSTREAM_COMMIT`) so that the
Railway template builds a reproducible artifact, using upstream's own
`mcpjam-inspector/Dockerfile` and `railway.json` unmodified.

Upstream's own agent instructions are preserved verbatim as
`UPSTREAM_AGENTS.md`. They document the vendored codebase and are the right
reference when you need to understand *that* code — but they are upstream's
brief to contributors to their monorepo, not standing orders here.

## The rule that matters

**Do not modify vendored application code, `mcpjam-inspector/Dockerfile`,
`railway.json`, `package.json`, `package-lock.json` or any workspace's
`package.json`.** A change there silently forks the product this template exists
to deploy, and the README's "changes no application code" claim stops being true.

`NOTICE` lists the exact set of differences from upstream. If you add one,
update `NOTICE` in the same commit. If you remove one, remove it from `NOTICE`
and say so in the commit message.

## What *is* ours to change

*   `README.md` — the deployment guide, and the text published as the template's
    overview in the Railway Marketplace.
*   `TEMPLATE.md` — the Marketplace overview itself. Railway caps it at 10,000
    characters and strips `<...>` (it reads them as HTML), so it uses
    `UPPERCASE-PLACEHOLDERs` and explicit `[text](url)` links. Re-publish with
    `railway templates update 768be898-4b4c-473f-97cc-bb8306e55da0
    --category "AI/ML" --description "..." --readme-file TEMPLATE.md` after any
    change; the published copy is a snapshot, not a symlink to the file.
*   `.dockerignore` — safe only to exclude paths no Dockerfile stage copies. See
    the comments in the file; it explains the one exclusion that looks harmless
    and is not.
*   `NOTICE`, `UPSTREAM_COMMIT`, `assets/`.

## Bumping to a new upstream commit

1.  In a scratch clone of `MCPJam/inspector`, check out the target commit and
    compare it with the pin in `UPSTREAM_COMMIT`.
2.  Re-vendor the tracked tree, keeping this repository's files and deletions:

    ```bash
    git -C /path/to/inspector archive <sha> | tar -x -C /path/to/this/repo
    ```

    Re-apply the deletions listed in `NOTICE` afterwards.
3.  Rewrite `UPSTREAM_COMMIT`.
4.  Check whether upstream added a new workspace to the root `package.json` or a
    new `COPY` to `mcpjam-inspector/Dockerfile`. If it did, `.dockerignore` needs
    review again — that is the step that silently breaks builds.
5.  Commit, push, and watch the Railway build log to the end. A green
    deployment is the only verification this packaging has.

## Verifying a change

There is no test suite here and there should not be one — the vendored
application has its own, upstream. Verification is a Railway deployment:

```bash
railway redeploy --service "MCPJam Inspector"
railway logs --service "MCPJam Inspector" --build
curl -sS https://<service>.up.railway.app/health
```

The build compiles a large TypeScript monorepo and installs Chromium, so budget
15–30 minutes for a cold build. A redeploy of an unchanged commit reuses the
cached layers.

## Deploying the template

The live template is managed from the Railway project "MCPJam Inspector"
(service `MCPJam Inspector`). Publishing is a dashboard action; see
`README.md` for the variable contract the template depends on. **Changing a
variable that the template sets requires updating the template too**, or the
published template keeps deploying the old contract.
