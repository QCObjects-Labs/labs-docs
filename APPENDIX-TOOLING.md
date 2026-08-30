# Appendix: tooling & verification notes

Practical notes from running the labs, so future labs can skip the fumbling.

## Verifying QCObjects rendering in a headless browser

Light-DOM snapshots (`--dump-dom`) **cannot see** component content because it
renders into a **native shadow root** (see FINDINGS.md §3). Use CDP instead:

- Launch Chrome headless with `--remote-debugging-port` and a user-data-dir.
- Connect via `webSocketDebuggerUrl` (Node's built-in `WebSocket` is enough).
- `Runtime.evaluate` →
  `document.querySelector(".shadowHost").shadowRoot.innerHTML` to confirm the
  template text landed.
- `Runtime.evaluate` on `window.logger.debugEnabled = true` before the page
  boots, or read `Log`/console events, to capture the framework trace.
- Check the `.tpl.html` fetch status in the Network domain — a 404 there is the
  blank-component smoking gun.

## Serving

The stock `qcobjects-http-server` and any static server are interchangeable for
the external-template pattern — QCObjects just XHRs the file from the document
root. We used `python3 -m http.server 8080 --directory browser`.

## Long-running servers in the sandbox

- Start with `setsid ... &` + `disown` so the process survives the shell.
- Avoid `pkill -f` patterns that match the grep itself — use
  `ss -ltnp` to find the PID and kill it directly, or bracket patterns like
  `8801]`.

## Build recipe

- Use `npx -y esbuild@0.17.19 ...` when there's no `node_modules` yet.
- Copy `src/templates/` (and `src/index.html`) into `browser/` as an assets
  step — don't rely on the bundler for `.tpl.html` files.

## Git hygiene for these repos

- macOS AppleDouble files (`._*`) appear in copies; add `._*` to every
  `.gitignore` and remove strays before committing.
- `.gitignore` per repo: `node_modules/`, `browser/` (build output), `dist/`,
  `._*`, `.DS_Store`.
- If a stray commit grabs `._*` files, unstage with `git reset`, add the ignore
  rule, `git add -A`, then `git commit --amend`.

## Creating org repos with gh

```bash
cd <app-dir>
git init -b main
git add -A && git commit -m "..."
gh repo create QCObjects-Labs/<name> --public --source=. --remote=origin --push
```

- `--source=.` requires the directory to already be a git repository.
- Use the positional `org/repo` form; older `gh` versions don't accept
  `--owner`.