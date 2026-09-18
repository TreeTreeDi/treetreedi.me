# Home

Home is the landing page at `/`. A visitor sees TreeTreeDi's name, a short welcome, a GitHub link, and the site header that reaches Blog, Projects, and Life.

## Sub-features

- `home-open` renders the landing copy and site identity.
- `home-logo` returns to `/` from another route via the logo link.
- `home-nav` exposes Blog, Projects, and Life in the header at desktop width.

## How to get to it (user POV)

- Open the site root `/`.
- Choose the TreeTreeDi logo in the header from any page.

## Driving it with control-treetreedi

Preconditions:

- TreeTreeDi is healthy at the URL from `control-treetreedi doctor`.
- Viewport is the helper default `1280×800`.

- **Open home.** Go to the root. Run `control-treetreedi browser navigate --path /`. The document title is `TreeTreeDi` and a heading named `TreeTreeDi` is visible.
- **Read welcome.** The page shows `Hey! I'm TreeTreeDi.` and `Welcome to my personal website.`
- **Identity.** The footer contains `CC BY-NC-SA 4.0 2025-PRESENT © TreeTreeDi`.
- **Header nav.** Links named `Blog`, `Projects`, and `Life` are visible. GitHub and RSS are present at this width.
- **Logo return.** Open Blog, then choose the logo. Run `control-treetreedi browser click --role link --name "Blog"` and `control-treetreedi browser click --role link --name "TreeTreeDi Logo"`. The URL is `/` and the `TreeTreeDi` heading returns.
- **Proof.** Capture the landing state. Run `control-treetreedi browser snapshot --aria --path artifacts/home/home.aria.txt` and `control-treetreedi browser screenshot --path artifacts/home/home.png`. Both artifacts show TreeTreeDi identity and the three primary nav names.

## Gotchas

- Below the `md` breakpoint, Blog / Projects / Life become icon-only. Their accessible names remain the `title` values, but a screenshot will not show the words. Stay at `1280×800`.
- The logo link has no visible text. Use `TreeTreeDi Logo`, not `Home`.
- Theme toggle is an `<a title="Toggle Color Scheme">` with no text. Do not treat a missing "Dark mode" name as a broken control.
- Decorative plum/dots canvas can animate behind the prose. Identity is the heading and footer, not the canvas.
