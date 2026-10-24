# CLAUDE.md

htmx (v2.0.x, `htmx.org`) is a single-file, dependency-free browser library: `hx-*` HTML attributes drive AJAX requests, DOM swaps, history, and events.

## Architecture

- `src/htmx.js` (~5.4k lines) is the entire library and the only source file to edit. The browser global is `htmx`.
- `dist/` is **generated** from `src/htmx.js` by `npm run dist` (copy + uglify + AMD/CJS/ESM wrappers + `.d.ts`). Never hand-edit it, and don't commit dist changes in feature/test work.
- `test/attributes/hx-*.js` hold per-attribute tests; `test/core/*.js` hold engine tests (`ajax`, `events`, `config`, `regressions`, ...). Only these two dirs are run by `web-test-runner.config.mjs` (globs `test/attributes/**/*.js`, `test/core/**/*.js`).
- `test/util/util.js` provides the shared test globals (`make`, `makeServer`, `byId`, `clearWorkArea`, `getParameters`, `getWorkArea`). The runner HTML in the config loads chai, chai-dom, sinon, mock-socket, `src/htmx.js`, and `util.js` itself; tests have no imports.
- Not run by automation: `test/manual/` (hand-opened HTML), `test/scratch/`, `test/lib/_hyperscript.js` (vendored), `www/` (docs site), `editors/` (generated web-types).

## Commands (run from repo root)

```bash
npm ci                                  # install; lockfile is committed
npm run lint                            # eslint src/htmx.js test/attributes test/core test/util scripts/*.mjs
npm run format                          # eslint --fix over the same paths (rewrites files)
npm run types-check                     # tsc --checkJs on src/htmx.js (JSDoc types)
npm run test:chrome                     # full suite, headless Chromium (auto-installs playwright browser)
npm run test:chrome test/core/ajax.js   # one file (path is forwarded to web-test-runner)
npm test                                # lint + types-check + test:chrome (what CI runs; Node 20)
npm run test:all                        # also Firefox + WebKit; slow, run only if asked
npm run dist                            # regenerates dist/ — only for releases
npx serve                               # dev server; open http://localhost:3000/test/ for the Mocha browser runner (serve isn't a devDependency; npx fetches it)
```

A focused file run plus lint does not cover `types-check`; finish any change with the full `npm test`.

The baseline at the time of writing: 868 pass, 0 fail, 3 skipped. Coverage output goes to the gitignored `coverage/`.

## Test conventions

- Mocha `describe`/`it` with `function()` (not arrows — `this.server` is used) and chai **`should`** style (`btn.innerHTML.should.equal('x')`; `should.equal(a, b)` for undefined/null). No semicolons, 2-space indent, `standard` style, and `space-before-function-paren: never` (`function() {}`).
- Standard scaffold: `beforeEach` sets `this.server = makeServer()` (a few files, e.g. `hx-target.js`, use raw `sinon.fakeServer.create()`; copy the file you're editing) and calls `clearWorkArea()`; `afterEach` calls `this.server.restore()` and `clearWorkArea()`.
- Per test: `this.server.respondWith('GET', '/test', 'body')` → `var el = make('<div hx-get="/test">…</div>')` → `el.click()` → `this.server.respond()` → assert. Requests are synchronous via the sinon fake server (no real network). Tests that need a delay (`hx-trigger` delays, settling) take Mocha's `done` callback and use `setTimeout`; the suite has no `async`/`await`.
- Tests that mutate `htmx.config` must restore it (save a copy in `beforeEach`/before the test, reassign in `afterEach`). Handlers registered with `htmx.on` must be removed with `htmx.off`.
- Use unique ids and URLs inside a test since all tests share one `#work-area`.
- Add tests to the existing file for the attribute/feature. A new test file would also need adding to the manual `<script>` list in `test/index.html` (browser-mode runner), which the automation doesn't use.

## Guardrails

- `npm run build` does not exist; the build script is `npm run dist`. `npm run ws-tests` points at a missing `test/ws-sse/`; ignore it.
- `it.only`/`describe.only` only produce a lint *warning* (`mocha/no-exclusive-tests`, exit code stays 0) and would silently shrink the suite. Grep for `.only` before finishing.
- `test:*` scripts run `playwright install`, which downloads browsers; this needs network on first run.
- Don't touch `test/lib/`, `dist/ext/` (deprecated copies), or `www/content/` for code tasks.
- Upstream takes code PRs against `dev`, not `master` (only docs-only `www/` PRs go to `master`), and asks for `npm run format` first. Keep changes small and test-backed; refactors with no behavior change are rejected.
