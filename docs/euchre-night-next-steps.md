# Euchre Night — Next Steps

*A handoff for later. The app is complete and deployed; nothing here is required. Two independent tracks: polish the app, and use the repo as a CI/CD sandbox.*

**Where things stand**
- Live at `https://euchre-tournament-dd9a4.web.app`, deployed from a single static `index.html`.
- Full arc works: Google host sign-in, sign-up lobby (code + join link), seeded/validated rotation with robots, live scoring, standings, results with the KLASK Champion badge, end-early, CSV/JSON export, a cloud archive that follows your login, and 18 imported historical events.
- Code is on GitHub. Deploys are currently **manual** (`firebase deploy`) — which is exactly what makes it a good CI/CD sandbox.
- Security model: no player accounts (join by code); host is identity-based; Firestore rules enforce that only the owner can change structure, while anyone with a code can check in and enter scores.
- **Open item:** the host archive occasionally threw `permission-denied` (a token-timing race, worst on Back-navigation). Patched with auth-ready waits, backoff retries, a Retry button, and dropping the multi-tab cache — awaiting one real-world confirmation. The "public archive" task below is expected to resolve this at the root and let us delete that retry scaffolding.

---

## Do this first — Public family archive (+ fixes the archive bug)

Kim wants the whole family to see past results, and decided the data being publicly readable is fine (it's first names and euchre scores). This is the recommended next task because it does double duty: it delivers the family archive **and** removes the token-timing fragility above.

**Why it fixes the bug:** the `permission-denied` came from the `list` rule requiring `request.auth` to be valid at query time. Make listing public and the read no longer depends on the host's token being ready — so cold loads and Back-navigations can't be denied.

**Rules change (one line opens up; writes stay owner-locked):**

```
allow get, list: if true;
allow create: if request.auth != null && request.resource.data.ownerUid == request.auth.uid;
allow update: if (request.auth != null && resource.data.ownerUid == request.auth.uid)
              || request.resource.data.diff(resource.data).affectedKeys().hasOnly(['players','scores']);
allow delete: if request.auth != null && resource.data.ownerUid == request.auth.uid;
```

**App change (small):**
- Host view needs nothing — "Your tournaments" already filters by `where ownerUid == you` in the query, so it stays scoped to the host regardless of the open rule.
- Add a public **"KLASK history"** entry on the landing screen, visible with no sign-in, that lists past tournaments (read-only) and opens any to its results screen. Family just open the app URL and browse — no accounts, no separate share link, no publish step.
- After deploying, retest the Back-navigation case; if clean, remove the retry/backoff scaffolding in `renderArchive`/`loadMine` since it's no longer load-bearing.

**Tradeoff Kim accepted:** "public" means anyone with the app URL could list all tournaments (first names + scores), not just family. Unguessable codes still protect individual events from being stumbled upon; this only opens the *list*.

---

## Track 1 — App enhancements (all optional)

Roughly ordered by value. Only the first is more than cosmetic.

1. **Archive pagination.** The one genuinely useful item. "Your tournaments" (and the new public history list) fetches every tournament on each load; with 18 imported plus every future night, that list will get heavy. The clean version: query the newest ~10, show a "Load more," page back by `createdAt` with a cursor (Firestore `limit()` + `startAfter()`). Do this once the list actually feels long.

2. **Robot naming.** Let the host rename the padding bots (your "BMO," etc.) instead of "Robot 1/2." Small: an editable label per robot at generate time, carried into the rotation and standings.

3. **Reuse last roster.** A "start from last tournament's players" shortcut so regular crews don't retype names every time. Pulls the roster from a prior event in your archive into a fresh sign-up.

4. **Co-hosts.** Now genuinely possible because hosting is identity-based. Store a list of owner UIDs instead of one; add a way to invite another signed-in Google account as co-host. Update the rules' owner checks to test membership in that list.

5. **Themes.** Beyond the current felt-and-cards look — alternate palettes, maybe a per-tournament accent. Purely cosmetic.

6. **Per-event dates on imported records.** Imported events show "Imported" rather than a real date, and standings rather than a round-by-round replay (the old sheets didn't hold reliable rotation data). If you ever want true dates, we'd backfill them by hand from the sheet tab names.

---

## Track 2 — CI/CD sandbox (the learning goal)

The app is a good teaching prop *because* it's simple: one static file, no build step, and the only real secret is the deploy credential. That isolates the CI/CD mechanics from build complexity you'd otherwise fight. Natural fit: **GitHub Actions → Firebase Hosting**. Build it up in stages so you touch each concept deliberately.

### Prereqs (already done)
- Repo on GitHub. ✓
- Firebase Hosting project. ✓

### Stage 1 — CI gate before anything ships (smallest real slice)
Add a workflow that runs on every push and *checks something* — formalizing the `node --check` syntax check we've run by hand all along. This teaches workflow files, triggers, and jobs, and instills the "gate before you ship" instinct that is the heart of CI.

Create `.github/workflows/ci.yml` (adjust the path to wherever `index.html` lives — likely `public/index.html`):

```yaml
name: CI
on: [push, pull_request]
jobs:
  syntax-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Extract inline module and syntax-check
        run: |
          node -e "const fs=require('fs');const h=fs.readFileSync('public/index.html','utf8');const m=h.match(/<script type=\"module\">([\s\S]*?)<\/script>/);if(!m){throw new Error('no module script found');}fs.writeFileSync('app.mjs',m[1]);"
          node --check app.mjs
```

Push it, then watch it run under the repo's **Actions** tab. Break the JS on purpose and confirm the check goes red — that red X is the whole point of CI.

### Stage 2 — PR preview deploys
Run `firebase init hosting:github` in your repo. It scaffolds an Actions workflow, creates a Firebase service-account **secret** in GitHub for you, and sets up **preview channels**: every pull request gets its own temporary URL. This is where you learn secrets handling and the "deploy preview per PR" pattern.

### Stage 3 — Branch protection + required checks
In GitHub repo settings, protect `main`: require the Stage 1 CI job to pass before a PR can merge. Now you've wired gating to the merge itself — a core real-world practice.

### Stage 4 — CD to production on merge
Have the Firebase deploy run only on push to `main` (the `hosting:github` setup gives you most of this). Merging a PR now ships to your live site automatically. That completes the loop: commit → CI gate → preview → merge → auto-deploy.

### Stage 5 — Stretch, for work-transferable breadth
- **Matrix builds** (run the check across multiple Node versions).
- **Dependency caching** (`actions/cache`) to feel build-speed tuning.
- **Manual approval gates** via GitHub Environments (require a click before prod deploy).
- **Status badge** in the README.
- **Swap the deploy target** (Cloudflare Pages or GitHub Pages) to feel how portable the CI half is versus the CD half.

### Secrets note
The Firebase service-account key lives as a GitHub Actions **secret**, never in the repo — the exact pattern for any real credential at work. Your Firebase *web config* stays in the client code as before; it isn't a secret. So the only sensitive value in the whole pipeline is the deploy credential, which keeps the secrets lesson clean and low-stakes.

---

## Suggested resume point

Start with **the public family archive** (top of this doc). It's what Kim asked for next, it's a small change, and it's expected to resolve the lingering archive `permission-denied` at the root — after which the retry scaffolding can come out. After that, the highest-value polish item is **archive pagination**, and the best first CI/CD step is **Stage 1** (the `node --check` gate you've already watched pass a dozen times). Each is a clean, self-contained session.
