# Real Estate AI Guide — Deployment Workflow

This is a single-file static site: `public/index.html` contains all HTML, CSS,
and JS inline. No build step, no npm dependencies, no bundler. `package.json`
exists only to describe the project (`"build": "echo 'No build needed..."`) —
there is nothing to actually install or compile.

**There is no automated test suite in this repo.** No Jest/Vitest, no test
files, nothing `npm test` would find. If a future request references a
numbered test count ("confirm all N tests pass") or a URL under
`gethellobrain.com`, stop and check — that almost certainly belongs to the
*separate* `hello-brain` repo (a real React/Vite/Supabase app with its own
Vitest suite), not this one. Do not fabricate a test-pass result here.

Verification in this repo is a manual JS syntax check plus a set of browser
assertions run through Claude's browser tools every session. See §3.

## 1. Git workflow

Remote: `origin` → `https://github.com/feaths/real-estate-ai-guide.git`, branch `main`.
Vercel auto-deploys on every push to `main` (project connected via Vercel's
GitHub integration — no manual `vercel deploy` needed, no `vercel.json`
`builds`/`routes`; it's the modern `{ "outputDirectory": "public" }` config).

Standard sequence:

```bash
cd ~/real-estate-ai-guide
git status                       # confirm you're clean before starting

# ... make edits to public/index.html ...

# Extract the inline <script> and check it actually parses
python3 -c "
import re
content = open('public/index.html').read()
m = re.search(r'<script>(.*)</script>', content, re.S)
open('/tmp/check.js','w').write(m.group(1))
"
node --check /tmp/check.js && echo "JS OK"

# Serve locally and run the browser checks in §3 before committing
git add -A
git diff --cached                # eyeball the diff — confirm scope matches intent
git commit -m "Short summary" -m "Longer body: what changed and why, what was verified"
git push origin main
```

There is no pre-commit hook and no CI gate — the JS syntax check and browser
verification in §3 *are* the test suite for this repo. Do them before every
commit that touches `public/index.html`, not after.

## 2. Verifying the live deployment

Vercel typically finishes a deploy within 5–20 seconds of the push. Poll
rather than guess:

```bash
i=0
while [ $i -lt 20 ]; do
  body=$(curl -s https://implement-ai-guide.vercel.app)
  if echo "$body" | grep -q "<some string unique to your change>"; then
    echo "LIVE after $((i*4))s"
    break
  fi
  sleep 4
  i=$((i+1))
done
echo "$body" | grep -o "<other strings to confirm>" | sort -u
```

**Cache gotcha:** Vercel's CDN can serve a `HIT` for a few seconds after
deploy. If a check seems stale, run `curl -sD - <url> -o /dev/null` and look
at the `age` and `x-vercel-cache` response headers — `age: 0` / `MISS` means
you're looking at the fresh deploy, not a cached prior version.

## 3. Pre-deployment checklist

Run this against a local server before pushing, and re-confirm against the
live URL after:

```bash
cd ~/real-estate-ai-guide/public
python3 -m http.server <port>
```

Then, via the browser tool:

- **Console**: zero errors on load (`read_console_messages` with `onlyErrors: true`).
- **Heading hierarchy**: exactly one `<h1>`, zero level-skips (walk
  `h1..h6`, fail if any level jumps by more than 1 from the previous heading).
- **ARIA**: `role="navigation"` present on both `<nav>` elements;
  `aria-expanded` flips correctly on click *and* on hash-based auto-open;
  every `aria-controls` id resolves to a real element.
- **Nav links**: every `.toc a` and `.fn-nav a` `href="#..."` resolves via
  `document.querySelector` — a dangling link to a hidden/removed section is
  the most common regression in this repo (see §5).
- **Keyboard focus — must use a real keypress, not `.focus()`.** Click
  somewhere on the page first (or reload), then send an actual `Tab` via the
  `computer` tool and read `document.activeElement`. Calling `.focus()`
  programmatically after any prior `.click()` in the same session frequently
  fails to trigger `:focus-visible` (browser input-modality heuristics),
  producing a false "no outline" reading that isn't real.
- **Color contrast**: spot-check computed `color`/`background-color` on a
  few key elements against the token table in §5 — don't eyeball it.
- **200% zoom**: `document.documentElement.style.fontSize = '200%'`, then
  confirm `document.body.scrollWidth <= document.body.clientWidth + 1`
  (no horizontal page scroll). Reset the font-size after.
- **375px width**: resize to the `mobile` preset, confirm zero elements have
  `getBoundingClientRect().right > viewportWidth`.
- **Skip link**: first focusable element in `<body>`, `href="#main-content"`,
  visible on focus.
- **`lang="en"`** still present on `<html>`.
- If a section was hidden or unhidden this session, confirm no
  `getElementById(...)` call in the JS throws against a now-missing element
  (see §5).

Only consider a deploy "done" once the same checks pass against the **live**
Vercel URL, not just localhost.

## 4. Repo consolidation — planned, not yet executed

The user has a standing plan (proposed, currently on hold — see chat history,
not yet approved to execute) to merge this repo into `github.com/feaths/hello-brain`
at `public/guides/real-estate/index.html`, preserving full commit history via:

```bash
git clone https://github.com/feaths/real-estate-ai-guide.git /tmp/guide-history-rewrite
cd /tmp/guide-history-rewrite
git filter-repo --path public/index.html \
  --path-rename public/index.html:public/guides/real-estate/index.html
# then fetched into hello-brain and merged with --allow-unrelated-histories
```

**What this means for how we work now, before the merge happens:**

- **Keep `public/index.html` at exactly that path.** The filter-repo command
  above targets that literal path. If it's ever renamed or moved within this
  repo, the merge command needs updating to match — flag that explicitly if
  it happens.
- **Keep commits atomic and clearly scoped.** Every commit on this file
  becomes permanent history inside hello-brain post-merge. Squashed or vague
  commits now are noise forever there — this is the reason recent commits in
  this repo carry detailed bodies, not just one-line summaries.
- **Keep the file fully self-contained.** No external JS/CSS files, no build
  step, no relative imports outside itself. This is *why* the merge plan is
  simple — Vite serves anything under hello-brain's `public/` verbatim, no
  React/bundler integration needed. Don't introduce a dependency that would
  break that.
- **hello-brain already proxies to this repo.** Its `vercel.json` has:
  ```json
  { "rewrites": [{ "source": "/guides/real-estate/:path*", "destination": "https://implement-ai-guide.vercel.app/:path*" }] }
  ```
  `gethellobrain.com/guides/real-estate` is a *rewrite proxy* today, not a
  real subpage — the files still live only here. Don't assume otherwise in
  future sessions just because the guide is reachable from that domain.
- **After the merge (when it happens):** remove that rewrite block from
  hello-brain's `vercel.json`, decommission (don't immediately delete) this
  repo's Vercel project, and archive rather than delete this GitHub repo as
  a rollback safety net.
- `git-filter-repo` is not installed on this machine yet
  (`brew install git-filter-repo` first).

## 5. Edge cases and gotchas from past sessions

- **Hiding a section**: wrap it in
  `<!-- Hidden section: Name ... End hidden section: Name -->` in the HTML,
  *and* comment out the matching JS that calls `getElementById` on ids
  inside it — otherwise the page throws `Cannot read properties of null` on
  load. Pure/reusable functions with no DOM dependency (e.g.
  `assembledPrompt()`) should stay active if another hidden feature still
  references them — check before commenting out a whole block.
- **Divider placement when hiding a section**: the `<hr class="part-divider">`
  that *precedes* a hidden section goes inside the hidden comment; the one
  that *follows* stays visible as the bridge to the next section — unless
  the hidden section was the last one, in which case no trailing divider is
  needed (the footer has its own `border-top`). Getting this backwards
  produces either a doubled `<hr>` or an orphaned gap.
- **Renumbering "Part N" touches more places than expected**: eyebrow
  `<div>` text, `<!-- PART N: ... -->` HTML comments, `/* ... (Part N) */`
  CSS comments, `// Part N: ...` JS comments, and prose that names a part by
  number (e.g. the download card's "every prompt from Part X..."). Always
  `grep -n "Part [0-9]"` after a renumbering pass to catch stragglers before
  committing.
- **The header `.toc` nav is manual.** Adding, hiding, or removing a section
  does not update it automatically — a stale `<a href="#save">` after `#save`
  is hidden is a dead link. Check it every time a section's visibility or id
  changes.
- **Two-tier rust color — don't collapse them.**
  `--primary` (`#C4623A`, bright) is safe only for *large* text/icons/borders
  (≥3:1 contrast). `--primary-strong` (`#A8502E`, darker) is required for
  *small* text (≥4.5:1). `--btn-bg`/`--btn-bg-hover` are separate,
  theme-invariant tokens used only for solid button fills with white text —
  they exist because dark mode's lighter `--primary` fails white-on-button
  contrast. Reusing bright `--primary` as a small-text color re-breaks WCAG AA;
  this has already been audited once with real contrast-ratio math — don't
  re-guess it, ask if unsure which tier a new element needs.
- **`:focus-visible` false negatives** — see §3. Always test with a real
  `computer` keypress, never trust a `.focus()` call after a prior click in
  the same session.
- **Screenshot tool intermittently returns a blank frame** in this sandbox
  when the Browser pane isn't frontmost ("pane is currently hidden"). Don't
  loop retrying — rely on JS-based DOM/CSS assertions instead (more
  reliable here) and take at most one visual screenshot per change, per the
  artifact-design "look once" convention.
- **`downloadAll()`'s `window.claude.use('downloads')` path** is dead code on
  the real Vercel deployment (`window.claude` doesn't exist outside a Claude
  Artifact preview) — it silently falls through to a normal blob-download
  `<a>` there. Currently unreachable anyway since the Download section is
  hidden; preserved for when it's re-enabled.
- **No CI/CD beyond Vercel's git-push trigger.** No GitHub Actions, no
  branch protection encountered so far. Pushing to `main` deploys directly.
