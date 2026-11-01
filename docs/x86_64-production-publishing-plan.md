# TermiNow x86_64 — Production apt-repository publishing plan (for review)

Status: **DRAFT, contingent on the isolated staging validation passing** (terminow-package run
`41274730768`, branch `x86_64-staging-validation`). Do not execute any of this until staging is green
and the termux-app PR #3 evidence is reviewed. x86_64 remains experimental; no Google Play upload.

## 1. Why production needs special care

The production publish in `.github/workflows/bootstrap.yml` (job `bootstrap`, step *Publish package
repository*) does:

```
git init -b gh-pages ; git add -A ; git commit ; git push --force ... gh-pages
```

It **force-pushes the whole site** every run — the branch is *replaced*, not appended. The live
`gh-pages` today holds the **aarch64-only** repository that every existing TermiNow user installs
from. Therefore:

- A run that builds **only x86_64** would overwrite gh-pages and **break every aarch64 user**.
- The only safe production publish builds **`aarch64,x86_64` together**, so the new site is a
  **superset** of today's.
- Because the push is `--force`, the previous commit is no longer referenced on the branch. It
  remains reachable only by its SHA until GitHub garbage-collects unreferenced objects, so we must
  not rely on it: make an explicit backup first (section 4).

## 2. One-line enablement (the change that turns it on)

Durable: set in `scripts/terminow/config.sh`

```
: "${TERMINOW_ARCHITECTURES:=aarch64,x86_64}"
```

or, for a one-off without changing the default, dispatch `bootstrap.yml` with
`-f architectures=aarch64,x86_64`. **Either way the run must include aarch64.** The guard in section 3
makes "forgot aarch64" fail the run rather than relying on discipline.

## 3. ARM64 preservation check (hard gate, BEFORE the force-push)

Add a step after *Create package repository* and before *Publish package repository* that fails the
run (so the force-push never happens) unless all of the following hold. It compares the **current
live** aarch64 repo against the **newly built** site on disk:

1. `dists/stable/main/binary-aarch64/Packages` exists in the new site and is non-empty.
2. The new `dists/stable/Release` lists `aarch64` in `Architectures:`.
3. **No aarch64 regression**: every `Package`/`Version` currently live at
   `${TERMINOW_REPO_URL}/dists/stable/main/binary-aarch64/Packages` is present in the new aarch64
   index at a version `>=` the live one. Fail, listing any missing or downgraded package.
4. `test-repo-install.sh <new site dir> aarch64 <extras> <repo packages>` passes (apt simulated
   install with signature verification) against the **new site on disk, pre-publish**.

Sketch (runs on the already-built `$RUNNER_TEMP/site`):

```bash
live="$TERMINOW_REPO_URL/dists/stable/main/binary-aarch64/Packages"
new="$RUNNER_TEMP/site/apt/terminow-main/dists/stable/main/binary-aarch64/Packages"
[ -s "$new" ] || { echo "::error::new site has no aarch64 index"; exit 1; }
grep -q '^Architectures:.*aarch64' "$RUNNER_TEMP/site/apt/terminow-main/dists/stable/Release" || exit 1
curl -fsS "$live" -o /tmp/live-aarch64.Packages
python3 scripts/terminow/tests/assert_no_regression.py /tmp/live-aarch64.Packages "$new"   # new helper
./scripts/terminow/test-repo-install.sh "$RUNNER_TEMP/site/apt/terminow-main" aarch64 \
    ${TERMINOW_EXTRA_PACKAGES//,/ } ${TERMINOW_REPO_PACKAGES//,/ }
```

`assert_no_regression.py` (new, ~30 lines): parse both `Packages`, map name->version (dpkg order),
exit non-zero if any live name is absent in new or `dpkg --compare-versions new lt live`.

## 4. Backup (BEFORE the force-push, non-destructive)

Before publishing, snapshot the current live branch so rollback is a single push:

```bash
backup="gh-pages-backup-$(date -u +%Y%m%d)-r${{ github.run_number }}"
git fetch origin gh-pages:refs/remotes/origin/gh-pages
git push origin "refs/remotes/origin/gh-pages:refs/heads/$backup"     # additive, no force
```

Also upload the fetched live `gh-pages` tree as a workflow artifact (`gh-pages-backup`, retention 30d)
so a copy exists independent of the repository's branches. Keep the last N backup branches; prune
older ones in a separate maintenance step, never in the publish path.

### 4.1 Backup coverage (what it does and does not capture)

The backup captures **exactly the files on the `gh-pages` branch at backup time**: the signed apt
tree — `pool/main/*.deb`, `dists/stable/main/binary-<arch>/Packages{,.gz,.xz}`, `dists/stable/Release`,
`Release.gpg`, `InRelease`, and `.nojekyll`. Restoring it returns the served repository bit-for-bit to
that state.

It does **not** capture, and the plan does not attempt to restore:

- **The live GitHub Pages deployment / CDN edge cache.** Restoring the branch starts a new Pages
  deployment; until it propagates (typically 1–5 min, occasionally longer) and edge caches expire,
  clients may still receive the just-published (broken) content. Restoration is therefore not
  instantaneous at the client.
- **Release assets** (bootstrap zips, proot loaders, `bootstrap-sha256sums`,
  `terminow-bootstrap.properties`). These live in GitHub **Releases**, not on `gh-pages`, and are
  additive per run (a new tag), so a publish does not overwrite prior releases. They are out of scope
  for this backup; roll them back, if ever needed, by pointing consumers at the prior release tag.
- **The app repository.** `app/terminow-bootstrap.properties` is updated separately in termux-app
  after manual verification; nothing here changes it.
- **In-flight client state.** A device that fetched the broken index between publish and rollback has
  cached it per its own apt cache TTL; it recovers on its next `apt update`, not when we restore.

## 5. Publish (unchanged mechanics, now gated)

The existing *Publish package repository* step runs only if section 3 passes and the section 4 backup
succeeded. The force-push itself is unchanged; it is now safe because the content is a verified
superset and a restorable backup exists.

## 6. Post-publish verification (BOTH arches, against the live URL)

The existing *Check published package repository* step already polls the live `InRelease` and runs
`test-repo-install.sh` per architecture. Extend it to re-run the aarch64 no-regression diff against
the **live URL** (not just the on-disk site). If any check fails → rollback (section 7) and fail the
run.

## 7. Rollback procedure and its verification limits

Automated-on-failure (add to the workflow, `if: failure()` after publish) and documented for manual
use.

**Steps:**

```bash
# 1. Restore the pre-publish site to gh-pages from the backup branch made in section 4:
git push --force "https://x-access-token:${TOKEN}@github.com/${REPO}.git" \
    "refs/heads/$backup:refs/heads/gh-pages"

# 2. Wait for the Pages redeployment to propagate (bounded poll), then confirm aarch64 is served:
for i in $(seq 1 30); do
  curl -fsS "$TERMINOW_REPO_URL/dists/stable/InRelease" | grep -q aarch64 && break
  sleep 20
done

# 3. Re-verify aarch64 installability against the live URL:
./scripts/terminow/test-repo-install.sh "$TERMINOW_REPO_URL" aarch64 <extras> <repo packages>
```

Manual fallback if CI is unavailable: run the same force-push from a maintainer workstation using the
backup branch (or the `gh-pages-backup` artifact), then step 3.

**What the rollback actually restores:** the `gh-pages` branch content (section 4.1) and, after Pages
propagates, the served repository.

**Verification limits — what a green rollback does and does not prove:**

- `test-repo-install.sh` verifies that the live index is **signature-valid and that apt can resolve
  and simulate installing** the required packages. It does **not** install or run anything on a real
  device, so it does not prove on-device install/launch; that remains manual QA.
- The check reads through GitHub Pages' CDN. Until propagation completes and edge caches expire, the
  poll in step 2 may pass at one edge while another still serves stale content; "healthy" means
  "healthy at the edge this runner hit", not globally instantaneous.
- The no-regression diff (section 3.3) compares package **name and version only**, not `.deb` byte
  content or dependency-closure equality; a same-version rebuild with changed contents would not be
  flagged.
- Recovery time is bounded by the force-push plus Pages propagation (typically a few minutes), not
  guaranteed instant. Clients recover on their next `apt update`, subject to their own cache TTL.

## 8. Release asset & app wiring

- `bootstrap.yml` writes `terminow-bootstrap.properties` as a release asset listing every built arch.
  With `aarch64,x86_64` it lists both bootstraps + loaders. The app repo copies those values into
  `app/terminow-bootstrap.properties` **only after** verifying the release (separate, existing
  process). x86_64 stays behind the experimental flag in the app until explicitly graduated.
- The app's default sources.list is unchanged; it already points at the production repo URL, which
  will then carry x86_64 too.

## 9. Go / no-go checklist

- [ ] Staging run `41274730768` green; PR #3 shows package-install + Ubuntu 5-checks PASS on x86_64.
- [ ] `assert_no_regression.py` added + unit-tested; preservation gate (section 3) added to the workflow.
- [ ] Backup (section 4) + rollback (section 7) steps added and dry-run-reviewed.
- [ ] Run dispatched with `architectures=aarch64,x86_64` (or config default changed).
- [ ] Post-publish verification green for BOTH aarch64 and x86_64 against the live URL.
- [ ] Decision recorded to keep x86_64 experimental in the app; no Play upload.

## 10. What this plan deliberately does NOT do

- No change to the app's experimental gating or Play track.
- No i686 (32-bit x86) — out of scope.
- No silent default flip without the preservation gate in place first.
