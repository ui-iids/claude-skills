---
name: fix-my-deployment
description: Diagnose and repair or update an EXISTING University of Idaho RCDS/IIDS deployment in the `ui-iids/kubernetes-apps` GitOps repo — audit its ArgoCD + Kustomize manifests against the canonical `apps/rcds/apps/deploy-template/` example and the app repo's CI, find what is broken or drifted, and fix it. Handles two cases — (1) REPAIR a broken deploy (ImagePullBackOff, pods pending, sync green but nothing running, previews never appearing), or (2) UPDATE a working deploy (new services, new env vars re-sealed with `kubeseal`, backing services, storage, adding PR previews or SonarQube, promoting to live). Does its kubernetes-apps work on a dated `fix/agent-fix-*` branch and opens a PR against `main` labeled `coding-agent` + `fix-deploy` for the user to review and merge. Use when the user says "fix my deployment", "my deployment is broken", "my pods won't start", "ArgoCD won't sync", "update my deployment", "add a service to my deployment", "add env vars to my deployment", "re-seal my secrets", "my preview builds aren't working", "check my kubeconfig", "why won't my pods start", or invokes /fix-my-deployment.
---

# Fix my deployment

You diagnose and repair — or update — an **existing** University of Idaho
**RCDS / IIDS** deployment. The app already has (or is supposed to have) a
manifest tree under `apps/rcds/apps/<app>/` in `ui-iids/kubernetes-apps`; your
job is to find where that tree has drifted from what the app repo builds, what
the house pattern requires, or what the user now needs — and correct it.

You do **not** run `kubectl apply` — fixes land when corrected manifests are
committed and ArgoCD syncs them. This is the same GitOps model, canonical
template, and naming contract as `/deploy-my-application`; read that skill's
`SKILL.md` if it is available, and always read the template:

```
kubernetes-apps/apps/rcds/apps/deploy-template/
```

**Read the template and its `README.md` first, every time** — it is the source
of truth for file layout, naming, and dev-vs-live differences. But remember it
is a *teaching* artifact and lags the fleet: where the template and working
sibling apps disagree, **the siblings win**. When judging whether something in
the target app is "wrong", check how PR-enabled / multi-service siblings
actually do it before flagging it.

## The two-repo model (most breakage lives on this seam)

An app spans **two** repos connected only through matching image names:

1. **The app repo** (`ui-iids/<app>` or `ui-insight/<app>`) — source + CI.
   Each `.github/workflows/build-<service>-container.yaml` publishes
   `ghcr.io/<org>/<app>[-<service>]:main`.
2. **kubernetes-apps** — the manifests referencing those exact image strings.

Most broken deployments are a violated contract between the two: an image
string that differs by one character from what CI builds, a CI trigger that
never fires for the branch previews are scoped to, a secret name that doesn't
match its `secretRef`. So diagnosis **always** reads both repos — never audit
the manifests in isolation.

## Scope and ground rules

- Apps come strictly from **`ui-iids`** and **`ui-insight`**. The GitOps repo is
  `https://github.com/ui-iids/kubernetes-apps.git`; manifests live under
  `apps/rcds/apps/<app>/`.
- **Never write to a cluster** — no `apply`, `delete`, `patch`, `exec`, or
  `rollout restart`. Read-only diagnostics are not just allowed but expected;
  they run on the context confirmed in **Step 0**, under the rules in
  **Guardrails** (never Secret contents).
- **Never** put plaintext secrets in manifests, chat, or commit messages. Env
  secrets are Bitnami `SealedSecret`s sealed by piping `.env.secrets` through
  `kubeseal` without ever reading it (Step 5 of `/deploy-my-application`,
  restated below for updates).
- Fix what you were asked to fix. While auditing you will often notice *other*
  drift (missing AppProject wildcard, unsealed live secrets, template lag) —
  **report** those findings, but don't silently expand the change set without
  asking.

## Step 0 — Establish cluster access

Do this **first**. Diagnosis is where cluster reads earn their keep — pod states
and events turn "something is broken" into a named failure mode in seconds — but
you must never guess which kubeconfig points at our dev cluster.

**Discover what's on the machine** — read-only, merging nothing, writing nothing:

```bash
echo "KUBECONFIG=${KUBECONFIG:-<unset>}"
ls -1 ~/.kube/config ~/.kube/*.yaml ~/.kube/*.conf ~/.kube/configs/* 2>/dev/null
kubectl config get-contexts        # context + cluster + server, never credentials
```

**Never `cat` a kubeconfig**, and never echo, copy, or log a token, client key,
or client certificate out of one. Context names, cluster names, and server URLs
are fine to show the user; credential fields are not — the same rule that governs
Secret contents in **Guardrails**.

**Then ask, based on what you found:**

- **Nothing found** — say so. Every invariant in Step 4 is checkable from the two
  repos alone, so the audit still runs in full; you simply lose the fast path to
  the symptom. Tell the user which conclusions are therefore inferred rather than
  observed.
- **Exactly one** — *"I found `<path>`, context `<ctx>`, server `<server>`. Can I
  use this read-only to reach the dev cluster (`k8s-dev.hpc.uidaho.edu`) for
  diagnostics?"* Wait for an actual yes.
- **More than one** — list them and **ask which to use**:

  | Path | Context | Cluster server |
  |------|---------|----------------|
  | `~/.kube/config` | `k8s-dev` | `https://…k8s-dev.hpc.uidaho.edu` |
  | `~/.kube/prod.yaml` | `k8s-prod` | `https://…` |

  **Do not pick by name.** A context called `dev` may be a personal
  kind/minikube cluster, and our `k8s-dev` may be sitting in a file called
  `config`. Only the user knows which is which — and on a repair task, pointing
  diagnostics at the wrong cluster produces confidently wrong conclusions.

**Once chosen**, pass it explicitly on every command — `kubectl --context <ctx> …`
— never relying on the current context, which the user may switch mid-session.
Confirm it reaches dev before trusting anything it tells you:

```bash
kubectl --context <ctx> cluster-info
kubectl --context <ctx> get ns apps-rcds-<app> --ignore-not-found
```

A missing `apps-rcds-<app>` namespace is itself a finding — either the manifests
were never merged, or you are pointed at the wrong cluster. Establish which
before drawing any conclusion from it.

A **"no" is final for the session.** Fall back to the repo-only audit and say
which checks went unverified.

## Step 1 — Locate the deployment and confirm identity

1. Find `apps/rcds/apps/<app>/` in kubernetes-apps. If the user is vague about
   which app, list candidates (`ls apps/rcds/apps/`) and confirm.
2. **If no manifest tree exists**, this isn't a fix — it's a first deploy.
   Stop and hand off to `/deploy-my-application`.
3. Confirm the app repo: org (`ui-iids` / `ui-insight`) and repo name, and that
   you can read its `.github/workflows/build-*-container.yaml`. If the app repo
   has **no** build workflow, the image pipeline itself is the broken piece —
   offer to scaffold it from `ui-iids/deploy-template` before touching manifests.

## Step 2 — Establish intent: repair or update?

Ask (or infer from the request) which mode you're in — it changes where you
look first:

**Mode A — Repair.** Something is failing. Get the symptom precisely:
- What's the observable failure? (pods not starting, 404 on the dev host, sync
  error in ArgoCD, previews never appearing, app can't see a config value…)
- When did it last work, and what changed since — a new service? renamed repo?
  edited workflow? rotated secrets?

**Mode B — Update.** The deploy works but needs to grow. Common asks:
- new service(s) added to the app repo,
- new/changed env vars (→ re-seal),
- a new backing service (postgres/redis/mongodb/mariadb) or persistent storage,
- adding PR previews or SonarQube after the fact,
- promoting to live.

If neither the symptom nor the desired change is clear, **ask** — you cannot
audit against an unknown target state.

## Step 3 — Gather ground truth from both repos

Before changing anything, snapshot reality:

**From the app repo:**
- every `build-*-container.yaml`: the exact image names published, the
  `on.push` / `on.pull_request` branch lists, any `paths:` filters, and the
  `DOCKER_METADATA_*` env settings,
- whether recent workflow runs actually succeeded and pushed
  (`gh run list --repo <org>/<app> --workflow=<file>` and, for tag existence,
  `gh api` against GHCR or `docker manifest inspect` — read-only),
- the branching model (`gh repo view <org>/<app> --json defaultBranchRef`).

**From kubernetes-apps:**
- the full `apps/rcds/apps/<app>/` tree as it stands,
- the canonical `deploy-template/` + at least one healthy sibling app of the
  same shape (multi-service, PR-enabled, etc.) for comparison.

**From the cluster (read-only, on the Step 0 context — see Guardrails):**

```bash
kubectl --context <ctx> get application -n argocd | grep <app>   # sync + health
kubectl --context <ctx> get pods -n apps-rcds-<app>              # what's actually running
kubectl --context <ctx> describe pod <failing-pod> -n apps-rcds-<app>
kubectl --context <ctx> get events -n apps-rcds-<app> --sort-by=.lastTimestamp | tail -30
kubectl --context <ctx> logs <pod> -n apps-rcds-<app> --tail=200 [--previous]
```

These turn "something is broken" into a named failure mode fast — see
`/deploy-my-application` Step 6b for the pod-state → cause table. But everything
below is also diagnosable from the two repos alone, so skip this block cleanly if
Step 0 came back without access, and say that you did.

## Step 4 — The audit: run the invariants as a checklist

Walk every applicable row. Most failures here are **silent** — valid YAML,
green sync, wrong result — so absence of errors proves nothing.

| # | Check | Symptom when violated |
|---|-------|----------------------|
| 1 | Every `image:` in Deployments and `imageName:` in the ImageUpdater is **byte-for-byte** what a `build-*-container.yaml` publishes (`ghcr.io/<org>/<app>[-<service>]`) | `ImagePullBackOff` / `ErrImagePull` |
| 2 | The image actually exists in GHCR — CI has run and pushed for the referenced tag | `ImagePullBackOff` on a correct-looking string |
| 3 | One Deployment+Service per built image; labels and Ingress backend `service.name` consistent (`<app>` single-service, `<app>-<service>` multi) | 404/502 on the dev host while pods run fine |
| 4 | SealedSecret `metadata.name` (`env`) == `envFrom.secretRef.name` in every Deployment | Pods stuck — referenced Secret doesn't exist |
| 5 | `deploy/overlays/dev/secrets/env.yaml` is `kind: SealedSecret` with non-empty `encryptedData`, key count matching `.env.secrets` (in `scripts/`, else repo root) | App boots but can't see (new) env vars — the classic "I added a var and nothing changed": **the file must be re-sealed, it never updates itself** |
| 6 | Every resource file is listed in its `kustomization.yaml` (base and overlays; `secrets` in the dev overlay, `env.yaml` in `secrets/kustomization.yaml`) | Resource silently absent from the cluster |
| 7 | `Application.spec.project` == `AppProject.metadata.name`; all control-plane names share the `apps-rcds-<app>` prefix; `spec.source.path` → `deploy/base`, dev overlay patch re-points to `deploy/overlays/dev` | Sync errors, or dev overlay changes never taking effect |
| 8 | Control-plane objects (`Application`, `AppProject`, `ImageUpdater`, `ApplicationSet`) live under `<app>/overlays/dev/`, **not** `<app>/deploy/overlays/dev/` | Object syncs green into the app namespace where nothing reads it — silently does nothing |
| 9 | Standalone PVC `claimName` in the Deployment matches `storage-volume.yaml` | Pod pending on a missing claim |
| 10 | Overlay relative paths to `deploy-repo-secrets-*` have the right depth (`../../../../../../../` from `deploy/overlays/<env>`) | Kustomize build error |
| 11 | Hosts: dev `<app>.k8s-dev.hpc.uidaho.edu`, PR `<app>-pr-{{.number}}...`; host-specific ConfigMap values (webhooks, CORS, OAuth redirects) match the environment | Wrong-environment callbacks; previews leaking traffic into shared dev |

**If the app has (or should have) PR previews**, additionally check every
preview invariant — these are the most common "previews just don't appear"
causes, in rough order of frequency:

| # | Check | Symptom when violated |
|---|-------|----------------------|
| P1 | `base/project.yaml` destination namespace carries the trailing `*`: `apps-rcds-<app>*` | ArgoCD rejects every preview Application as off-project — **first thing to check on an app that gained previews later** |
| P2 | The PR is actually **labeled `preview`**, and the label exists in the repo (`gh label list`) | Generator matches nothing; zero Applications, zero errors |
| P3 | `appSecretName` matches the org: `creds-github-ui-iids` / `creds-github-ui-insight` | Generator silently produces zero Applications |
| P4 | `pull-request-builds.yaml` sits in `<app>/overlays/dev/` and is listed in that `kustomization.yaml` (check #8) | Syncs green, does nothing |
| P5 | Every service's CI fires for the previewed PRs: `on.pull_request.branches` includes the PR **target** branch; no `paths:` filter on `pull_request` | One service (or all) in `ImagePullBackOff` per preview |
| P6 | Workflows set `DOCKER_METADATA_SHORT_SHA_LENGTH: 8` and `DOCKER_METADATA_PR_HEAD_SHA: true`, and emit `type=ref,event=pr,suffix=-{{sha}}` | Tag ArgoCD asks for (`pr-<N>-<8-char-sha>`) never exists |
| P7 | The env SealedSecret was sealed `--scope=cluster-wide` (annotation `sealedsecrets.bitnami.com/cluster-wide: "true"` on both `metadata` and `spec.template.metadata`) | Secret never decrypts in `apps-rcds-<app>-pr-<N>`; every preview pod hangs |
| P8 | `filters:` entries (if any) use the right field — `branchMatch` = source, `targetBranchMatch` = destination — and regexes are anchored (`^main$`) | Previews for the wrong PRs, or none |
| P9 | Namespace `apps-rcds-<app>-pr-<N>` and DNS label `<app>-pr-<N>` stay under 63 chars | Failures appear only at longer PR numbers |

One non-check worth knowing: `newTag: main` + `digest: sha256:…` together in
`deploy/overlays/dev/kustomization.yaml` **looks** like a bug that would pin
previews to `main`. It isn't — ArgoCD's `kustomize.images` override replaces
the whole entry, digest included. Leave it alone; don't "fix" it.

**Present the diagnosis before fixing.** List each violated invariant, the file
and line, the symptom it explains, and the fix. If nothing in the checklist
explains the reported symptom, say so and widen the search (cluster events,
sibling comparison) rather than changing manifests speculatively.

## Step 4b — Create the working branch in kubernetes-apps

**Do this after the diagnosis is agreed and before you change a single file in
kubernetes-apps.** Everything Step 5 touches there — manifests, a re-sealed
`env.yaml` — belongs on a dedicated branch that you open a PR from in Step 6.

**Branch name:** `fix/agent-fix-<login>-<MM>-<DD>-<YYYY>`

Resolve the two tokens rather than guessing:

```bash
gh api user --jq .login     # GitHub login, e.g. Jarred6068 — lowercase it
date +%m-%d-%Y              # zero-padded, e.g. 09-03-2026
```

If `gh` is unauthenticated or unavailable, fall back to `git config user.name`
lowercased with spaces replaced by hyphens (the convention this repo's other
skills use), and **say which source you used**.

```bash
git -C <kubernetes-apps> fetch origin
git -C <kubernetes-apps> switch -c fix/agent-fix-<login>-<MM>-<DD>-<YYYY> origin/main
```

- **Branch from an up-to-date `origin/main`.** On a repair especially, a stale
  base can make you "fix" something that was already fixed on `main` — and the
  PR arrives full of diff that isn't yours.
- **Never commit to `main`.** If the repo is already on a non-`main` branch with
  uncommitted work, **stop and ask** — on a shared GitOps repo that is very
  likely someone else's in-flight change.
- **If the branch already exists** (a second run the same day), ask whether to
  continue on it or start a fresh one with a `-2` suffix. Don't force anything.
- **Only kubernetes-apps gets a branch.** App-repo edits (CI workflows,
  `Dockerfile`s, `scripts/`, `.gitignore`) stay on whatever branch the user has
  checked out, and you never commit them.

Nothing is committed here — this only establishes where Step 5's writes land.

## Step 5 — Apply the fix or update

Make the smallest change that restores each invariant. Recipes for the common
updates:

**Add a service.** Both repos change:
- App repo: a `build-<service>-container.yaml` + `Dockerfile.<service>`
  (scaffold from `ui-iids/deploy-template`), publishing
  `ghcr.io/<org>/<app>-<service>:main`. Note: going from one service to two
  usually means the *existing* image gains no suffix retroactively — follow
  what CI actually builds, don't rename working images without the user
  opting in.
- kubernetes-apps: a Deployment+Service per new image, an ImageUpdater entry,
  Ingress routing if externally reachable, and every new file listed in its
  `kustomization.yaml`. Re-verify checks 1–4 for the new service.
- Flag that the new service's CI must run once before ArgoCD can pull.

**Add / change env vars (re-seal).** The sealed `env.yaml` is replaced
**wholesale** — you cannot append to it. The owner updates `.env.secrets` with
**all** keys, old and new.

**Find the file first: `scripts/.env.secrets`, then the repo root.** New deploys
put it in `scripts/` alongside `create-credential-secrets.sh`; apps deployed
before that convention keep it at the root. Use whichever exists — relocating it
is not this skill's job.

**Then gate on the gitignore, before re-sealing.** An existing app is exactly
where a repo predating the ignore rule surfaces, and the two plaintext artifacts
(`.env.secrets`, and the `secrets.yaml` that `create-credential-secrets.sh`
leaves on disk between its two commands) may well be uncovered:

```bash
git -C <app-repo> check-ignore -v \
  .env.secrets secrets.yaml scripts/.env.secrets scripts/secrets.yaml
git -C <app-repo> ls-files -- .env.secrets secrets.yaml scripts/
```

`check-ignore -v` prints the matching rule and stays silent for uncovered paths —
append those to the app repo's `.gitignore`, then re-run and confirm all four are
covered. **If any is still uncovered, stop and fix that before sealing.** If
`ls-files` shows a plaintext secrets file **already tracked**, `.gitignore` does
not protect it: say so loudly, treat those values as leaked and needing rotation,
and leave the decision (rotate now? rewrite history?) to the owner rather than
quietly `git rm --cached`-ing it.

Only then, from the directory holding `.env.secrets`:

```bash
kubectl create secret generic env --dry-run=client -o yaml \
    --from-env-file=.env.secrets \
  | kubeseal \
      --cert https://sealed-secrets.k8s-dev.hpc.uidaho.edu/v1/cert.pem \
      -o yaml --scope=cluster-wide \
  > <kubernetes-apps>/apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml
```

If the app has no `scripts/create-credential-secrets.sh` yet, offer to scaffold it
and the `scripts/.env.secrets` placeholder per `/deploy-my-application` Step 5b —
it makes the next re-seal reproducible by the owner without you.

**Never read, `cat`, `grep`, echo, or log `.env.secrets`** — you don't need to
see it to seal it, and never invent or reconstruct values. Verify the output as
in `/deploy-my-application` Step 5: `kind: SealedSecret` (if it says `Secret`,
plaintext escaped — delete immediately, tell the user, stop), non-empty
`encryptedData` with the expected key *count* (name keys if needed, never
values), `metadata.name: env`, and the `cluster-wide` annotation in both
places. Non-secret config changes belong in `deploy/base/config.yaml`, not the
SealedSecret. Remind the user that live has its own `env.yaml` sealed against
the live cert — a dev re-seal does not update live.

**Add a backing service / storage.** Copy `services/<svc>.yaml` /
`storage-volume.yaml` from the template, wire the `*_ENABLED` env and volume
mount, list everything in `kustomization.yaml` (check #6), and match the PVC
`claimName` (check #9). If the app has previews, say the cost out loud: every
backing service and PVC is duplicated per open preview.

**Add PR previews to an existing app.** Follow `/deploy-my-application`
Steps 1c and 4b in full (branch-scoping questions, label gate, ApplicationSet
placement, and **creating the `preview` label** — `gh label create preview
--repo <org>/<app>` after asking, since a repo that never had previews almost
certainly lacks it). On an existing app, start with **P1** — the AppProject
namespace wildcard is the thing apps deployed without previews almost never
have.

**Add SonarQube.** Follow `/deploy-my-application` Step 1b: scaffold
`sonar-main.yaml` + `sonar-project.properties` immediately with a placeholder
`sonar.projectKey`, keep going, and hand the owner the manual checklist
(project creation, `SONAR_TOKEN` secret, `SONAR_HOST_URL` variable).

**Promote to live.** Add/complete the `deploy/overlays/live/` tree from the
template with the real host, TLS, and a pinned image tag. Live's
`secrets/env.yaml` must be sealed separately against the **live** cert — never
copy the dev-sealed file.

## Step 6 — Verify, open the PR, and hand off

In this order: prove it locally, open the PR, tell the user to merge, then verify
on the cluster once they have.

### Step 6a — Verify locally, before the PR

- Re-run the Step 4 checklist rows touched by your change; state which
  invariants now hold that didn't before.
- Where possible, verify renders locally: `kustomize build
  apps/rcds/apps/<app>/deploy/overlays/dev` (and `overlays/dev` for the control
  plane) must succeed and contain the expected resources.

### Step 6b — Commit, push, and open the labeled PR

You commit and PR the **kubernetes-apps** work yourself, on the Step 4b branch.
You never merge it.

**The pre-commit gate.** You are committing automatically, so "check `kind:`
before a file moves" is a **stop condition**, not advice — after a push, anything
wrong is in the remote's history:

```bash
git -C <kubernetes-apps> status --short
git -C <kubernetes-apps> diff --cached --name-only
head -3 <kubernetes-apps>/apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml
```

1. **`secrets/env.yaml` says `kind: SealedSecret`.** If it says `Secret` — which
   a botched re-seal is exactly how you'd get — **do not commit**: delete it,
   tell the user, and stop.
2. **Every staged path is under `apps/rcds/apps/<app>/`** (plus
   `deploy-repo-secrets-<project>-live/` if promoting). Unstage anything else and
   say why — on a repair you have often been reading sibling apps, and a stray
   edit there is easy to make.
3. **No `.env.secrets` or plaintext `secrets.yaml` is staged.**

Stage explicitly — `git add apps/rcds/apps/<app>/` — **never `git add -A`**.
Then commit (message naming the app and the fix, no secret values, with this
session's attribution footer) and push:

```bash
git -C <kubernetes-apps> push -u origin fix/agent-fix-<login>-<MM>-<DD>-<YYYY>

gh pr create --repo ui-iids/kubernetes-apps \
  --base main --head fix/agent-fix-<login>-<MM>-<DD>-<YYYY> \
  --title "Fix <app> deployment: <short symptom or change>" \
  --label coding-agent --label fix-deploy \
  --body-file <body-file>
```

**PR body:** the diagnosis (what was broken, which invariant it violated, and
why it produced the reported symptom) or the update requested; the files changed;
the Step 6a verification result; if secrets were re-sealed, that it's sealed and
verified plus the **key count** — **never a value**; drift you noticed and
deliberately left alone; and any ordering blocker.

**If labeling fails** (label missing, or no permission), the PR is still open —
report it and hand over `gh pr edit <N> --add-label coding-agent --add-label
fix-deploy`. Don't close and recreate the PR to retry labels.

**Never merge it.** No `gh pr merge`, no `--admin`, no auto-merge, no push to
`main`.

### Step 6c — Hand off

- **Lead with the action they owe you**, as its own line:

  > **Action required: review and merge the PR in `ui-iids/kubernetes-apps`** —
  > `<PR URL>`. The fix isn't live until you merge it; ArgoCD syncs on merge.

  Then say what's blocked behind it: the cluster verification below, and any
  app-repo commit/PR they still owe — which you did **not** open for them.
- Summarize: the diagnosis (what was broken and why), the files changed in
  each repo, and any drift you noticed but deliberately left alone.
- **The two repos are handled differently.** kubernetes-apps is committed,
  pushed, and PR'd by you; the **app repo** (CI workflows, Dockerfiles,
  `scripts/`, `.gitignore`) is left uncommitted in their working tree and needs
  its own commit and PR from them. They never share a commit — and "the PR is
  open" must not be allowed to imply both halves are handled.
- Call out ordering blockers: a new service's image must build in CI before
  ArgoCD can pull it; a `preview` label may need creating
  (`gh label create preview`); live secrets still need their own seal.

### Step 6d — Verify on the cluster (after they merge)

- **Verify on the cluster once the fix has merged and ArgoCD has synced** — if
  Step 0 gave you a context. Re-running the checklist proves the manifests are
  right, not that the app recovered:
  1. `kubectl --context <ctx> get application -n argocd | grep <app>` → `Synced`
     / `Healthy`. `OutOfSync` means it hasn't merged or polled yet: wait.
  2. `kubectl --context <ctx> get pods -n apps-rcds-<app>` → every pod `Running`
     **and** `Ready`, restarts back at 0. See `/deploy-my-application` Step 6b for
     the pod-state → cause table (`ImagePullBackOff`,
     `CreateContainerConfigError`, `Pending`, `CrashLoopBackOff`, `0/1` ready…) —
     don't duplicate it here, and don't declare a repair done while any pod is in
     one of those states.
  3. `describe pod` / `get events --sort-by=.lastTimestamp` on anything unhealthy,
     and `logs --tail=200` (with `--previous` on a crash loop) where the cause
     isn't in the events. Never paste credential-shaped log content anywhere.
  4. Hit the dev site: `curl -fsSI https://<app>.k8s-dev.hpc.uidaho.edu`. For a
     repair, re-run the exact interaction the user reported as broken — the
     symptom is the test. Say plainly whether the original symptom is gone.

  If Step 0 came back without access, **say the fix is unverified on the cluster**
  and list what the user should check. Don't let a green `kustomize build` imply
  a working app.
- Run this when the user says they've merged, or offer it on a later invocation.
  Don't sit and poll for the merge.

## Guardrails

- Two write locations only: (1) in kubernetes-apps, `apps/rcds/apps/<app>/`
  (plus a root `deploy-repo-secrets-<project>-live/` dir if promoting);
  (2) in the app repo, `.github/workflows/`, `Dockerfile[.<service>]`, and
  `.gitignore`. Don't edit other apps or either template — if the fix seems to
  require a template change, stop and raise it instead.
- **Three sanctioned GitHub-side writes, and no others:** the kubernetes-apps
  branch (Step 4b), the labeled PR (Step 6b), and the `preview` label when
  adding previews to an app that lacks it. Still forbidden: **merging** any PR,
  pushing to `main`, changing repo settings, branching or committing in the
  **app repo**, and any write to a repo outside `ui-iids`/`ui-insight`.
- Plaintext secrets are write-only: stream `.env.secrets` into `kubeseal`,
  never read it back, never let a `kind: Secret` file reach the manifest repo.
  Check `kind:` before a file moves — and, since you now commit and push
  yourself, the Step 6b gate is the last place that check can save you.
- **The repo is the source of truth; the cluster is a diagnostic aid.** Grep
  sibling apps to settle conventions; read at least one healthy app of the same
  shape before declaring the target app wrong.
- **Cluster access: read-only, on the Step 0 context, never Secret contents.**
  Diagnosis is where cluster reads earn their keep — `get application -n argocd`,
  `get pods -n apps-rcds-<app>`, `describe pod`, `get events`, `logs
  [--previous]`, and `apply --dry-run=server` for schema checks. But:
  - **Use the context the user confirmed in Step 0**, passed explicitly as
    `--context <ctx>` on every command. Never fall back to the ambient current
    context, and never use a kubeconfig the user didn't approve — a diagnosis
    read off the wrong cluster is worse than no diagnosis.
  - If Step 0 found nothing or the user declined, **don't retry later** — run the
    repo-only audit and report what went unverified.
  - Read-only means read-only: fix manifests, not clusters. A live `patch` or
    `rollout restart` appears to work exactly until ArgoCD's next sync reverts
    it, and you will have hidden the real fault.
  - Prefer existence checks that cannot leak (`kubectl get secret <name>
    --ignore-not-found`). **Never** read a Secret's `data`/`stringData`, and
    avoid `-o jsonpath` on Secrets entirely — on Secrets the habit is the risk.
  - Never `exec` into any pod. If you need a tool a pod has (e.g. its exact
    `kustomize` version), fetch that version locally and test in a scratch dir.
- Repair mode changes nothing the user didn't sanction: diagnose, present, then
  fix. If a fix is destructive or ambiguous (renaming images, deleting
  resources, rescoping previews), confirm first.
