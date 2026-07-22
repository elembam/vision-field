# Setup — Moving This Project into VS Code

This file is for **Claude Code** (or any AI coding assistant) running inside
VS Code, to get this project from "pasted into GitHub's web editor" to "a
normal local repo I can edit, preview, and push from." It's a one-time setup
doc — for what the project *does* and where it's going next, see
`HANDOVER.md` in the same repo.

## What you're setting up

A GitHub repo (already created by the user, name may vary — ask if it's not
obvious from `git remote -v` or the folder name) containing:

```
index.html      — currently whichever tool was most recently deployed
                  (check the <title> tag to identify which one — see below)
HANDOVER.md     — domain knowledge + feature roadmap, written for Copilot,
                  equally relevant to you
```

The repo is deployed via **GitHub Pages**, serving `index.html` from the
repo root at a `https://<username>.github.io/<repo-name>/` URL. That's the
user's only current way to test on their iPhone, so **don't break the
assumption that `index.html` in the root is the deployed file** unless you
restructure deployment deliberately (e.g. moving to a `docs/` folder or a
`gh-pages` branch — either is fine, just update GitHub's Pages source
setting to match, and tell the user).

### There may be multiple tool variants floating around

Over the course of building this, three related-but-separate single-file
tools were created:

1. A **projection/sightline calculator** (comparing a near plane like a
   printed page to a far plane like a canvas on a wall) — title contains
   "Projector Model"
2. A **live camera overlay** (acuity-zone rings drawn over live video, no
   image filtering) — title contains "Vision Field — Camera"
3. A **static-image foveation filter** (take/choose a photo, actually blurs
   it per-zone) — title contains "Vision Field — Foveation Filter"

Only one of these is likely sitting in the repo as `index.html` right now
(whichever the user last deployed). **Check with the user which one(s) they
want tracked in this repo** — if they want more than one, they're separate
pages (e.g. `index.html`, `camera.html`, `calculator.html`), not variants to
merge, since they serve different purposes. `HANDOVER.md`'s stated priority
is developing the foveation filter (#3) further, so default to assuming
that's the one to focus local dev tooling around unless told otherwise.

## Step 1 — Confirm the repo state

```bash
git remote -v          # confirms the GitHub URL, sanity-checks you're in the right repo
git log --oneline -10  # recent history, useful context
ls                     # confirm index.html / HANDOVER.md are present
```

If this is a fresh clone and these commands fail, the user hasn't cloned it
yet — walk them through `git clone <url>` first (they may prefer GitHub
Desktop over the CLI; either is fine, just confirm which before assuming
terminal git commands will be run by them personally).

## Step 2 — Local preview setup

No build step, no npm dependencies, no framework — these are single static
HTML files. But **don't just double-click to open as a `file://` URL** for
anything involving the camera or file inputs with certain APIs — browsers
restrict some features (camera access in particular) to "secure contexts,"
and `file://` doesn't qualify. `http://localhost` **does** qualify as
secure, so a trivial local server is enough:

```bash
# either works, pick whichever is already installed
python3 -m http.server 8000
npx serve .
```

Then open `http://localhost:8000` in a desktop browser. This is sufficient
for testing everything **except** actually using the iPhone's camera from
the iPhone itself — desktop testing covers UI logic, the render pipeline,
calibration math, etc., using the desktop's own webcam or file uploads as
stand-ins.

If VS Code's **Live Server** extension is installed, that's an equally
good alternative to the two commands above — auto-reloads on save, which is
convenient during active editing. Use whichever the user already has
installed; don't insist on installing a new extension if a one-line server
command works fine.

## Step 3 — Testing on the actual iPhone

This is the part that can't be fully replicated locally: iOS requires real
HTTPS for camera access (localhost exemption doesn't extend across devices
on the network). Options, roughly in order of friction:

1. **Push to GitHub, let Pages redeploy, test at the live URL.** Slowest
   iteration loop but zero extra setup — this is what the user's been doing
   so far.
2. **A tunneling tool** (ngrok, Cloudflare Tunnel, or similar) pointed at
   the local dev server, giving a temporary `https://` URL reachable from
   the phone without a full deploy. Worth setting up if iteration speed
   becomes a real bottleneck — don't set this up preemptively if the user
   hasn't asked for it, since it's one more account/tool in their stack.

Either way, **after any change touching the camera, calibration math, or
image loading, ask the user to actually test on their phone** before
considering a task done — this project has already hit one real bug
(iOS silently killing the page from memory pressure on large photo decode)
that was invisible in desktop testing and only showed up on-device.

## Step 4 — Git workflow

Standard stuff, but worth stating explicitly since the user's prior workflow
was pasting into GitHub's web textbox (no git involved at all):

```bash
git add -A
git commit -m "<description>"
git push
```

If the user is using GitHub Desktop instead of CLI git, the equivalent is
the "Commit to main" + "Push origin" buttons — don't assume CLI commands are
how they'll actually ship changes; ask if unsure, and match VS Code's
built-in Source Control panel to whichever they're more comfortable with
(it works with either).

## Step 5 — Confirm nothing regressed

After moving to local dev, do a quick smoke test before any new feature
work starts, since the goal here is transferring the environment, not
changing behavior:

- [ ] Photo load (both "Take photo" and "Choose from library" inputs)
- [ ] Tap-to-set-fixation on the image
- [ ] Calibration flow: drag markers, save, badge updates to "calibrated"
- [ ] Apply foveation filter renders without a crash/white-screen
- [ ] Save/share produces a downloadable PNG

If any of these regress after the environment change (they shouldn't — it's
the same file, just served differently), that's a local-serving issue
(MIME types, relative paths, etc.), not a logic bug — check the browser
console first before assuming the JS itself broke.

## What NOT to do in this setup pass

- Don't start on the feature roadmap (WebGL shader, live video, etc.) as
  part of "setup" — that's `HANDOVER.md`'s job, and it's a much bigger
  scope than getting the environment running. Finish the transfer, confirm
  the smoke test passes, then stop and let the user direct what's next.
- Don't introduce a build tool (Vite, webpack, npm) as part of this setup
  unless the user asks — the project is deliberately zero-build-step right
  now (see `HANDOVER.md`'s note on this), and that's a decision to revisit
  with them, not something to change incidentally while "just setting up
  the dev environment."
- Don't assume the fonts (`@import` from Google Fonts) or any other
  external references need fixing for local dev — they'll load fine over
  a normal internet connection locally; that's a separate offline/PWA
  concern already tracked in `HANDOVER.md`, not a local-dev blocker.
