# AndroidAPS fork CI/CD Implementation Plan

> **Status note (2026-07-22).** This spec and its plan were written against
> `funman300/AndroidAPS-old-fork`, which turned out to be superseded — this
> repository is the live one, and the work was ported here. Two things landed
> differently from what is described below:
>
> - **There is no ktlint job.** A full-repo `ktlintCheck --continue` on
>   unmodified upstream code reports 94,345 violations across 3,249 files in
>   every module, so it cannot act as a gate. The cause looks like upstream
>   configuration: `.editorconfig` still uses ktlint 0.x `disabled_rules=`
>   syntax, which ktlint 1.x ignores. See the header of `verify.yml`.
> - **Releases publish to Gitea**, not GitHub releases, so signed APKs stay
>   private now that this repository is public.
>
> Everything else — the verification workflow, the post-merge `ref` fix, the
> sync failure/conflict reporting, and the signing-certificate gate — is
> implemented as described.


> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make this fork notice automatically when a commit — in practice, the daily unattended upstream merge — breaks the build, and stop it publishing an APK signed with the wrong key.

**Architecture:** A composite action holds the JDK/Gradle setup shared by three parallel verification jobs in a new `verify.yml`. `sync-upstream.yml` calls `verify.yml` explicitly after a merge that actually moved `master`, and opens a single tracking issue when it fails. `auto-release.yml` gains a signing-certificate gate and richer release notes.

**Tech Stack:** GitHub Actions, Gradle 8 / Android Gradle Plugin, Kotlin, JDK 21 (from `.github/jdk-map.json`), ktlint via `org.jlleitschuh.gradle.ktlint` 14.0.1, `apksigner`, Gitea releases API, `gh` and `jq` (both preinstalled on GitHub runners).

## Global Constraints

- Everything lives inside this repository as GitHub Actions workflows. Nothing runs against any host.
- JDK version always comes from `jq -r '.default' .github/jdk-map.json` — currently `21`. Never hardcode it.
- Gradle JVM tuning must match what `branch-ci.yml` already uses: `-Xmx8g -XX:+UseParallelGC -Xss1024m` for the build JVM, `-Xmx2g` for the Kotlin daemon, in-process Kotlin compilation, `workers.max=8`, build caching on. `-Xss1024m` is load-bearing for Kotlin compilation of this codebase — do not drop it.
- Caching uses `actions/setup-java@v5`'s `cache: gradle`. Do not introduce `gradle/actions/setup-gradle` — a second caching mechanism in the same repo is a divergence with no benefit.
- **No verification job may reference any secret.** Debug variants are signed with the standard debug key, which is what makes `pull_request` runs from forks safe. `KEYSTORE_SET`, `KEYSTORE_BASE64`, `KEY_ALIAS`, `KEY_PASSWORD`, `GDRIVE_OAUTH2` and `ZIP_PASSWORD` must not appear in `verify.yml`.
- `verify.yml` has **no `push` trigger**. See the spec's section 1 rationale: with `sync-upstream.yml` checking out via `WORKFLOW_PUSH_TOKEN || github.token`, a push trigger either never fires or double-builds, depending on whether that PAT secret happens to exist.
- `app/google-services.json` is committed; no Firebase setup step is needed. `-PfirebaseDisable` is carried for parity with `runtests.sh` only — it matches no `hasProperty` check in the build and is a no-op.
- Existing workflows keep their current behaviour. `aaps-ci.yml`, `branch-ci.yml`, `pr-ci.yml`, `cherry-pick-ci.yml`, `keystore-export.yml` and `cleanup-workflow-runs.yml` are not touched at all.
- Work happens on branch `ci/verify`. Do not push to `master`.

## File Structure

| File | Responsibility |
| --- | --- |
| `.github/actions/setup-aaps-build/action.yml` (new) | JDK resolution, Java setup with Gradle cache, `chmod +x gradlew`. Shared by all verify jobs. |
| `.github/workflows/verify.yml` (new) | Three parallel jobs: ktlint, unit tests, assemble. No secrets. |
| `.github/workflows/sync-upstream.yml` (modify) | Emit a `changed` output, call `verify.yml`, open/comment one tracking issue on failure. |
| `.github/workflows/auto-release.yml` (modify) | Verify the signing certificate before publishing; enrich release notes. |

---

### Task 1: Shared build-setup composite action

Three verification jobs need identical JDK and Gradle setup. A composite action keeps that in one place instead of triplicating it.

**Files:**
- Create: `.github/actions/setup-aaps-build/action.yml`

**Interfaces:**
- Consumes: nothing. Assumes the repository is already checked out.
- Produces: a composite action referenced as `uses: ./.github/actions/setup-aaps-build`. Takes no inputs. Sets `JAVA_VERSION` in `$GITHUB_ENV`, installs that Temurin JDK with the Gradle cache enabled, and makes `gradlew` executable.

- [ ] **Step 1: Write the action**

```yaml
name: Set up AAPS build
description: >-
  Resolve the JDK version from .github/jdk-map.json, install it with the Gradle
  cache enabled, and make gradlew executable. Shared by every verify.yml job so
  the setup exists in exactly one place.

runs:
  using: composite
  steps:
    - name: Load JDK version from jdk-map.json
      shell: bash
      run: |
        JAVA_VERSION=$(jq -r '.default' .github/jdk-map.json)
        echo "Using JDK ${JAVA_VERSION}"
        echo "JAVA_VERSION=${JAVA_VERSION}" >> "$GITHUB_ENV"

    - name: Set up JDK
      uses: actions/setup-java@v5
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: temurin
        cache: gradle

    - name: Grant execute permission for gradlew
      shell: bash
      run: chmod +x gradlew
```

- [ ] **Step 2: Validate it parses and matches the repo's JDK map**

Run:
```bash
python3 -c "import yaml; d=yaml.safe_load(open('.github/actions/setup-aaps-build/action.yml')); print(d['runs']['using'], len(d['runs']['steps']), 'steps')"
jq -r '.default' .github/jdk-map.json
```
Expected: `composite 3 steps` and `21`.

- [ ] **Step 3: Commit**

```bash
git add .github/actions/setup-aaps-build/action.yml
git commit -m "ci: add shared build-setup composite action"
```

---

### Task 2: Verification workflow, dispatch-only first

The workflow is created with `workflow_dispatch` only so it can be run and observed before it gates anything. Two facts are unknown until it runs for real: whether `master` is ktlint-clean, and how long each job takes. Task 3 acts on the answers.

**Files:**
- Create: `.github/workflows/verify.yml`

**Interfaces:**
- Consumes: `./.github/actions/setup-aaps-build` from Task 1.
- Produces: a workflow named `Verify` with jobs `ktlint`, `unit-tests`, `assemble`. In Task 3 it gains `workflow_call` so `sync-upstream.yml` can invoke it as `uses: ./.github/workflows/verify.yml`.

- [ ] **Step 1: Write the workflow**

```yaml
name: Verify

# Automated build verification for this fork.
#
# Every other build workflow here is workflow_dispatch-only, so nothing notices
# when a commit breaks the build — and in practice the commits that land on
# master are unattended daily merges from sync-upstream.yml.
#
# No secrets are referenced anywhere in this file. Debug variants are signed
# with the standard debug key, which is what makes pull_request runs from forks
# safe to execute.

on:
  workflow_dispatch: {}   # Task 3 adds pull_request and workflow_call

concurrency:
  group: verify-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

env:
  # Matches branch-ci.yml's tuning. -Xss1024m is load-bearing for Kotlin
  # compilation of this codebase. GRADLE_OPTS is parsed by the gradle launcher,
  # which respects the embedded quoting.
  GRADLE_OPTS: >-
    -Dorg.gradle.jvmargs="-Xmx8g -XX:+UseParallelGC -Xss1024m"
    -Dkotlin.daemon.jvm.options="-Xmx2g"
    -Dkotlin.compiler.execution.strategy=in-process
    -Dorg.gradle.daemon=true
    -Dorg.gradle.workers.max=8
    -Dorg.gradle.caching=true

jobs:
  ktlint:
    name: ktlint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Set up build
        uses: ./.github/actions/setup-aaps-build

      - name: Run ktlintCheck
        run: ./gradlew ktlintCheck -PfirebaseDisable

      - name: Upload ktlint reports
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: ktlint-reports
          path: '**/build/reports/ktlint/**'
          retention-days: 7
          if-no-files-found: ignore

  unit-tests:
    name: Unit tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Set up build
        uses: ./.github/actions/setup-aaps-build

      - name: Run unit tests
        run: ./gradlew -PfirebaseDisable testFullDebugUnitTest

      - name: Upload test reports
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-reports
          path: '**/build/reports/tests/**'
          retention-days: 7
          if-no-files-found: ignore

  assemble:
    name: Assemble
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Set up build
        uses: ./.github/actions/setup-aaps-build

      - name: Assemble debug APKs
        run: ./gradlew -PfirebaseDisable :app:assembleFullDebug :wear:assembleFullDebug
```

- [ ] **Step 2: Verify no secret is referenced**

Run:
```bash
grep -nE 'secrets\.|KEYSTORE|KEY_ALIAS|KEY_PASSWORD|GDRIVE|ZIP_PASSWORD' .github/workflows/verify.yml; echo "exit=$?"
```
Expected: no output, `exit=1`. Any match violates a Global Constraint and makes fork PRs unsafe.

- [ ] **Step 3: Validate the YAML and the Gradle tuning**

Run:
```bash
python3 - <<'PY'
import yaml
w = yaml.safe_load(open('.github/workflows/verify.yml'))
print("jobs:", sorted(w['jobs']))
opts = w['env']['GRADLE_OPTS']
for needle in ['-Xmx8g', '-Xss1024m', 'UseParallelGC', 'workers.max=8', 'caching=true']:
    assert needle in opts, f"missing {needle}"
print("GRADLE_OPTS OK")
PY
```
Expected: `jobs: ['assemble', 'ktlint', 'unit-tests']` then `GRADLE_OPTS OK`.

- [ ] **Step 4: Commit and push the branch**

```bash
git add .github/workflows/verify.yml
git commit -m "ci: add verification workflow (dispatch-only for now)"
git push -u origin ci/verify
```

- [ ] **Step 5: Dispatch it against the branch and record the outcome**

> **This step does not work as written.** GitHub only permits `workflow_dispatch`
> for workflow files that exist on the repository's **default branch**, so
> dispatching `Verify` while it lives only on `ci/verify` returns HTTP 404.
> Observation is therefore deferred to Task 3, which adds the `pull_request`
> trigger — a same-repo pull request runs workflows from the pull request's own
> branch, so opening the PR is what makes the three jobs observable.

From the Actions tab, run `Verify` selecting branch `ci/verify`. Wait for all three jobs.

Record for the next task:
- Did `ktlint` pass? If it failed, download the `ktlint-reports` artifact and count the violations and how many distinct files they touch.
- Did `unit-tests` pass?
- Did `assemble` pass?
- Wall-clock duration of each job.

- [ ] **Step 6: Handle a red `unit-tests` or `assemble`**

If either fails on unmodified upstream code, stop and report before changing anything. That is a real pre-existing breakage in the fork and is exactly what this work exists to surface — but fixing it is separate work and must not be folded into this branch silently.

---

### Task 3: Wire the triggers based on what Task 2 observed

**Files:**
- Modify: `.github/workflows/verify.yml`

**Interfaces:**
- Consumes: the Task 2 observations.
- Produces: `verify.yml` callable as `uses: ./.github/workflows/verify.yml` and running automatically on pull requests.

- [ ] **Step 1: Replace the `on:` block**

Replace:
```yaml
on:
  workflow_dispatch: {}   # Task 3 adds pull_request and workflow_call
```

with:
```yaml
on:
  pull_request:
  workflow_call: {}
  workflow_dispatch: {}
```

There is deliberately no `push` trigger — see Global Constraints and the spec's section 1 rationale.

- [ ] **Step 2: If and only if ktlint failed in Task 2, narrow it to changed files**

Skip this step entirely if `ktlint` passed — the full-repo check is preferable and is the default.

If it failed with pre-existing violations, do not reformat the fork: that creates conflicts with every future upstream merge. Narrow the check to the **Gradle modules** containing changed Kotlin files.

Scoping by module rather than by file is deliberate. The ktlint Gradle plugin registers a real `ktlintCheck` task per project (`:app:ktlintCheck`, `:core:ui:ktlintCheck`, …), so module scoping uses documented task addressing. Per-file filtering would depend on plugin-internal properties, which is not something to build a gate on.

A Gradle project path is derived by convention: walk up from the changed file to the nearest directory containing a `build.gradle.kts` or `build.gradle`, take that path relative to the repo root, and replace `/` with `:`.

Replace the `Run ktlintCheck` step with:

```yaml
      - name: Determine changed Gradle modules
        id: changed
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            BASE="${{ github.event.pull_request.base.sha }}"
          else
            BASE="$(git rev-parse HEAD~1)"
          fi

          git diff --name-only --diff-filter=d "$BASE" HEAD -- '*.kt' '*.kts' > /tmp/changed.txt || true
          echo "Changed Kotlin files:"; cat /tmp/changed.txt

          : > /tmp/modules.txt
          while IFS= read -r f; do
            [ -n "$f" ] || continue
            d="$(dirname "$f")"
            while [ "$d" != "." ] && [ "$d" != "/" ]; do
              if [ -f "$d/build.gradle.kts" ] || [ -f "$d/build.gradle" ]; then
                echo ":$(echo "$d" | tr '/' ':'):ktlintCheck" >> /tmp/modules.txt
                break
              fi
              d="$(dirname "$d")"
            done
          done < /tmp/changed.txt

          sort -u /tmp/modules.txt -o /tmp/modules.txt
          TASKS="$(paste -sd' ' /tmp/modules.txt)"
          echo "tasks=$TASKS" >> "$GITHUB_OUTPUT"
          echo "ktlint tasks: ${TASKS:-<none>}"

      - name: Run ktlintCheck on changed modules
        if: steps.changed.outputs.tasks != ''
        run: ./gradlew -PfirebaseDisable ${{ steps.changed.outputs.tasks }}
```

Before committing this variant, confirm the derived task names are real:

```bash
./gradlew tasks --all 2>/dev/null | grep -E ':ktlintCheck$' | head -20
```

Every task emitted into `/tmp/modules.txt` must appear in that list. If a derived name is absent, the convention-based mapping is wrong for that module and must be corrected before relying on it.

and add `fetch-depth: 0` to that job's checkout so the diff base is available:

```yaml
      - name: Checkout
        uses: actions/checkout@v5
        with:
          fetch-depth: 0
```

Then re-dispatch `Verify` on `ci/verify` and confirm the ktlint job passes.

- [ ] **Step 3: Validate the triggers**

Run:
```bash
python3 -c "import yaml; print(sorted(yaml.safe_load(open('.github/workflows/verify.yml'))[True]))"
```
Expected: `['pull_request', 'workflow_call', 'workflow_dispatch']`

(PyYAML parses the bare key `on` as the boolean `True` — that is expected, not a bug in the workflow.)

- [ ] **Step 4: Confirm there is no push trigger**

Run:
```bash
python3 -c "
import yaml,sys
t = yaml.safe_load(open('.github/workflows/verify.yml'))[True]
assert 'push' not in t, 'push trigger present — see Global Constraints'
print('OK: no push trigger')
"
```
Expected: `OK: no push trigger`

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/verify.yml
git commit -m "ci: run verification on pull requests and via workflow_call"
```

---

### Task 4: Verify synced master and report breakage

**Files:**
- Modify: `.github/workflows/sync-upstream.yml`

**Interfaces:**
- Consumes: `.github/workflows/verify.yml` from Task 3.
- Produces: job `sync` gains outputs `changed` (`'true'`/`'false'`) and `range` (`<before>..<after>`); new jobs `verify` and `report-failure`.

- [ ] **Step 1: Add outputs to the `sync` job**

Change the job header from:
```yaml
  sync:
    name: Sync Fork with Upstream
    runs-on: ubuntu-latest
    steps:
```
to:
```yaml
  sync:
    name: Sync Fork with Upstream
    runs-on: ubuntu-latest
    outputs:
      changed: ${{ steps.sync.outputs.changed }}
      range: ${{ steps.sync.outputs.range }}
    steps:
```

- [ ] **Step 2: Make the sync step report whether master actually moved**

Replace the whole `Sync master branch` step with:

```yaml
      - name: Sync master branch
        id: sync
        run: |
          git checkout master
          BEFORE="$(git rev-parse HEAD)"
          if git merge upstream/master --ff-only; then
            echo "✅ Fast-forward merge succeeded."
          else
            echo "⚠️ Fast-forward not possible, doing regular merge..."
            git merge upstream/master -m "chore: sync with upstream master"
          fi
          AFTER="$(git rev-parse HEAD)"
          git push origin master
          if [ "$BEFORE" = "$AFTER" ]; then
            echo "changed=false" >> "$GITHUB_OUTPUT"
            echo "ℹ️ master was already up to date — no new upstream commits."
          else
            echo "changed=true" >> "$GITHUB_OUTPUT"
            echo "range=${BEFORE}..${AFTER}" >> "$GITHUB_OUTPUT"
            echo "✅ master synced with upstream (${BEFORE} → ${AFTER})."
          fi
```

- [ ] **Step 3: Add the verify and report jobs**

Append to the end of the file, at the same indentation as `sync:`:

```yaml
  verify:
    name: Verify synced master
    needs: sync
    if: needs.sync.outputs.changed == 'true'
    uses: ./.github/workflows/verify.yml

  report-failure:
    name: Report broken sync
    needs: [sync, verify]
    if: always() && needs.verify.result == 'failure'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    steps:
      - name: Open or update the breakage issue
        env:
          GH_TOKEN: ${{ github.token }}
          RANGE: ${{ needs.sync.outputs.range }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        run: |
          TITLE="Upstream sync broke the build"
          {
            echo "Verification failed after syncing upstream \`nightscout/AndroidAPS\`."
            echo
            echo "- Merged range: \`${RANGE}\`"
            echo "- Failed run: ${RUN_URL}"
            echo
            echo "\`master\` is not currently known to build."
          } > /tmp/body.md

          EXISTING=$(gh issue list --state open --search "\"$TITLE\" in:title" \
            --json number,title \
            --jq ".[] | select(.title == \"${TITLE}\") | .number" | head -1)

          if [ -n "$EXISTING" ]; then
            echo "Commenting on existing issue #${EXISTING}"
            gh issue comment "$EXISTING" --body-file /tmp/body.md
          else
            echo "Opening a new issue"
            gh issue create --title "$TITLE" --body-file /tmp/body.md
          fi
```

Consecutive daily syncs against a still-broken `master` comment on one issue rather than opening a new one each morning.

- [ ] **Step 4: Validate the job graph**

Run:
```bash
python3 - <<'PY'
import yaml
w = yaml.safe_load(open('.github/workflows/sync-upstream.yml'))
jobs = w['jobs']
print("jobs:", list(jobs))
assert jobs['sync']['outputs'] == {
    'changed': '${{ steps.sync.outputs.changed }}',
    'range':   '${{ steps.sync.outputs.range }}',
}
assert jobs['verify']['uses'] == './.github/workflows/verify.yml'
assert jobs['verify']['if'] == "needs.sync.outputs.changed == 'true'"
assert jobs['report-failure']['if'] == "always() && needs.verify.result == 'failure'"
print("OK")
PY
```
Expected: `jobs: ['sync', 'verify', 'report-failure']` then `OK`.

- [ ] **Step 5: Confirm the untouched parts really are untouched**

Run:
```bash
git diff .github/workflows/sync-upstream.yml | grep -E '^[-+]' | grep -vE '^[-+]{3}' | grep -cE 'gh workflow run|NEW_TAGS|GITEA'
```
Expected: `0`. The tag-detection and Gitea-release-triggering logic must be byte-for-byte unchanged; only the sync step and the new jobs change.

- [ ] **Step 6: Commit**

```bash
git add .github/workflows/sync-upstream.yml
git commit -m "ci: verify master after an upstream sync and report breakage"
```

---

### Task 5: Gate the release on the signing certificate

Obtainium refuses to update an app whose signing key changed, so publishing an APK signed with the wrong key silently strands every installed client. There is no recovery short of a manual reinstall by each user.

**Files:**
- Modify: `.github/workflows/auto-release.yml`

**Interfaces:**
- Consumes: `env.VERSION` and `env.VERSION_SUFFIX`, both already set by earlier steps in this workflow; the renamed APKs `aaps-<version><suffix>.apk` and `aaps-wear-<version><suffix>.apk` produced by `Rename APKs with version`.
- Produces: a hard failure before publication when the certificate does not match the repository variable `RELEASE_CERT_SHA256`.

- [ ] **Step 1: Insert the verification step**

Insert immediately after the `Rename APKs with version` step and before the `# ── Publish to Gitea ──` comment:

```yaml
      - name: Verify APK signing certificate
        env:
          EXPECTED: ${{ vars.RELEASE_CERT_SHA256 }}
        run: |
          set -euo pipefail
          BT="$(ls -1 "$ANDROID_HOME/build-tools" | sort -V | tail -1)"
          APKSIGNER="$ANDROID_HOME/build-tools/$BT/apksigner"
          if [ ! -x "$APKSIGNER" ]; then
            echo "❌ apksigner not found at $APKSIGNER"
            exit 1
          fi
          echo "Using $APKSIGNER"

          APK="aaps-${{ env.VERSION }}${{ env.VERSION_SUFFIX }}.apk"
          WEAR_APK="aaps-wear-${{ env.VERSION }}${{ env.VERSION_SUFFIX }}.apk"
          EXPECTED_LC="$(echo "${EXPECTED:-}" | tr 'A-F' 'a-f' | tr -d ' :')"

          FAILED=0
          for FILE in "$APK" "$WEAR_APK"; do
            echo "🔍 Verifying $FILE"
            if ! OUT="$("$APKSIGNER" verify --print-certs "$FILE" 2>&1)"; then
              echo "❌ $FILE failed signature verification:"
              echo "$OUT"
              FAILED=1
              continue
            fi
            echo "$OUT"
            FP="$(echo "$OUT" | grep -i 'SHA-256 digest:' | head -1 \
                  | awk '{print $NF}' | tr 'A-F' 'a-f' | tr -d ' :')"
            if [ -z "$FP" ]; then
              echo "❌ Could not read a certificate fingerprint from $FILE"
              FAILED=1
              continue
            fi
            if [ -z "$EXPECTED_LC" ]; then
              echo "❌ Repository variable RELEASE_CERT_SHA256 is not set."
              echo "   Observed fingerprint: $FP"
              echo "   Once you have confirmed that is your real release key, set it under"
              echo "   Settings → Secrets and variables → Actions → Variables."
              FAILED=1
              continue
            fi
            if [ "$FP" != "$EXPECTED_LC" ]; then
              echo "❌ Signing certificate mismatch for $FILE"
              echo "   expected: $EXPECTED_LC"
              echo "   actual:   $FP"
              echo "   Publishing this would break Obtainium updates for every existing install."
              FAILED=1
              continue
            fi
            echo "✅ $FILE is signed with the expected release certificate."
          done
          [ "$FAILED" -eq 0 ]
```

- [ ] **Step 2: Confirm the step is positioned before publication**

Run:
```bash
python3 - <<'PY'
import yaml
steps = yaml.safe_load(open('.github/workflows/auto-release.yml'))['jobs']['build-release']['steps']
names = [s.get('name') for s in steps]
v = names.index('Verify APK signing certificate')
p = names.index('Create Gitea release and upload APKs')
r = names.index('Rename APKs with version')
print(f"rename={r} verify={v} publish={p}")
assert r < v < p, "verification must sit between renaming and publishing"
print("OK")
PY
```
Expected: the three indices in ascending order, then `OK`.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/auto-release.yml
git commit -m "ci: refuse to publish an APK signed with an unexpected certificate"
```

- [ ] **Step 4: Record the required one-time setup**

The first `auto-release` run after this merges **will fail on purpose**, printing the observed fingerprint. That is the intended bootstrap: read the fingerprint from the log, confirm it is the real release key, then set repository variable `RELEASE_CERT_SHA256` to it and re-run. Note this in the PR description so it is not mistaken for a regression.

---

### Task 6: Enrich the Gitea release notes

**Files:**
- Modify: `.github/workflows/auto-release.yml`

**Interfaces:**
- Consumes: `$APK`, `$WEAR_APK`, `$TAG`, `$VERSION`, `$SUFFIX`, `$BASE` and `$AUTH`, all already defined inside the `Create Gitea release and upload APKs` step.
- Produces: a release body containing a per-asset table of size, size delta versus the previous release, and SHA-256.

- [ ] **Step 1: Build the body before the release is created**

Inside the `Create Gitea release and upload APKs` step, insert immediately after the `AUTH="Authorization: token $GITEA_TOKEN"` line:

```bash
          # ── Build the release body ───────────────────────────────────────────
          # Fetch a few recent releases and pick the newest that is not this tag,
          # so a re-run against an existing release still compares against the
          # genuine predecessor.
          PREV=$(curl -sf -H "$AUTH" "$BASE/releases?limit=5" || echo '[]')

          prev_size_of() {
            echo "$PREV" | jq -r --arg n "$1" --arg t "$TAG" \
              '[ .[] | select(.tag_name != $t) ] | .[0].assets[]? | select(.name == $n) | .size' \
              | head -1
          }

          fmt_size() { awk -v s="$1" 'BEGIN{printf "%.2f MiB", s/1048576}'; }

          fmt_delta() {
            local now="$1" prev="$2"
            if [ -z "$prev" ] || [ "$prev" = "null" ]; then echo "n/a"; return; fi
            awk -v d="$((now - prev))" -v p="$prev" \
              'BEGIN{printf "%+.2f MiB (%+.1f%%)", d/1048576, (p==0 ? 0 : 100*d/p)}'
          }

          {
            printf 'Automated build of AAPS %s from upstream nightscout/AndroidAPS.\n\n' "$VERSION"
            printf '| Asset | Size | Δ vs previous | SHA-256 |\n'
            printf '| --- | --- | --- | --- |\n'
            for FILE in "$APK" "$WEAR_APK"; do
              NAME="$(basename "$FILE")"
              SIZE="$(stat -c%s "$FILE")"
              SHA="$(sha256sum "$FILE" | awk '{print $1}')"
              printf '| `%s` | %s | %s | `%s` |\n' \
                "$NAME" "$(fmt_size "$SIZE")" \
                "$(fmt_delta "$SIZE" "$(prev_size_of "$NAME")")" "$SHA"
            done
          } > /tmp/release-body.md

          RELEASE_BODY="$(cat /tmp/release-body.md)"
          echo "Release body:"
          cat /tmp/release-body.md
```

- [ ] **Step 2: Use the body on create, and update it on re-run**

Replace this block:

```bash
          if [ -n "$EXISTING_ID" ]; then
            echo "ℹ️ Release $TAG already exists (id=$EXISTING_ID) — uploading assets."
            RELEASE_ID="$EXISTING_ID"
          else
            echo "📦 Creating Gitea release $TAG..."
            BODY=$(jq -n \
              --arg tag  "$TAG" \
              --arg name "AAPS ${VERSION}${SUFFIX}" \
              --arg body "Automated build of AAPS ${VERSION} from upstream nightscout/AndroidAPS." \
              '{tag_name: $tag, name: $name, body: $body, draft: false, prerelease: false}')
            RESPONSE=$(curl -sf -X POST -H "$AUTH" -H "Content-Type: application/json" \
              "$BASE/releases" -d "$BODY")
            RELEASE_ID=$(echo "$RESPONSE" | jq -r '.id')
            echo "✅ Release created (id=$RELEASE_ID)"
          fi
```

with:

```bash
          if [ -n "$EXISTING_ID" ]; then
            echo "ℹ️ Release $TAG already exists (id=$EXISTING_ID) — refreshing notes and uploading assets."
            RELEASE_ID="$EXISTING_ID"
            PATCH=$(jq -n --arg body "$RELEASE_BODY" '{body: $body}')
            curl -sf -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
              "$BASE/releases/$RELEASE_ID" -d "$PATCH" >/dev/null
          else
            echo "📦 Creating Gitea release $TAG..."
            BODY=$(jq -n \
              --arg tag  "$TAG" \
              --arg name "AAPS ${VERSION}${SUFFIX}" \
              --arg body "$RELEASE_BODY" \
              '{tag_name: $tag, name: $name, body: $body, draft: false, prerelease: false}')
            RESPONSE=$(curl -sf -X POST -H "$AUTH" -H "Content-Type: application/json" \
              "$BASE/releases" -d "$BODY")
            RELEASE_ID=$(echo "$RESPONSE" | jq -r '.id')
            echo "✅ Release created (id=$RELEASE_ID)"
          fi
```

- [ ] **Step 3: Test the body-building logic locally**

The shell functions are testable without GitHub or Gitea. Run:

```bash
cd /tmp && mkdir -p relnotes && cd relnotes
head -c 3000000 /dev/urandom > aaps-3.4.2.3.apk
head -c 1500000 /dev/urandom > aaps-wear-3.4.2.3.apk

TAG=3.4.2.3
VERSION=3.4.2.3
APK=aaps-3.4.2.3.apk
WEAR_APK=aaps-wear-3.4.2.3.apk
PREV='[{"tag_name":"3.4.2.2","assets":[{"name":"aaps-3.4.2.3.apk","size":2900000}]}]'

prev_size_of() {
  echo "$PREV" | jq -r --arg n "$1" --arg t "$TAG" \
    '[ .[] | select(.tag_name != $t) ] | .[0].assets[]? | select(.name == $n) | .size' | head -1
}
fmt_size() { awk -v s="$1" 'BEGIN{printf "%.2f MiB", s/1048576}'; }
fmt_delta() {
  local now="$1" prev="$2"
  if [ -z "$prev" ] || [ "$prev" = "null" ]; then echo "n/a"; return; fi
  awk -v d="$((now - prev))" -v p="$prev" \
    'BEGIN{printf "%+.2f MiB (%+.1f%%)", d/1048576, (p==0 ? 0 : 100*d/p)}'
}

{
  printf 'Automated build of AAPS %s from upstream nightscout/AndroidAPS.\n\n' "$VERSION"
  printf '| Asset | Size | Δ vs previous | SHA-256 |\n'
  printf '| --- | --- | --- | --- |\n'
  for FILE in "$APK" "$WEAR_APK"; do
    NAME="$(basename "$FILE")"
    SIZE="$(stat -c%s "$FILE")"
    SHA="$(sha256sum "$FILE" | awk '{print $1}')"
    printf '| `%s` | %s | %s | `%s` |\n' \
      "$NAME" "$(fmt_size "$SIZE")" \
      "$(fmt_delta "$SIZE" "$(prev_size_of "$NAME")")" "$SHA"
  done
}
```

Expected: a well-formed markdown table where `aaps-3.4.2.3.apk` shows `2.86 MiB` and a delta of about `+0.10 MiB (+3.4%)`, and `aaps-wear-3.4.2.3.apk` shows `1.43 MiB` and `n/a` (no matching prior asset). Confirm no stray leading whitespace on the table rows — leading spaces would render as a code block instead of a table.

- [ ] **Step 4: Clean up**

```bash
rm -rf /tmp/relnotes
```

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/auto-release.yml
git commit -m "ci: report size, delta and checksum in Gitea release notes"
```

---

### Task 7: Open the pull request and verify end to end

**Files:**
- No file changes.

**Interfaces:**
- Consumes: everything from Tasks 1–6.
- Produces: a merged, verified pipeline on `master`.

- [ ] **Step 1: Confirm the branch state**

```bash
git status --short
git log --oneline master..ci/verify
```
Expected: clean tree, six commits.

- [ ] **Step 2: Push and open the PR**

```bash
git push -u origin ci/verify
gh pr create --base master --head ci/verify \
  --title "Add automated build verification" \
  --body "Implements docs/superpowers/specs/2026-07-21-androidaps-cicd-design.md

Note: the first auto-release run after this merges will fail on purpose,
printing the signing certificate fingerprint to set as the repository
variable RELEASE_CERT_SHA256. See Task 5 of the implementation plan."
```

If `gh` is not authenticated on this machine, open the PR through the web UI.

- [ ] **Step 3: Confirm `Verify` runs on the PR itself**

This is the real test of the `pull_request` trigger. Expected: `Verify / ktlint`, `Verify / Unit tests` and `Verify / Assemble` all appear as checks on the PR and pass. Their runtimes should be close to what Task 2 recorded.

- [ ] **Step 4: Merge**

Merge once the three checks are green.

- [ ] **Step 5: Verify the sync integration on the next dispatch**

From the Actions tab, run `Sync Upstream & Trigger Release` manually. Expected, depending on upstream state:

- **No new upstream commits:** the `sync` job logs `ℹ️ master was already up to date`, and both `verify` and `report-failure` are skipped.
- **New upstream commits:** `sync` logs the `BEFORE → AFTER` range, `verify` runs the three jobs, and `report-failure` is skipped when they pass.

Either outcome confirms the wiring. Do not force a sync just to see the second path.

- [ ] **Step 6: Set `RELEASE_CERT_SHA256`**

Trigger `auto-release.yml` for an already-released upstream tag. It will fail at `Verify APK signing certificate` and print the observed fingerprint. Confirm that fingerprint matches your real release key, set it as repository variable `RELEASE_CERT_SHA256`, and re-run. Expected on the re-run: both APKs report `✅ ... signed with the expected release certificate`, and the Gitea release body contains the asset table.

---

## Post-implementation notes

- If `verify` becomes the bottleneck on daily syncs, the cheapest saving is
  dropping `:wear:assembleFullDebug` from the `assemble` job — the wear module
  rarely breaks independently of `app`. Do not drop the unit tests.
- A daily upstream merge invalidates much of the Gradle cache, so the post-sync
  run is the slowest of the week. That is also the run that matters most.
- `report-failure` intentionally reuses one issue title. If you close the issue
  while `master` is still broken, the next failing sync opens a fresh one.
