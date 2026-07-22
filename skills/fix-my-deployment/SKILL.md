---
name: fix-my-deployment
description: Diagnose and repair or update an EXISTING University of Idaho RCDS/IIDS deployment in the `ui-iids/kubernetes-apps` GitOps repo — audit its ArgoCD + Kustomize manifests against the canonical `apps/rcds/apps/deploy-template/` example and the app repo's CI, find what is broken or drifted, and fix it. Handles two cases — (1) REPAIR a broken deploy (ImagePullBackOff, pods pending, sync green but nothing running, previews never appearing), or (2) UPDATE a working deploy (new services, new env vars re-sealed with `kubeseal`, backing services, storage, adding PR previews or SonarQube, promoting to live). Use when the user says "fix my deployment", "my deployment is broken", "my pods won't start", "ArgoCD won't sync", "update my deployment", "add a service to my deployment", "add env vars to my deployment", "re-seal my secrets", "my preview builds aren't working", or invokes /fix-my-deployment.
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
- **Never write to a cluster** — no `apply`, `delete`, `patch`, or `exec`.
  Read-only cluster diagnostics are allowed under the rules in **Guardrails**
  (announced first, never Secret contents).
- **Never** put plaintext secrets in manifests, chat, or commit messages. Env
  secrets are Bitnami `SealedSecret`s sealed by piping `.env.secrets` through
  `kubeseal` without ever reading it (Step 5 of `/deploy-my-application`,
  restated below for updates).
- Fix what you were asked to fix. While auditing you will often notice *other*
  drift (missing AppProject wildcard, unsealed live secrets, template lag) —
  **report** those findings, but don't silently expand the change set without
  asking.

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

**From the cluster (optional, read-only, announced — see Guardrails):** ArgoCD
`Application` sync/health status, pod states and events in `apps-rcds-<app>`,
recent `kubectl describe` output on failing pods. These turn "something is
broken" into a named failure mode fast, but everything below is also
diagnosable from the two repos alone.

## Step 4 — The audit: run the invariants as a checklist

Walk every applicable row. Most failures here are **silent** — valid YAML,
green sync, wrong result — so absence of errors proves nothing.

| # | Check | Symptom when violated |
|---|-------|----------------------|
| 1 | Every `image:` in Deployments and `imageName:` in the ImageUpdater is **byte-for-byte** what a `build-*-container.yaml` publishes (`ghcr.io/<org>/<app>[-<service>]`) | `ImagePullBackOff` / `ErrImagePull` |
| 2 | The image actually exists in GHCR — CI has run and pushed for the referenced tag | `ImagePullBackOff` on a correct-looking string |
| 3 | One Deployment+Service per built image; labels and Ingress backend `service.name` consistent (`<app>` single-service, `<app>-<service>` multi) | 404/502 on the dev host while pods run fine |
| 4 | SealedSecret `metadata.name` (`env`) == `envFrom.secretRef.name` in every Deployment | Pods stuck — referenced Secret doesn't exist |
| 5 | `deploy/overlays/dev/secrets/env.yaml` is `kind: SealedSecret` with non-empty `encryptedData`, key count matching `.env.secrets` | App boots but can't see (new) env vars — the classic "I added a var and nothing changed": **the file must be re-sealed, it never updates itself** |
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
**wholesale** — you cannot append to it. The owner updates `.env.secrets` in
the app repo (all keys, old and new); then, from the app repo root:

```bash
kubectl create secret generic env --dry-run=client -o yaml \
    --from-env-file=.env.secrets \
  | kubeseal \
      --cert https://sealed-secrets.k8s-dev.hpc.uidaho.edu/v1/cert.pem \
      -o yaml --scope=cluster-wide \
  > <kubernetes-apps>/apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml
```

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
placement). On an existing app, start with **P1** — the AppProject namespace
wildcard is the thing apps deployed without previews almost never have.

**Add SonarQube.** Follow `/deploy-my-application` Step 1b: scaffold
`sonar-main.yaml` + `sonar-project.properties` immediately with a placeholder
`sonar.projectKey`, keep going, and hand the owner the manual checklist
(project creation, `SONAR_TOKEN` secret, `SONAR_HOST_URL` variable).

**Promote to live.** Add/complete the `deploy/overlays/live/` tree from the
template with the real host, TLS, and a pinned image tag. Live's
`secrets/env.yaml` must be sealed separately against the **live** cert — never
copy the dev-sealed file.

## Step 6 — Verify and hand off

- Re-run the Step 4 checklist rows touched by your change; state which
  invariants now hold that didn't before.
- Where possible, verify renders locally: `kustomize build
  apps/rcds/apps/<app>/deploy/overlays/dev` (and `overlays/dev` for the control
  plane) must succeed and contain the expected resources.
- Summarize: the diagnosis (what was broken and why), the files changed in
  each repo, and any drift you noticed but deliberately left alone.
- Tell the user the remaining manual steps: commit + PR to each affected repo
  **separately** (app repo CI changes and kubernetes-apps manifests never share
  a commit); ArgoCD picks up kubernetes-apps on merge. Do not commit or push
  unless explicitly asked.
- Call out ordering blockers: a new service's image must build in CI before
  ArgoCD can pull it; a `preview` label may need creating
  (`gh label create preview`); live secrets still need their own seal.

## Guardrails

- Two write locations only: (1) in kubernetes-apps, `apps/rcds/apps/<app>/`
  (plus a root `deploy-repo-secrets-<project>-live/` dir if promoting);
  (2) in the app repo, `.github/workflows/`, `Dockerfile[.<service>]`, and
  `.gitignore`. Don't edit other apps or either template — if the fix seems to
  require a template change, stop and raise it instead.
- Plaintext secrets are write-only: stream `.env.secrets` into `kubeseal`,
  never read it back, never let a `kind: Secret` file reach the manifest repo.
  Check `kind:` before a file moves — not after.
- **The repo is the source of truth; the cluster is a diagnostic aid.** Grep
  sibling apps to settle conventions; read at least one healthy app of the same
  shape before declaring the target app wrong.
- **Cluster access: read-only, announced, never Secret contents.** Diagnosis is
  where cluster reads earn their keep — `kubectl get application -n argocd`,
  `kubectl get pods -n apps-rcds-<app>`, `kubectl describe pod`, `kubectl get
  events`, `kubectl apply --dry-run=server` for schema checks. But:
  - **Announce before the first cluster command** — the user is thinking about
    repo files; unannounced `kubectl` against their live context is a surprise.
  - Prefer existence checks that cannot leak (`kubectl get secret <name>
    --ignore-not-found`). **Never** read a Secret's `data`/`stringData`, and
    avoid `-o jsonpath` on Secrets entirely — on Secrets the habit is the risk.
  - Never `exec` into any pod. If you need a tool a pod has (e.g. its exact
    `kustomize` version), fetch that version locally and test in a scratch dir.
- Repair mode changes nothing the user didn't sanction: diagnose, present, then
  fix. If a fix is destructive or ambiguous (renaming images, deleting
  resources, rescoping previews), confirm first.
