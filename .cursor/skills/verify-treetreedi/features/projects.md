# Projects

Projects is the portfolio grid at `/projects`. A visitor opens it from the header, reads grouped cards, and can follow a project or GitHub link.

## Sub-features

- `projects-open` shows the Projects heading and intro line.
- `projects-cards` lists the current cards (`CloudQuery`, `StreamFusion`, `Multi-Mode Selection`, `markdown-flow`).
- `projects-github` exposes the page-level GitHub button.

## How to get to it (user POV)

- Choose `Projects` in the header.
- Open `/projects` directly.

## Driving it with control-treetreedi

Preconditions:

- TreeTreeDi is healthy at the URL from `control-treetreedi doctor`.
- Viewport is `1280×800`.
- `control-treetreedi http --path /projects --expect 200` succeeds.

- **Header entry.** From `/`, choose `Projects`. Run `control-treetreedi browser navigate --path /` and `control-treetreedi browser click --role link --name "Projects"`. The URL is `/projects` and the heading reads `Projects`.
- **Intro.** The page shows `Projects that I created or maintaining.`
- **Cards.** Links named `CloudQuery`, `StreamFusion`, `Multi-Mode Selection`, and `markdown-flow` are visible. `CloudQuery` is in the Current Focus group.
- **Page GitHub.** A `GitHub` link on the page points at `https://github.com/treetreedi`. Do not follow it for an in-app proof.
- **Proof.** Capture the grid. Run `control-treetreedi browser snapshot --aria --path artifacts/projects/projects.aria.txt` and `control-treetreedi browser screenshot --path artifacts/projects/projects.png`. Both show the `Projects` heading and at least `CloudQuery` and `StreamFusion`.

## Gotchas

- Project cards are `<a target="_blank">`. Clicking a card leaves the site. Prove presence from the snapshot; do not treat an outbound tab as an in-app state change.
- The header also has a `GitHub` link. After opening `/projects`, prefer the card names or the heading when asserting location.
- Group labels `Current Focus` and `Projects` are decorative large strokes (`pointer-events: none`). Do not click them.
