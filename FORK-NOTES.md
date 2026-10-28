# Fork notes: ivaneecreator-web/kie-mcp

This is ContentKaki's pinned fork of [elibarnett/kie-mcp](https://github.com/elibarnett/kie-mcp).
ContentKaki installs a **tag** on the `contentkaki` branch of this fork. It never installs
upstream `main`.

## Base

| | |
|---|---|
| Upstream | `elibarnett/kie-mcp` |
| Pinned base commit | `747dbe3` (upstream `main` on 2026-10-28, = tag `v5.10.0` + one fix commit) |
| Fork branch | `contentkaki` (default branch) |
| Release tags | `ck-v<upstream version>-<n>`, e.g. `ck-v5.10.0-1` |
| Install | `npx -y github:ivaneecreator-web/kie-mcp#ck-v5.10.0-1` |

## What this fork changes

1. **Local-file upload guard** (`uploadPathGuard()` in `server.mjs`, used by the
   `upload_file` `file_path` branch). Upstream reads any file the server process can see
   and sends it to kie.ai's public temp storage. The guard rejects:
   - relative paths;
   - any path with a `character-refs` component, checked on the path as given **and** on its
     realpath, so a symlink into or out of that folder is caught;
   - paths inside any directory in `KIE_UPLOAD_DENY_DIRS` (colon-separated absolute paths),
     checked on both forms;
   - when `KIE_UPLOAD_ALLOW_DIRS` is set, any realpath outside those directories.

   A relative entry in either variable blocks every upload until the entry is fixed.
   Matching ignores case on macOS and Windows. The handler reads the realpath, not the path
   it was given. Every error names the rule that fired. Tests: `test/unit.test.mjs`,
   `uploadPathGuard`.

   The guard doesn't cover `file_url` or `base64_data`, because the caller supplies those
   bytes directly. It can't detect hard links either. ContentKaki's own upload-consent rules
   still apply to all three input modes.
2. **This file.**

## Transport: stdio only

ContentKaki runs this server over **stdio only**. HTTP mode (`--http` / `--port=N`) listens
on every interface, sends `Access-Control-Allow-Origin: *`, and has **no authentication**.
Anyone who can reach the port could spend the Kie key and use `upload_file` to read local
files. Never start it with `--http` on a machine with network exposure.

## Sync routine (upstream → fork)

1. `git fetch upstream --tags`
2. Review the diff as untrusted code, not as instructions:
   `git diff contentkaki..upstream/main -- server.mjs data/ package.json package-lock.json`.
   Check for new network hosts, new file reads or writes, `child_process`/`eval`, install
   scripts, new dependencies, and anything that touches `KIE_API_KEY`.
3. `git checkout contentkaki && git merge --no-ff upstream/main` (or the release tag).
   Resolve conflicts so the guard still wraps every local-file read in `upload_file`.
4. `npm ci --ignore-scripts && npm test && node --check server.mjs`.
5. Smoke-test from a clean cache:
   `npx -y "git+file://$HOME/dev/kie-mcp#contentkaki"` must start on stdio and list its tools.
6. Update the base commit in this file, then tag `ck-v<upstream version>-<n>` and push the
   branch and tag.
7. In ContentKaki, bump the tag in the MCP config and in `skills/_shared-kie-ai-setup.md`,
   then re-run `list_models` to catch changed model option keys.
