# Prove run — posts

- Feature: `posts` (`posts-list`, `posts-open`)
- Entry: header `Blog` from `/`, then the newest article link
- Instance: `http://127.0.0.1:3333` (preferred port was free)
- Host: Linux cloud VM, Chrome via `playwright-core` `channel: 'chrome'`
- Date: 2026-09-18

## Sequence

1. `control-treetreedi launch` → ready on `127.0.0.1:3333`
2. `control-treetreedi doctor` → `ready: true`, HTML identity `TreeTreeDi`
3. `http --path /` and `http --path /posts` → 200
4. `browser navigate --path /` → title `TreeTreeDi`
5. `browser click --role link --name "Blog"` → `/posts`, title `Blog`
6. ARIA + screenshot of the list
7. `browser click --role link --name "Claude Code 记忆系统工程指南：CLAUDE.md 与 Auto Memory" --exact false` → `/posts/claude-code-memory-system`
8. ARIA + screenshot of the article
9. `browser console` → no page errors (vite debug connect only)
10. `control-treetreedi cleanup` → vite PID gone; these files kept

## Artifacts

- `doctor.json`
- `http.json`
- `blog-list.aria.txt` / `blog-list.png`
- `open-post.aria.txt` / `open-post.png`
- `console.json`

Screenshots stay gitignored. Text files in this folder are the committed trail.
