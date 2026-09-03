# dagger-offload

Composite action that offloads a heavy, containerised gate command to a
Tailscale-reachable Docker host instead of running it on the GitHub-hosted
runner. Extracted from `Consiliency/governed-pipeline` PR #68 (the CI-offload
pilot) — proven live/E2E on that repo (PR #70) before extraction.

It is a **composite action** (`using: composite`), not a reusable
`workflow_call` workflow, on purpose: a `workflow_call` runs as a separate job
on a fresh runner, so it can't share the caller's already-checked-out /
`npm ci`'d workspace, and it would split the caller's required `test` check
into two separate check contexts. This action inlines as steps inside the
caller's existing job.

## Why the trust gate stays in your workflow, not in this action

This action takes a plain `eligible: 'true'/'false'` string input and no-ops
(every step skipped) when it isn't `'true'`. It does **not** compute
eligibility itself. Compute it in your own workflow, where it's auditable in
the same file that owns your CI trust boundary:

```yaml
- name: Determine CI-offload eligibility
  id: elig
  env:
    EVENT_NAME: ${{ github.event_name }}
    REF: ${{ github.ref }}
    PR_HEAD_REPO: ${{ github.event.pull_request.head.repo.full_name }}
    REPO: ${{ github.repository }}
    TS_AUTHKEY_SET: ${{ secrets.TS_AUTHKEY != '' }}
  run: |
    trusted=false
    if [ "$EVENT_NAME" = "push" ] && [ "$REF" = "refs/heads/main" ]; then
      trusted=true
    fi
    if [ "$EVENT_NAME" = "pull_request" ] && [ "$PR_HEAD_REPO" = "$REPO" ]; then
      trusted=true
    fi
    eligible=false
    if [ "$trusted" = "true" ] && [ "$TS_AUTHKEY_SET" = "true" ]; then
      eligible=true
    fi
    echo "eligible=$eligible" >> "$GITHUB_OUTPUT"
```

`trusted` means: push to `main`, or a `pull_request` whose head repo is your
own repo (never a fork). GitHub already withholds secrets from fork-PR runs,
so an absent `TS_AUTHKEY` independently forces forks onto the hosted path
even if the trust check were ever wrong.

## Adopter snippet

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      # ... your existing checkout / npm ci / fast checks, unchanged ...

      # (paste the eligibility step from above here)

      - name: Gate (offloaded)
        if: steps.elig.outputs.eligible == 'true'
        uses: Consiliency/ci-actions/dagger-offload@<pin-to-a-commit-sha>
        with:
          command: "npm run agent:gate"
          eligible: ${{ steps.elig.outputs.eligible }}
          remote-host: ai
          ts-authkey: ${{ secrets.TS_AUTHKEY }}
          # dagger-version / ci-tag / ssh-user: defaults match the fleet
          # convention (auto / tag:ci-gp / ci-docker) — override only if
          # your remote host differs. "auto" = the engine version already
          # running on the remote host (see below).

      - name: Gate (hosted)
        if: steps.elig.outputs.eligible != 'true'
        run: npm run agent:gate
```

**Always pin `uses:` to a commit SHA**, not a branch or floating tag — this
repo is private and shared across the fleet; a SHA pin means a change here
never silently changes behavior for a repo that hasn't re-pinned.

**Keep both gate steps** (offloaded + hosted) in your caller, each gated on
the *opposite* value of `eligible`. This is the mechanism that keeps the
whole thing optional-by-default (see below) and fail-closed once eligible —
do not gate the hosted step on `failure()` or `always()`; it must only ever
run when `eligible != 'true'`, never as a fallback from a failed offload.

## Why this repo is public

The fleet this action serves spans **four separate GitHub owners**:
`Consiliency` (org), `ViperJuice` (personal user account),
`Frontierstrategies-ai` (org), and `regenesis-ai` (org). GitHub does not let a
workflow in one owner reference an action in a **private** repo owned by a
different owner — cross-owner `uses:` only resolves against a public repo (or
one covered by the same enterprise/org, which doesn't apply across these four
independent owners). So `Consiliency/ci-actions` is public.

This is safe: `action.yml` contains no secrets. The real access control is
the ephemeral `TS_AUTHKEY` each caller supplies at the call site, plus
tailnet ACL membership. The action only references non-secret names (host
`ai`, user `ci-docker`, tag `tag:ci-gp`) — infra topology, not credentials.
Public visibility means anyone can *read* this action; nobody can *use* your
compute without your own key.

## Prerequisites per adopting repo

1. **A containerised, `dagger call`-able gate.** Your `command` needs a code
   path that, when `AGENT_DAGGER=1` and `AGENT_REMOTE_HOST=<host>` (or
   `DOCKER_HOST=ssh://<host>`) are set, runs the gate via the `dagger` CLI
   rather than a local `npm test`/equivalent. See
   `governed-pipeline`'s `scripts/agent-validation.mjs` (`daggerRemoteEnv()`)
   for the reference implementation — it's a small, portable pattern to copy.
2. **`TS_AUTHKEY` available — the exact mechanism depends on which of the
   four owners your repo lives in:**
   - **`Consiliency`, `Frontierstrategies-ai`, `regenesis-ai` (orgs):** set
     `TS_AUTHKEY` once as an **org-level** secret with `visibility: all`
     (Consiliency already runs this way). Every repo in that org, including
     new ones, inherits it automatically — no per-repo secret to provision.
   - **`ViperJuice` (personal user account):** GitHub personal accounts
     don't have org-level/fleet-wide secrets. Provision a **per-repo**
     `TS_AUTHKEY` secret on each repo under this account that adopts the
     action (`gh secret set TS_AUTHKEY -R ViperJuice/<repo>`). This is the
     one owner where "adopt by reference" still means a manual secret step
     per repo.
3. **The shared `tag:ci-gp` ACL grant** (or your own tag) already wired on
   your tailnet, granting that tag SSH access to your remote host's docker
   tag (e.g. `tag:ai-docker:22`) — see governed-pipeline PR #68's
   description for the exact ACL policy block. This is a one-time,
   fleet-wide setup; you don't redo it per repo, only per new remote host.
4. **No repo-visibility settings to configure on your side.** Because
   `Consiliency/ci-actions` is public, any repo under any of the four owners
   can already reference `uses: Consiliency/ci-actions/dagger-offload@<sha>`
   with no Actions-access settings change of its own.

## Dagger CLI version: derived from the remote engine

The dagger CLI's docker provisioner starts an engine container named for
**its own** version (`dagger-engine-v<X>`) and removes every other
`dagger-engine-*` container it finds on the Docker host. A CLI that does not
match the engine already running there therefore does not just skew
coverage — it kills that engine (and any gate mid-flight on it, from any
caller) and provisions its own. A hardcoded CLI pin turns every engine
upgrade on the host into a CI outage until every adopter re-pins.

So `dagger-version` defaults to `auto`: the action lists the running
`dagger-engine-v*` container on `remote-host` and installs exactly that CLI
version. The CLI is a pure function of the host's engine state:

- **Upgrading the engine** is an operator step on the host and needs no
  change in any adopting repo. Do it when no gate is running there (the
  GC above kills a live one): run a CLI of the target version once against
  the host — e.g. `DAGGER_VERSION=<X> sh -c "$(curl -fsSL
  https://dl.dagger.io/dagger/install.sh)"` and then
  `DOCKER_HOST=ssh://<host> dagger version` — and it provisions
  `dagger-engine-v<X>` and removes the old one. Every subsequent run of
  this action derives `<X>`.
- **No engine running** (fresh host, after a prune): `auto` installs the
  latest release and the CLI provisions that engine on first use; later
  runs derive it. This is the only path on which the version is not read
  from the host, and the step log says so.
- **Any other CLI that reaches the host** — a developer's local `dagger`
  with `DOCKER_HOST=ssh://<host>` — must match the running engine too, for
  the same reason. Keep local CLIs at the host's engine version.

Pass an explicit `dagger-version: "0.21.7"` to disable the derivation; then
you own keeping it equal to the host's engine.

## Optional-by-default (nobody is blocked by lacking tailnet access)

This is a layered progressive enhancement, not a requirement:

1. **Baseline** — direct local gate, needs nothing. Always available, always
   correct fallback.
2. **Dagger** — `AGENT_DAGGER=1 dagger call ...`, needs Docker locally. Opt-in.
3. **Offload** (this action) — needs a reachable `TS_AUTHKEY`-gated remote
   host. Opt-in, secret-gated: no secret → every run is `eligible: false` →
   the hosted step runs exactly as it does without this action at all.

## Fail-closed guarantee (re-verified under composite execution)

The inline form (gp PR #68) proved fail-closed as a single job's linear
steps. Extracting the offload path into a **composite action** changes step
ownership (steps 2+ now live inside `uses: .../dagger-offload@...` instead of
being peers in the caller's `steps:` list), which changes how failure
propagates. This was re-tested empirically, not assumed:

- Built a throwaway test workflow (in this repo, `uses: ./dagger-offload`
  locally) reproducing the caller's dual-step structure (offloaded step
  gated `if: eligible == 'true'`, hosted-fallback step gated
  `if: eligible != 'true'`), with `remote-host` pointed at a nonexistent
  tailnet host so the **preflight Docker probe** — not tailnet connect
  itself — is what fails.
- Result: the composite action's preflight step failed, the action step
  failed, the job's conclusion was `failure`, and the hosted-fallback step's
  conclusion was `skipped` — **not** `success`. No silent fallback, no
  false-green.
- A second run with `eligible: 'false'` confirmed the opposite shape: every
  action step skipped, the hosted step ran, job green — the intended no-op
  behavior when offload isn't eligible.

Concrete run (branch `test/fail-closed-proof`, deleted after capture — see
`.github/workflows/failclosed-proof.yml` in git history at commit
`e5dedeb`): https://github.com/Consiliency/ci-actions/actions/runs/28619246704

- Job `unreachable-host-must-fail-closed` → conclusion `failure`. Tailscale
  connected successfully (real org `TS_AUTHKEY`); the `docker -H
  ssh://nonexistent-host-ci-failclosed-proof.invalid version` preflight step
  failed on `ssh: Could not resolve hostname ...: Name or service not
  known`, exit code 1. The `Gate (hosted fallback — must be SKIPPED, never
  run)` step's conclusion was `skipped`. Job conclusion: `failure`.
- Job `ineligible-must-noop-and-use-hosted` → conclusion `success`. The
  `dagger-offload` action's every internal step was `skipped` (input
  `eligible: 'false'`); the hosted-fallback step ran and printed its
  expected message. Job conclusion: `success`.

No false-green: the offload path's failure never let the hosted step run,
and it never reads as a passing job.
