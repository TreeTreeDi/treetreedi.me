# Talks

Talks is the talk list at `/talks`. It is not in the primary header. A visitor who knows the URL sees a Talks heading, a Blog/Talks/Podcasts/Streams/Notes subnav, and talk titles with conference rows.

## Sub-features

- `talks-open` shows the Talks heading and subnav.
- `talks-list` lists talk titles such as `Introducing Vite DevTools`.
- `talks-speaker` reaches `/giving-talks` from the `Speaker Info` link.

## How to get to it (user POV)

- Open `/talks` directly.
- From Talks, choose `Speaker Info` to open `/giving-talks`.
- From Talks subnav, choose `Blog` to reach `/posts`.

## Driving it with control-treetreedi

Preconditions:

- TreeTreeDi is healthy at the URL from `control-treetreedi doctor`.
- Viewport is `1280×800`.
- `control-treetreedi http --path /talks --expect 200` succeeds.

- **Direct entry.** Open Talks by URL. Run `control-treetreedi browser navigate --path /talks`. The URL is `/talks` and a heading named `Talks` is visible.
- **Subnav.** Links named `Blog`, `Talks`, `Podcasts`, `Streams`, and `Notes` are visible. The `Talks` subnav item is the current one.
- **List.** A heading named `Introducing Vite DevTools` is visible, with a `VueConf China` or `ViteConf` conference link nearby.
- **Speaker page.** Choose `Speaker Info`. Run `control-treetreedi browser click --role link --name "Speaker Info"`. The URL is `/giving-talks`.
- **Proof.** Capture the talks index. Run `control-treetreedi browser navigate --path /talks`, `control-treetreedi browser snapshot --aria --path artifacts/talks/talks.aria.txt`, and `control-treetreedi browser screenshot --path artifacts/talks/talks.png`. Artifacts show the `Talks` heading and at least one talk title.

## Gotchas

- Talks is not in the primary header. A header-only walk will never reach it. Do not mark Talks verified by visiting Blog.
- `/giving-talks` still reads as Anthony Fu speaker copy. That is the current page content. Do not rewrite it during verification. Assert the route and the `Giving Talks` heading only if you are proving that leftover page, and say so.
- `English Only` on the subnav hides non-English presentations when enabled. Start from a clean helper Chrome profile.
- Conference `Watch` / `Slides` / `PDF` links are external. Prove they exist; do not follow them as in-app navigation.
