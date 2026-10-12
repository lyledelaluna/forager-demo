# Forager interactive walkthrough

A self-contained, static demo of Forager, the supplier-document intake pipeline. It needs no backend,
API keys, Gmail, Dropbox, or LLM. The folder contains:

| Path | What it is |
|---|---|
| `index.html` | The landing page and guided-tour shell. It's a single file with inline CSS and JS and makes no external requests. |
| `app/` | The real React app, built in demo mode by `npm run build:demo` in `frontend/`. Every API call is answered in the browser from fictional fixtures. |

The tour loads `app/?embedded=1` in a same-origin iframe. It reads that iframe's DOM to place the
spotlight and to press buttons when you click **Next**, so the page **must be served over http(s)**.
Opening `index.html` from disk (`file://`) won't work, because browsers block that cross-frame access.
The page detects that case and shows instructions.

## Build and deploy flow

From the repo root:

```sh
cd frontend
npm install
npm run build:demo    # writes ../demo-site/app/
```

Then deploy the `demo-site/` folder (see Deploy below). There is no PDF step: the sample PDFs are
rendered in the browser on demand by `frontend/src/demo/pdfRender.js`, so nothing is generated or
copied into `public/`. `index.html` has no build step; edit it in place.

## Configuration

`index.html` has one constant near the top of its script:

```js
const CONTACT = { label: '', href: '' };
```

Leave it empty and the end screen shows no call to action. Set both fields (for example
`{ label: 'Talk to the team', href: 'mailto:you@example.com' }`) and a primary button with that label
appears on the end screen. Only `http(s):` and `mailto:` links are rendered. There is no default, so
no contact address or URL ships with the demo.

The tour passes `?theme=light|dark` to the app from the visitor's `prefers-color-scheme`, so the shell
and the app match.

## Preview locally

Use either of these:

```sh
npx serve demo-site                       # then open the printed URL
# or
cd demo-site && python3 -m http.server 8000   # then open http://localhost:8000/
```

Add `#tour` to the URL to skip the landing page and start the walkthrough straight away.

## Deploy

The folder is fully static and uses relative paths only, so it works at a domain root or under a
subpath such as `https://you.github.io/forager-demo/`.

- **GitHub Pages, using a branch.** Push the *contents* of `demo-site/` to the root of a `gh-pages`
  branch, or of a dedicated repo. Then go to Settings → Pages and pick that branch and `/ (root)`.
  One way to do it from this repo:
  `git subtree push --prefix demo-site origin gh-pages`.
- **GitHub Pages, using `/docs`.** Copy the contents of `demo-site/` into a `docs/` folder on your
  default branch. Then go to Settings → Pages → "Deploy from a branch" and choose `/docs`. This repo
  already has a `docs/` folder, so use a separate repo if you don't want to mix the two.
- **Netlify Drop.** Drag the `demo-site` folder onto <https://app.netlify.com/drop>.
- **Vercel.** Run `npx vercel demo-site`, or import the repo with *Root Directory* set to `demo-site`,
  *Framework* set to "Other", and no build command.

Don't put the iframe and `index.html` on different origins (for example, the app on a CDN), because
the tour needs same-origin access.

## How the tour works

- **Runtime hooks.** The app exposes `window.__foragerDemo = { ready, resetDemo() }` (see
  `frontend/src/demo/CONTRACT.md`). The tour waits for `ready` before it starts. **Restart** calls
  `resetDemo()`, which reseeds the in-memory store and reloads the iframe, and then the tour returns
  to step 1. Refreshing the page has the same effect.
- **Next always works.** Each step can declare actions, such as clicks, typing, or picking a dropdown
  option. **Next** performs them inside the app.
  - If the viewer clicks the same control themselves, the tour notices and moves on without clicking
    it a second time.
  - If the viewer clicks the next section in the app's sidebar, the tour follows them there.
- **Back is safe.** **Back** only rewinds the narration; it does not undo anything in the app, so the
  app may show a later state than the step describes (a filed document is not un-filed). Steps that
  change data (approve, reject, generate, confirm, send) are recorded once under a `once` key. When
  you press **Next** on a step the tour has already gone past, it skips the actions that no longer
  apply (or gives each one a ~0.6 s timeout), so Back then Next is instant and never approves the
  same document twice. To see the original state again, use **Restart**.
- **Missing targets.** If a step's target never appears, the narration still shows without the
  spotlight. The tour never gets stuck.
- **Keyboard.** `→`, `Space`, and `Enter` go to the next step. `←` goes back. `Esc` hides or shows
  the guide (a prominent **Show guide** button remains). Arrow keys pressed inside the app only move
  the tour when no input, dropdown, or other widget has focus. While the end card is open, the rest of
  the page is inert and Tab stays inside the card. The section dots jump between sections; jumping forward replays the skipped steps'
  actions in order.
- **Narrow screens.** Below 820px the guide becomes a bottom sheet (collapsed, it is a single
  "Step N of M · Show guide" strip with a Next button), and the tour collapses the app's sidebar
  using the app's own toggle.
- **Spotlight.** The ring scrolls the target to the middle of the app's own scroll pane, waits for
  drawers and modals to finish sliding in, glides between targets (unless reduced motion is on), and
  stops its per-frame tracking when nothing has moved, resuming on scroll, resize, or app changes.
- **Reduced motion.** Animations are turned off when the system asks for reduced motion.
