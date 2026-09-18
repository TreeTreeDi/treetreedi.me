# Photos

Photos is the Life gallery at `/photos`. A visitor opens Life from the header, sees a grid of photographs, and can switch cover/contain layout.

## Sub-features

- `photos-open` shows the photo grid and thank-you line.
- `photos-view` toggles cover vs contain with `Switch view`.
- `photos-preview` opens a photo overlay from a grid image (optional; keyboard Escape closes it).

## How to get to it (user POV)

- Choose `Life` in the header.
- Open `/photos` directly.

## Driving it with control-treetreedi

Preconditions:

- TreeTreeDi is healthy at the URL from `control-treetreedi doctor`.
- Viewport is `1280×800`.
- `control-treetreedi http --path /photos --expect 200` succeeds.

- **Header entry.** From `/`, choose `Life`. Run `control-treetreedi browser navigate --path /` and `control-treetreedi browser click --role link --name "Life"`. The URL is `/photos`.
- **Gallery.** One or more `img` elements are visible in the photos grid. The page shows `Thank you for being interested in my photos.`
- **No page title heading.** There is no `h1`. Empty `display` hides the wrapper heading. Do not fail the page for a missing `Photos` heading.
- **Switch view.** Choose `Switch view`. Run `control-treetreedi browser click --role button --name "Switch view"`. The button remains and the grid is still populated (layout class changes; images stay).
- **Proof.** Capture the gallery. Run `control-treetreedi browser snapshot --aria --path artifacts/photos/photos.aria.txt` and `control-treetreedi browser screenshot --path artifacts/photos/photos.png`. The screenshot shows a photo grid and the thank-you line. The snapshot includes at least one image.

## Gotchas

- Frontmatter title is still `Photos - Anthony Fu`. Document title may not say TreeTreeDi. Use the header logo and footer for identity.
- Images load from Vite `?url` modules under `photos/`. First paint can be empty while large JPEGs load — wait for `img` elements, not a fixed sleep.
- Overlay preview is a click on an `img` inside `.photos`. There is no dialog name. Escape or a backdrop click closes it. Skip overlay if the grid itself already proves the page.
- `Switch view` is icon-only. Use the `title` name, not a visible "Cover" label.
