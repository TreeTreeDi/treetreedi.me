---
name: verify-treetreedi
description: Drive TreeTreeDi's public Vite site (treetreedi.me) as a user — launch an isolated local server, doctor the instance, click Blog/Projects/Life, and capture ARIA/screenshot proof. Use when verifying pages, nav, or content on this Vue + Vite-SSG site.
---

# Verify TreeTreeDi

Primary surface is the public web UI. Local verification binds a disposable Vite process (prefer `http://127.0.0.1:3333`). Production is `https://treetreedi.me`. This is not a CLI product.

Repo scripts: `packageManager` is `pnpm@10.19.0`. `pnpm dev` is `vite --port 3333 --open`. Verification never uses `--open` and never attaches to a vite the user already started.

There is no Playwright/Cypress harness in the app. Drive through `scripts/control-treetreedi`.

Read `features/README.md` before a run. Drive the feature file that matches the claim. One convenient entry point does not cover the rest of the map.

## Launch

From the repo root:

```bash
pnpm install
.cursor/skills/verify-treetreedi/scripts/control-treetreedi launch
```

What launch does:

- Refuses if `.run/instance.json` already names a live vite this run started.
- Prefers `127.0.0.1:3333` only when that bind is free. Otherwise it takes the next free port in `3334–3399` and records it.
- Starts `pnpm exec vite --host 127.0.0.1 --port <port> --strictPort` with `BROWSER=none` (no `--open`).
- Writes `.cursor/skills/verify-treetreedi/.run/instance.json` (`pid`, `url`, `port`, `logPath`).
- Waits until `GET <url>/` returns HTTP 2xx/3xx (up to 90s). First compile can be slow.

Ready signal: command exits 0 and prints `"status": "ready"` plus the URL. Log is `.run/vite.log`.

Do not run `pnpm dev` for verification. Do not reuse a foreign listener on 3333.

Teardown is `control-treetreedi cleanup` (see Cleanup).

## Doctor

Read-only. Run before driving whenever anything looks off:

```bash
.cursor/skills/verify-treetreedi/scripts/control-treetreedi doctor
```

Pass means all of:

- `.run/instance.json` exists
- the recorded PID is alive and its process tree still contains `vite`
- the recorded port is not owned by a foreign PID
- `GET <url>/` is HTTP 200
- the HTML contains `TreeTreeDi`

Fail (exit 1) if any check fails. Do not drive that instance. Launch a new one after cleanup, or stop — never attach to someone else's vite.

Optional route probe:

```bash
.cursor/skills/verify-treetreedi/scripts/control-treetreedi http --path /posts --expect 200
```

## Drive

Harness: Playwright Chrome (`channel: 'chrome'`) behind `control-treetreedi browser`. Viewport is `1280×800` so the desktop nav shows text (`Blog`, `Projects`, `Life`) instead of icon-only mobile labels.

Stable handles from this repo:

| Control  | Role / name                    | Notes                                                        |
| -------- | ------------------------------ | ------------------------------------------------------------ |
| Home     | `link` / `TreeTreeDi Logo`     | Logo `RouterLink` to `/`; SVG `aria-label="TreeTreeDi Logo"` |
| Blog     | `link` / `Blog`                | Nav `title="Blog"` → `/posts`                                |
| Projects | `link` / `Projects`            | Nav `title="Projects"` → `/projects`                         |
| Life     | `link` / `Life`                | Nav `title="Life"` → `/photos`                               |
| Theme    | `link` / `Toggle Color Scheme` | `<a title="Toggle Color Scheme">`                            |
| RSS      | `link` / `RSS`                 | `/feed.xml` (dev may 404; not a page proof)                  |
| GitHub   | `link` / `GitHub`              | External `https://github.com/treetreedi`                     |

Home identity: heading `TreeTreeDi`, body `Hey! I'm TreeTreeDi.`, footer `© TreeTreeDi`.

Blog identity: heading `Blog`, then dated post links from `pages/posts/*.md`. Newest title at interview: `Claude Code 记忆系统工程指南：CLAUDE.md 与 Auto Memory` → `/posts/claude-code-memory-system`.

Projects identity: heading `Projects`, copy `Projects that I created or maintaining.`, cards `CloudQuery`, `StreamFusion`, `Multi-Mode Selection`, `markdown-flow`.

Life identity: photo grid (`img` alts from `photos/data.ts`), button `Switch view`, line `Thank you for being interested in my photos.` No `h1` — `display` is an empty string.

Talks is not in the primary nav. Reach it with `browser navigate --path /talks`.

Recipe shape (posts example):

```bash
CTL=.cursor/skills/verify-treetreedi/scripts/control-treetreedi
$CTL browser navigate --path /
$CTL browser click --role link --name "Blog"
$CTL browser snapshot --aria --path artifacts/prove-posts/blog-list.aria.txt
$CTL browser click --role link --name "Claude Code 记忆系统工程指南：CLAUDE.md 与 Auto Memory" --exact false
$CTL browser snapshot --aria --path artifacts/prove-posts/open-post.aria.txt
$CTL browser screenshot --path artifacts/prove-posts/open-post.png
```

Prefer `getByRole` names over CSS or coordinates. Exact name match is the default (`--exact false` to loosen).

## Evidence

Write proof under `.cursor/skills/verify-treetreedi/artifacts/<feature>/`. That directory is gitignored except the checked-in `prove-posts` text trail.

Standards:

- Exercise the real user path (nav click or typed URL). Do not set Vue router state from the console.
- Capture the action and the resulting state. A final screenshot alone is not proof.
- Pair every UI claim with an ARIA snapshot and a screenshot that show site identity (`TreeTreeDi` logo/title/footer).
- Record HTTP status for the route (`control-treetreedi http --path …`).
- Capture console with `browser console` when the page should be quiet. Note third-party script noise (`platform.twitter.com/widgets.js` is in `index.html`).
- Record the feature ID and entry point next to the files (a `RUN.md` in the artifact folder).
- Do not mock content. This site serves Markdown from `pages/`.

Example:

```bash
CTL=.cursor/skills/verify-treetreedi/scripts/control-treetreedi
$CTL http --path /posts --expect 200
$CTL browser snapshot --aria --path artifacts/prove-posts/blog-list.aria.txt
$CTL browser screenshot --path artifacts/prove-posts/blog-list.png
$CTL browser console --path artifacts/prove-posts/console.json
```

## Cleanup

```bash
.cursor/skills/verify-treetreedi/scripts/control-treetreedi cleanup
```

Kills only the PID recorded in `.run/instance.json` and its descendants. Removes `instance.json`, `.run/browser-state.json`, and the disposable Chrome profile. Leaves `artifacts/` and `.run/vite.log` in place. Each `browser` command restores the last URL from `browser-state.json` so click → snapshot stays on the same page.

Never `pkill vite` or kill by process name. After a failed iteration, run cleanup before the next launch so ports and PIDs do not leak.

After cleanup, confirm the proof files still exist at the paths above.

## Helpers

Executable: `.cursor/skills/verify-treetreedi/scripts/control-treetreedi`

| Invocation                                                      | Result                                        |
| --------------------------------------------------------------- | --------------------------------------------- |
| `control-treetreedi launch`                                     | Start an isolated vite; print URL/PID         |
| `control-treetreedi launch --port 3340`                         | Bind an explicit free port                    |
| `control-treetreedi doctor`                                     | Exit 0 only if this run's instance is healthy |
| `control-treetreedi http --path /photos --expect 200`           | Route status + identity flag                  |
| `control-treetreedi browser navigate --path /projects`          | Open a path in the persistent Chrome profile  |
| `control-treetreedi browser click --role link --name "Life"`    | Click by role and accessible name             |
| `control-treetreedi browser snapshot --aria --path artifacts/…` | Write an ARIA snapshot                        |
| `control-treetreedi browser screenshot --path artifacts/…`      | Full-page PNG                                 |
| `control-treetreedi browser console --path artifacts/…`         | Console + pageerror dump                      |
| `control-treetreedi cleanup`                                    | Stop this run's vite; keep evidence           |

First `browser` command installs `playwright-core` into `.run/playwright/` (gitignored) and uses the machine Chrome. No Playwright dependency is added to the app `package.json`.
