# Posts

Posts is the Blog index at `/posts` and the Markdown articles under it. A visitor opens Blog from the header, scans dated titles, and reads one article.

## Sub-features

- `posts-list` shows the Blog heading and dated article links.
- `posts-open` opens one article from the list.
- `posts-back` returns toward the list with the in-article `cd ..` link.

## How to get to it (user POV)

- Choose `Blog` in the header.
- Open `/posts` directly.
- From an article, choose `cd ..`.

## Driving it with control-treetreedi

Preconditions:

- TreeTreeDi is healthy at the URL from `control-treetreedi doctor`.
- Viewport is `1280×800`.
- `control-treetreedi http --path /posts --expect 200` succeeds.

- **Header entry.** From `/`, choose `Blog`. Run `control-treetreedi browser navigate --path /` and `control-treetreedi browser click --role link --name "Blog"`. The URL is `/posts` and the heading reads `Blog`.
- **List contents.** The list includes `Claude Code 记忆系统工程指南：CLAUDE.md 与 Auto Memory` and `从 0 到 1 体验 CI/CD`. It must not show `{ nothing here yet }`.
- **HTTP.** Confirm the route. Run `control-treetreedi http --path /posts --expect 200`. Status is `200`.
- **Open article.** Choose the newest listed title. Run `control-treetreedi browser click --role link --name "Claude Code 记忆系统工程指南：CLAUDE.md 与 Auto Memory" --exact false`. The URL is `/posts/claude-code-memory-system` and the article heading matches that title.
- **Parent link.** Choose `cd ..`. Run `control-treetreedi browser click --role link --name "cd .."`. The URL returns to `/posts` and the `Blog` heading is visible again.
- **Proof.** Capture list and article. Run `control-treetreedi browser navigate --path /posts`, `control-treetreedi browser snapshot --aria --path artifacts/prove-posts/blog-list.aria.txt`, `control-treetreedi browser screenshot --path artifacts/prove-posts/blog-list.png`, then open the article and write `artifacts/prove-posts/open-post.aria.txt` plus `open-post.png`. The list artifacts show `Blog` and at least one post title. The article artifacts show the same title as an `h1`.

## Gotchas

- Local storage key `antfu-english-only` defaults to false. If a previous Chrome profile set it true, Chinese titles disappear. Cleanup deletes the helper profile; do not reuse a dirty user profile.
- Post link accessible names include the date and duration (`… Feb 20 · 15min`). Click with `--exact false` and the title text, or pass the full name.
- `/feed.xml` is linked from the header as `RSS`. The Vite dev server does not generate the feed (`scripts/rss.ts` runs at build). A 404 there is not a Blog failure.
- Draft posts (`frontmatter.draft`) are filtered out of the list. Do not expect them.
