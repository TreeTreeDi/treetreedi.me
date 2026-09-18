# TreeTreeDi verification map

This directory is the maintained source for verifying the user-facing behavior of TreeTreeDi's site. Read the index before driving the app, then use the matching feature file as the recipe.

## Baseline preconditions

- Launch with `.cursor/skills/verify-treetreedi/scripts/control-treetreedi launch` from the repo root after `pnpm install`.
- Run `control-treetreedi doctor` and require `ready: true`, the printed URL, and HTML identity `TreeTreeDi`.
- Drive at viewport `1280×800` (the helper default) so the header shows `Blog`, `Projects`, and `Life` as text.
- Never drive an instance that was not started by this verification run.
- `pnpm build` / `vite-ssg` needs the `static` degit step and is out of scope for these recipes. Use the Vite dev server.

## Driving conventions

- Start every recipe from `/` unless the feature file says otherwise.
- Prefer ARIA roles and accessible names over CSS selectors or DOM position.
- Treat every command as literal. Keep quoted names and flags unchanged.
- Run browser and HTTP actions through `control-treetreedi`.
- Do not remove proof artifacts during cleanup.

## Proof and skip reporting

- Capture the user action and the resulting state, not only the final screen.
- UI proof includes an ARIA snapshot and a screenshot with TreeTreeDi identity visible (logo, title, or footer `© TreeTreeDi`).
- Route proof includes `control-treetreedi http --path <path> --expect 200`.
- Record the feature ID and entry point used with every artifact.
- Report an unreachable path with the attempted command and the unmet precondition.
- Do not report a skipped entry point as verified through a different path.

## Feature entry contract

Each feature file starts with an H1 title and one paragraph describing the user-visible behavior. It then uses exactly four H2 sections in this order.

1. `Sub-features` lists short IDs with one line for each behavior.
2. `How to get to it (user POV)` lists every user entry point.
3. `Driving it with control-treetreedi` starts with `Preconditions:` and uses labeled bullets that pair each user action with an exact command and observable result.
4. `Gotchas` lists traps that can waste or invalidate a verification run.

Keep implementation details out of the map. Name only user paths, stable handles, required state, commands, and observable proof.

## Features

- [Home](./home.md) covers the landing page, logo return, and header identity.
- [Posts](./posts.md) covers Blog list and opening one article.
- [Projects](./projects.md) covers the project cards and outbound GitHub links.
- [Photos](./photos.md) covers the Life gallery and view toggle.
- [Talks](./talks.md) covers the talks index (URL-only; not in the primary nav).
