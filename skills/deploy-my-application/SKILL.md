---
name: deploy-my-application
description: Deploy a University of Idaho RCDS/IIDS application to our Kubernetes cluster by generating ArgoCD + Kustomize deploy manifests in the `ui-iids/kubernetes-apps` GitOps repo, modeled on the canonical `apps/rcds/apps/deploy-template/` example. Handles two cases — (1) deploy an EXISTING repo from the `ui-iids` or `ui-insight` GitHub orgs, or (2) scaffold and deploy a NEW project (pick a namespace, generate the full manifest tree). Also seals the app's dev env secrets into a Bitnami SealedSecret with `kubeseal`, piping `.env.secrets` through without ever reading it, and gitignores the plaintext in the app repo. Reaches the cluster read-only through either a local kubeconfig or a service account over SSH to a master node, and builds locally with Docker or Podman (defaulting to Podman). Does its kubernetes-apps work on a dated `deploy/agent-deploy-*` branch and opens a PR against `main` labeled `coding-agent` + `deploy-application` for the user to review and merge. Use when the user says "deploy my application", "deploy this repo", "deploy my app to the cluster", "set up deploy manifests", "create a namespace and deploy", "add my project to kubernetes-apps", "seal my secrets", "kubeseal my env file", "set up sealed secret scripts", "check my kubeconfig", "create the preview label", "validate my deployment", or invokes /deploy-my-application.
---

# Deploy my application

You generate the ArgoCD + Kustomize manifests that deploy a University of Idaho
**RCDS / IIDS** application onto our cluster via GitOps. You do **not** run
`kubectl apply` — deployment happens when manifests are committed to the
`ui-iids/kubernetes-apps` repo and ArgoCD syncs them.

The output must match our house pattern exactly. The canonical, fully-annotated
example lives at:

```
kubernetes-apps/apps/rcds/apps/deploy-template/
```

**Read that template and its `README.md` first, every time.** It is the source
of truth for file layout, naming, and the dev-vs-live differences. Your job is
to copy that structure and substitute the project-specific tokens.

## The two-repo model (read this before anything else)

Deploying an app spans **two** repos, and they connect only through matching
image names:

1. **The app repo** (`ui-iids/<app>` or `ui-insight/<app>`) — the source code +
   the CI that builds the container images. Its template is
   `github.com/ui-iids/deploy-template`, containing per service:
   - `.github/workflows/build-<service>-container.yaml` — builds & pushes to GHCR
   - `Dockerfile[.<service>]`

   Each workflow publishes `ghcr.io/<org>/<app>[-<service>]:main`
   (via `ghcr.io/${{ github.repository }}[-<service>]`).

2. **kubernetes-apps** — the deploy manifests (this is where you write). They
   reference `ghcr.io/<org>/<app>[-<service>]:main` and tell ArgoCD how to run it.

**The naming contract:** the app slug `<app>` is the app repo's name, and the
image string in the manifests must be *byte-for-byte* what the workflow builds.
Single-service apps use **no** suffix (`ghcr.io/<org>/<app>:main`); multi-service
apps use one image per service with a `-<service>` suffix and one Deployment per
service (worked example: `apps/rcds/apps/universo/`, backend + frontend).

So before writing manifests, know what the app repo's CI actually builds —
inspect its `.github/workflows/build-*-container.yaml`, or (Mode B) scaffold that
CI from `ui-iids/deploy-template` first.

## Scope and ground rules

- Apps come strictly from our two GitHub orgs: **`ui-iids`** and **`ui-insight`**.
- The GitOps repo is always `https://github.com/ui-iids/kubernetes-apps.git`.
  Manifests live under `apps/rcds/apps/<app>/`.
- ArgoCD control-plane objects (`AppProject`, `Application`, `ImageUpdater`,
  `ApplicationSet`) live in the `argocd` namespace; workloads live in
  `apps-rcds-<app>`.
- **Never** write to a cluster — no `apply`, `delete`, `patch`, or `exec`.
  Produce files, then let the user commit/PR them for ArgoCD. Step 5's
  `--dry-run=client` seal touches no cluster at all. Cluster **reads** are
  legitimate and expected — that's what Step 0 negotiates and Step 6b spends —
  but they stay read-only and never touch Secret contents; see the
  cluster-access guardrail.
- **Never** put plaintext secrets in manifests, chat, or commit messages. Env
  secrets are Bitnami `SealedSecret`s: you seal the **dev** values yourself
  (Step 5) by piping `.env.secrets` through `kubeseal` without ever reading it.
  Sealed output is encrypted and safe to commit. **Live** secrets stay empty
  until promotion.

## Step 0 — Establish cluster access

Do this **before** any other fact-finding. Several later steps (Step 6b's dev
verification above all) need read-only cluster access, and the one thing you must
never do is guess how the user reaches our cluster.

There are **two access points**, and which one a person has is not something you
can infer:

- **Mode 1 — a local kubeconfig** on their machine.
- **Mode 2 — a service account they SSH into a master node with**, running
  `kubectl` there. Some users have only this, and some are required to use it
  even when a local config exists.

### Discover local kubeconfigs

Read-only, merging nothing, writing nothing:

```bash
echo "KUBECONFIG=${KUBECONFIG:-<unset>}"
ls -1 ~/.kube/config ~/.kube/*.yaml ~/.kube/*.conf ~/.kube/configs/* 2>/dev/null
kubectl config get-contexts        # context + cluster + server, never credentials
```

**Never `cat` a kubeconfig**, and never echo, copy, or log a token, client key,
or client certificate out of one. Context names, cluster names, and server URLs
are fine to show the user; the credential fields are not — the same rule that
governs Secret contents in **Guardrails**.

### Ask which access point to use

**Always ask.** Finding a kubeconfig does not mean it's the one to use:

> *"How should I reach the dev cluster — one of these local kubeconfigs, or a
> service account on a master node?"*

- **Kubeconfig(s) found and the user picks one** → Mode 1.
- **Nothing found** → go to **Mode 2**, don't declare the cluster unreachable.
  The absence of `~/.kube/config` is not the absence of access; plenty of people
  here only ever reach the cluster through a service account.
- **The user names a service account, or says they're required to use one** →
  Mode 2, even if a local kubeconfig exists.
- **Neither is available** → then, and only then, proceed repo-only. Every
  repo-side step still works: manifest generation, sealing (the `--cert` fetch
  and `--dry-run=client` need no credentials), and all invariant checks. Say that
  Step 6b will be skipped and hand its checks back to the user.

### Mode 1 — a local kubeconfig

- **Exactly one found** — *"I found `<path>`, context `<ctx>`, server `<server>`.
  Can I use this read-only to reach the dev cluster (`k8s-dev.hpc.uidaho.edu`)?"*
  Wait for an actual yes.
- **More than one** — list them and **ask which to use**:

  | Path | Context | Cluster server |
  |------|---------|----------------|
  | `~/.kube/config` | `k8s-dev` | `https://…k8s-dev.hpc.uidaho.edu` |
  | `~/.kube/prod.yaml` | `k8s-prod` | `https://…` |

  **Do not pick by name.** A context called `dev` may be a personal kind/minikube
  cluster, and our `k8s-dev` may be buried in a file called `config`. Only the
  user knows which is which.

Pass the context explicitly on every command rather than relying on the current
context, so nothing depends on ambient state the user might switch mid-session.

### Mode 2 — a service account over SSH

`kubectl` runs **on the master node**, reached as the service account. Ask for
the two things you must not guess:

1. **The master node** — hostname or IP. Ask every time; never assume one, since
   it differs per cluster.
2. **The service account username** to SSH as.

**Never ask for, echo, store, or write a password.** Don't put credentials in a
command line, a file, or the shell history, and if the user volunteers one, don't
repeat it back.

**Probe how authentication works** — this decides whether you can act at all:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 <sa>@<host> true
```

- **Exit 0 → key-based auth.** You can run remote commands yourself:
  `ssh <sa>@<host> kubectl …`
- **Non-zero → a password would be prompted.** **Do not attempt an interactive
  `ssh`** — the prompt blocks with no way for you to answer it and hangs the
  session. Instead, hand the user the exact command to run themselves (they can
  prefix it with `!` in this session so the output lands in the conversation) and
  work from what they paste back. Say plainly that this makes cluster checks a
  back-and-forth rather than something you can do alone — it changes how the rest
  of the run feels, and they should hear it once, up front.
- **Host unreachable / DNS failure** → almost always off the campus network or
  VPN. Say that and stop; don't retry in a loop.

**Remote execution is narrow, and these are hard rules:**

- The **only** thing you run on the master node is read-only `kubectl`. No file
  edits, no `scp`, no installs, no looking around — it is shared infrastructure,
  not a workspace.
- **Don't pass `--context`** to a remote `kubectl` unless the user names one;
  contexts are a local-kubeconfig concept and the node has its own.
- **All repo work stays local.** Manifests are authored and committed on the
  user's machine; nothing is copied to the node.
- **Never `cat` a kubeconfig on the master either.** The credential rule applies
  identically on the remote side.

### Verify the chosen access point works

Do this **before** relying on it for anything. Connecting is not the same as
having permission:

```bash
<KUBECTL> cluster-info
<KUBECTL> get ns --no-headers | head        # does apps-rcds-* exist?
```

- **Mode 1**: if the server isn't a `k8s-dev.hpc.uidaho.edu` endpoint, stop and
  re-ask — a kubeconfig aimed at production is not a substitute.
- **Mode 2**: a `Forbidden` is a permissions finding to **report**, not something
  to work around. A missing `apps-rcds-*` namespace is itself informative
  (manifests never merged, or the wrong cluster) — establish which before
  concluding anything from it.

### Use only what was chosen

Once an access point is settled, **use only it.** If it fails — SSH down,
`Forbidden`, expired credential — **stop and report**. Do **not** quietly try a
local kubeconfig because the service account failed, or vice versa.

The reason matters, so it doesn't get "simplified away" later: the two access
points are **different identities with different RBAC**. A check that fails as
the service account and passes as the user's personal context has not been
answered — it has been answered about the wrong person, and a confident "looks
healthy" from the wrong identity is worse than no answer at all.

A **"no" to cluster access is final for the session.** Proceed repo-only and say
explicitly, at handoff, which checks went unverified. Cluster access is read-only
in **both** modes: SSH onto a master node is more power, not more permission —
see **Guardrails**.

### The resolved command form: `<KUBECTL>`

Everything after this step writes `<KUBECTL>`. Substitute whichever form Step 0
resolved to:

| Mode | `<KUBECTL>` |
|------|-------------|
| Local kubeconfig | `kubectl --context <ctx>` |
| Service account (key auth) | `ssh <sa>@<host> kubectl` |
| Service account (password auth) | the same, handed to the user to run |

## Step 0b — Choose the container runtime

Resolve this **before any container command**. Both Docker and Podman are in use
here, and a hardcoded `docker` simply fails for half the team.

**Detect what's installed:**

```bash
command -v podman docker
podman --version 2>/dev/null; docker --version 2>/dev/null
```

- **Both present** → ask: *"Docker or Podman for local container work?"*
- **Only one present** → use it and say which. Don't ask a question with one
  possible answer.
- **Neither** → local container validation can't run. Say so, skip it, and carry
  the gap into the handoff rather than letting silence imply it passed.

**Default to Podman** when the user expresses no preference, *or* when nothing
locally suggests the project already uses Docker — meaning there are no existing
local containers or images for this app to go by:

```bash
docker ps -a --format '{{.Image}}' 2>/dev/null | grep -i <app>
docker images --format '{{.Repository}}' 2>/dev/null | grep -i <app>
```

Empty results (or `docker` erroring outright) mean there's no local Docker state
to match, so **Podman is the default**. Say that you defaulted and why, so the
user can override it in one word.

**Verify the Podman CLI actually works** before committing to it — installed is
not the same as usable, and this is where people lose time on macOS:

```bash
podman info >/dev/null 2>&1 || echo "podman not ready (machine likely not running)"
```

On macOS, Podman runs inside a VM. If `podman info` fails, the fix is
`podman machine start` (or `podman machine init` on a fresh install). **Give the
user the command; don't start their VM for them** — it's their machine and their
resources. If they'd rather not, offer Docker instead of stalling.

**Command mapping.** Later steps write `<RUNTIME>` where either works;
where a command has no drop-in equivalent, translate with this table:

| Task | Docker | Podman |
|------|--------|--------|
| Build | `docker build` | `podman build` |
| Compose up | `docker compose up -d` | `podman compose up -d` (Podman ≥ 4.4) or `podman-compose up -d` |
| Status | `docker compose ps` | `podman compose ps` / `podman ps -a` |
| Logs | `docker compose logs <svc>` | `podman compose logs <svc>` |
| Tear down | `docker compose down` | `podman compose down` |
| Inspect a remote image | `docker manifest inspect <ref>` | `skopeo inspect docker://<ref>` |

Two notes that save a wrong turn:

- `podman compose` shells out to `podman-compose` or `docker-compose`. If it
  reports no provider, try `podman-compose` directly; if neither exists, build
  and run per service rather than declaring failure.
- `podman manifest inspect` is **not** the analogue of `docker manifest inspect`
  — it inspects *local* manifest lists. For "does this tag exist in GHCR?", use
  `skopeo inspect docker://…`, or `gh api`, which needs no container runtime at
  all.

**Never switch runtimes mid-run.** Half the stack built in one and half in the
other produces "it built but there's no container" confusion that costs far more
than asking. If the chosen runtime breaks, say so and ask before switching.

## Step 1 — Determine the mode

**Mode A — Deploy an existing repo.** The user names (or you're working inside)
a repo already in `ui-iids`/`ui-insight`. Confirm:
- the GitHub org (`ui-iids` or `ui-insight`) and repo name → this is `<app>`,
- **what its CI builds** — read `.github/workflows/build-*-container.yaml` in the
  app repo to get the exact image name(s) and service count (one service vs.
  backend+frontend etc.). This is the authoritative source of the image strings.
  If no build workflow exists, the image pipeline is a prerequisite — offer to
  scaffold it from `ui-iids/deploy-template` (Mode B's step) before continuing,
- the port each service listens on, and any backing services / persistent storage.

**Mode B — New project + deploy.** The user is starting fresh. In addition to
the above:
- settle on the app slug `<app>` (kebab-case, = the eventual repo name) and
  confirm the namespace will be `apps-rcds-<app>`,
- **scaffold the app repo's CI** from `ui-iids/deploy-template`: a
  `.github/workflows/build-<service>-container.yaml` and a `Dockerfile[.<service>]`
  per service, each publishing `ghcr.io/<org>/<app>[-<service>]:main`. The
  manifests you write must reference those same image names.

If any of org, app name, service list, listen port(s), or storage/back-end needs
are unknown, **ask** before generating — these drive the substitutions.

## Step 1b — Ask about SonarQube (non-blocking)

SonarQube code-quality scanning is **opt-in**. Ask the project owner once,
early: *"Do you want SonarQube set up for this project?"* If they decline, skip
both Sonar files entirely and move on (deployment doesn't depend on it).

If they say yes, give them **both links up front** — they will need each one:

- **How-to (start here):**
  <https://knowledgebase.k8s-dev.hpc.uidaho.edu/index.php/Adding_Sonarqube_to_a_repository>
- **Our SonarQube instance:** <https://sonarqube.k8s-dev.hpc.uidaho.edu/>

Then **do not wait** on the manual setup — it happens in two web UIs you cannot
drive, and blocking the deploy on it buys nothing. Instead:

1. **Scaffold immediately.** Copy `.github/workflows/sonar-main.yaml` +
   `sonar-project.properties` from `ui-iids/deploy-template` into the app repo,
   leaving `sonar.projectKey` as a clearly-marked placeholder
   (`sonar.projectKey=TODO-paste-key-from-sonarqube`). Adjust
   `sonar.sources` / `sonar.tests` / language + coverage settings to the repo's
   layout, and the build/test steps in `sonar-main.yaml` to its stack.
2. **Keep going** with the rest of the deploy (manifests, etc.). The Sonar scan
   simply won't pass until the owner finishes the steps below — that's harmless.
3. **Hand the owner these steps.** Only they can do them — you can't create the
   project or set GitHub secrets for them. Give the whole sequence, not a
   summary; the first-time flow is where people get stuck:

   1. Sign in at <https://sonarqube.k8s-dev.hpc.uidaho.edu/> with campus SSO.
      No account? That's a request to RCDS, not something the UI self-serves.
   2. **Projects → Create project → Manually.** Use the repo name for both the
      display name and the project key (e.g. `my-app`). SonarQube appends a
      UUID, so the key you get back is *not* what you typed — it looks like
      `ui-iids_my-app_c4489195-…`. Copy it exactly.
   3. Set the **new code definition** when prompted — *Previous version* is the
      right default for our repos.
   4. Choose **With GitHub Actions** as the analysis method. The page shows the
      exact secret/variable names and a sample workflow; our scaffolded
      `sonar-main.yaml` already matches it, so you only need the values.
   5. **Generate a token** when offered (*Account → Security → Generate token*
      if you skipped past it). Scope it to the project, and copy it now — the
      value is shown **once** and is unrecoverable afterward.
   6. In the app repo on GitHub: **Settings → Secrets and variables → Actions**.
      - **Secrets** tab → *New repository secret* → name `SONAR_TOKEN`, value =
        the token from step 5.
      - **Variables** tab → *New repository variable* → name `SONAR_HOST_URL`,
        value `https://sonarqube.k8s-dev.hpc.uidaho.edu` (no trailing slash).

        `SONAR_HOST_URL` is a **variable**, not a secret — putting it in the
        wrong tab leaves the workflow reading an empty `vars.SONAR_HOST_URL`
        and defaulting to sonarcloud.io, which fails with a confusing auth
        error.
   7. Paste the project key from step 2 into `sonar.projectKey` in
      `sonar-project.properties` — or hand it to you and you'll set it.
   8. Push to `main` (or re-run the workflow) and confirm the analysis lands on
      the project dashboard.

**The two failures you'll be asked about:**

- **Wrong or placeholder `sonar.projectKey`** → the workflow fails with a
  "project not found" / "could not find a default branch" style error. It is
  almost always the missing UUID suffix: the key SonarQube generated is not the
  name you typed.
- **Coverage reports at 0%** → not an error, and the scan stays green. It means
  `sonar.<lang>.coverage.reportPaths` points somewhere the test step didn't
  write. Confirm the workflow actually runs the tests *before* the scan step and
  that the report path matches (e.g. `coverage.xml`, `lcov.info`).

## Step 1c — Ask about PR preview builds

PR preview builds — a throwaway deploy per open pull request — are **opt-in**,
like SonarQube.

**First, know where the file goes.** An app has *two* overlay trees and they are
easy to confuse:

| Tree | Contains | PR builds? |
|------|----------|------------|
| `apps/rcds/apps/<app>/overlays/dev/` | **control plane** — `Application`, `AppProject`, `ImageUpdater`, `ApplicationSet` (all in ns `argocd`) | ✅ **here** |
| `apps/rcds/apps/<app>/deploy/overlays/dev/` | **workloads** — Deployments, Ingress, secrets, image pins | ❌ never |

`pull-request-builds.yaml` is an `ApplicationSet` — a control-plane object — so it
lives at `<app>/overlays/dev/pull-request-builds.yaml` and is listed in
`<app>/overlays/dev/kustomization.yaml`. Putting it under `deploy/overlays/dev/`
gets it applied into the app's own namespace, where nothing reads it: it syncs
green and silently does nothing.

**How previews are gated (the house pattern).** The `pullRequest` generator in
the template is gated on a **label**, not a branch:

```yaml
github:
  owner: ui-iids
  repo: <app>
  appSecretName: creds-github-<org>   # creds-github-ui-iids | creds-github-ui-insight
  labels:
    - preview                          # PR must carry this label
```

Only PRs labeled `preview` get an environment. That is the whole gate in every
app in the repo today — **the template ships no branch filter at all.** So don't
go reading the filter field off the template (it isn't there); if you want branch
scoping you are *adding* a block, not editing one.

**Ask the owner three things, in this order:**

**1. Do they want previews at all?** *"Do you want PR preview builds — a
temporary deploy spun up per open pull request labeled `preview`?"* Say the cost
out loud when they answer: each preview is a **full stack** — every service,
every backing service, every PVC — duplicated per open PR until it closes.

**2. Which PRs get one?** **Do not offer `main` as a presumed default.** Look at
the repo's actual branching model *before* asking, so the question is concrete:

```bash
gh repo view <org>/<app> --json defaultBranchRef -q .defaultBranchRef.name
gh api repos/<org>/<app>/branches --jq '.[].name'      # is there a long-lived test?
gh pr list --repo <org>/<app> --state all --limit 20 \
   --json number,headRefName,baseRefName                # where do PRs actually go?
```

Then present the two shapes that exist in our fleet:

- **A long-lived `test` branch exists** → features merge `feature/* → test`, and
  `test → main` is the promotion PR. Previewing **PRs into `test`** is almost
  always what people mean: it previews the day-to-day work. Previewing **PRs into
  `main`** on such a repo yields previews for exactly one PR — the promotion —
  which is a legitimate but very different choice (a last look at the release
  candidate). Ask which they want; don't assume the common one.
- **No `test` branch** → every feature PR targets `main`, so `* → main` is the
  only option and *every* PR gets a full stack. Restate the cost here — this is
  the configuration that surprises people on a busy repo.

Restate their answer as an explicit arrow (`feature/* → test`, or `test → main`)
and get confirmation before generating — "PRs into X" and "PRs from X" are
opposite ends of the arrow and users mix them up constantly.

**3. Do they want a walkthrough?** *"Want a short walkthrough of how to use
preview builds?"* If yes, see **Teach the workflow** below.

**If they decline previews**, omit `overlays/dev/pull-request-builds.yaml` and its
entry in `overlays/dev/kustomization.yaml`. Nothing else changes. Don't create the
label. Mention in the Step 8 handoff that previews were skipped and can be added
later with `/fix-my-deployment`.

**If they accept**, generate `pull-request-builds.yaml` from the template with the
generator pointed at `<org>/<app>`, the `preview` label gate kept, and the PR host
pattern `<app>-pr-{{.number}}.k8s-dev.hpc.uidaho.edu` — **and create the label now**.

### Create the `preview` label (only when previews are accepted)

The gate is a label, so a repo without that label has previews that are wired
correctly and do nothing — and a missing label is indistinguishable from
"previews are broken": the generator matches zero PRs and reports no error
anywhere. Don't defer this to the handoff; a written-down instruction is a label
that never gets created.

This writes to the user's GitHub repo, so **ask before running it**:

```bash
gh label list --repo <org>/<app> --search preview          # check first
gh label create preview --repo <org>/<app> \
  --description "Spin up a PR preview environment" --color 0E8A16
```

- **Already exists** → say so and move on; never `--force` an existing label
  (someone may be using it for something else — if its description suggests a
  different purpose, raise that instead of overwriting).
- **`gh` not installed or not authenticated**, or the user lacks write access →
  don't improvise. Hand them the `gh label create` line above, or the equivalent
  UI path (*Issues → Labels → New label*), and note it in the Step 8 handoff as
  an outstanding blocker rather than a footnote.

### Teach the workflow (only if they asked for a walkthrough)

Put this in the Step 8 handoff, and offer to drop it in the app repo as
`PREVIEW-BUILDS.md` so it outlives the conversation:

1. **Open the PR** against the branch previews are scoped to — state the arrow
   you configured (`feature/* → test`), because a PR against the *other* branch
   gets nothing and looks broken.
2. **Add the `preview` label** — `gh pr edit <N> --add-label preview`, or the
   sidebar in the GitHub UI.
3. **Wait for CI.** The build workflows must push an image tagged
   `pr-<N>-<8-char-sha>` before anything can start. Watch it with
   `gh pr checks <N> --repo <org>/<app>`.
4. **ArgoCD picks it up** within a couple of minutes of the image existing, and
   creates a namespace `apps-rcds-<app>-pr-<N>` with the whole stack in it.
5. **Visit** `https://<app>-pr-<N>.k8s-dev.hpc.uidaho.edu`.
6. **Tear down** by removing the label or closing/merging the PR. The namespace
   and everything in it is pruned automatically — nothing to clean up by hand.

Tell them what "nothing happened" looks like, in the order worth checking:
is the label actually on the PR? did the build workflow run and go green for
*this* PR (not just for `main`)? does the `pr-<N>-<sha>` tag exist in GHCR? is
the PR's target branch the one previews are scoped to? Only after all four does
it become a manifest problem — at which point `/fix-my-deployment` has the full
preview invariant checklist.

**Branch scoping (only if they asked for it).** Add a `filters:` list as a sibling
of `github:` under `pullRequest:`. The two field names are easy to swap and mean
opposite things:

- `branchMatch` — the **source** (head) branch, where the code comes *from*
- `targetBranchMatch` — the **destination** (base) branch, where it merges *into*

```yaml
filters:
  # "PRs into test"           -> feature/* -> test
  - targetBranchMatch: "^test$"
  # "PRs from test into main" -> the promotion PR only
  - branchMatch: "^test$"
    targetBranchMatch: "^main$"
```

Entries in the list are **OR**'d; conditions *within* one entry are **AND**'d. The
label gate applies on top (label AND filter). Anchor the regexes (`^…$`) —
`main` unanchored also matches `maintenance-fix`.

`filters` needs ArgoCD ≥ 2.12; we run 3.3.x, so it's available. If you need to
confirm the running version, see the cluster-access note in **Guardrails**.

**Cross-repo contract — check this or previews never start.** The app repo's build
workflows must actually fire for the PRs you just scoped. In each
`build-*-container.yaml`, `on.pull_request.branches` filters by the PR's **target**
branch. If previews are scoped to `test → main` but CI only builds
`pull_request: branches: [test]`, no image is ever pushed for that PR and every
preview pod sits in `ImagePullBackOff`. The workflow's branch list must include
the target branch of every PR you want previewed.

## Step 2 — Gather the substitution values

Collect this table for the project (see the template README's substitution
reference):

| Token | Source |
|-------|--------|
| `<app>` | repo / project slug (kebab-case) |
| `<org>` | `ui-iids` or `ui-insight` |
| `apps-rcds-<app>` | derived — ArgoCD project + namespace |
| image(s) | **exactly** what the app repo's `build-*-container.yaml` builds: `ghcr.io/<org>/<app>:main` (single) or `ghcr.io/<org>/<app>-<service>:main` per service (multi) |
| services | one Deployment+Service per built image (e.g. backend, frontend) |
| container port | each service's listen port (e.g. 8000) — **confirmed by Step 2b** when the smoke build runs; an inferred port is the most common cause of a green sync with a dead Ingress |
| dev host | `<app>.k8s-dev.hpc.uidaho.edu` |
| live host + TLS | only if promoting to prod (real domain) |
| backing services | none / mongodb / redis / postgres / mariadb |
| persistent storage | does the app need its own PVC? |
| PR previews? | **ask** (Step 1c) — include the ApplicationSet or not |
| PR scope | PR previews only — the confirmed source→target arrow (Step 1c). Label-gated by default; add `filters:` only if they asked for branch scoping. Never assume `main`. |

## Step 2b — Smoke-build the stack locally (optional)

The values you just collected — container port above all — are about to be
written into several files. A short local build confirms them from the running
thing instead of from a Dockerfile that may be out of date.

**Ask:**

> *"Want me to build and start the stack locally first? It takes a few minutes,
> and it confirms the ports and health-check paths I'm about to write into the
> manifests instead of me inferring them."*

State the trade plainly: a few minutes now, against a wrong port baked into a
Deployment, a Service, and an Ingress and not discovered until Step 6a. Use the
runtime resolved in **Step 0b**.

**Default the recommendation to what the repo actually supports:**

- **A compose file or `Dockerfile` exists** → recommend yes.
- **Mode B (new project) with nothing built yet** → skip it; there is no stack to
  build. Say so rather than asking a question with no possible answer.
- **No container runtime at all** (Step 0b found neither) → skip, and say that
  Step 6a will be skipped for the same reason, so **nothing** in this run gets
  local verification. That's worth hearing once, early.

**What it is — a smoke build, deliberately not the full pass:**

```bash
<RUNTIME> compose build && <RUNTIME> compose up -d     # or per-service build
<RUNTIME> compose ps
curl -fsS localhost:<port>/                            # and any documented health path
```

**What to harvest, and feed back into Step 2's table:**

| Token | What the running stack tells you |
|-------|----------------------------------|
| container port | the port actually bound and answering — not just what `EXPOSE` claims |
| health path | the path that returns 200, which becomes the readiness probe |
| services | one container per image the manifests will reference |
| backing services | which of Postgres / Redis / Mongo the app genuinely needs to boot |

**When observation and assumption disagree, the observation wins** — and say so
out loud. It usually means the Dockerfile drifted or the user is remembering an
older version, and that is worth them knowing regardless of the deploy.

**Tear down when done** (`<RUNTIME> compose down`). A stack left running holds
the ports Step 6a needs.

**This is a smoke build, not the validation.** It answers "does it build, and
what does it expose?" It does **not** replace Step 6a's crash-loop check, its
backing-service exercise, or its test-suite run. A green Step 2b never stands in
for Step 6a — if nothing has changed since, Step 6a's *build* may come back fast
off the layer cache, but every functional check still runs.

### Declining and failing are not the same thing

- **The user declines** → proceed. Author the manifests from the repo instead
  (`EXPOSE`, compose `ports:`, the app's own config), and **record that the
  values were not locally verified** — it goes in the Step 7 PR body and the
  Step 8 handoff, so a reviewer knows the port is inferred rather than observed.
- **No runtime available** → same: proceed, values inferred, flagged.
- **The build fails** → this is a finding, not a formality. Report it and fix the
  app-side problem **before generating manifests**: a stack that can't build
  won't deploy, and the manifests would otherwise encode guesses about something
  that doesn't run. A failed build is **not** waved through to Step 7.

|  | Manifests | PR (Step 7) |
|---|---|---|
| Declined | proceed, values inferred | allowed, flagged as unverified |
| No runtime | proceed, values inferred | allowed, flagged |
| **Build failed** | **stop, fix the app first** | **blocked until Step 6a passes** |

The principle, since these get conflated: **not looking is a choice the user is
allowed to make; looking and seeing it broken is not something to route around.**

## Step 3 — Decide what to include

Use the template as the maximal set and trim per the README's "Choosing what to
include":

- **Stateless, no deps** → drop `storage-volume.yaml`, `services/`, and the
  volume wiring in `deployment.yaml`; remove those from `deploy/base/kustomization.yaml`.
- **App-owned files** → keep `storage-volume.yaml` + the volume mount.
- **Needs a DB/cache** → keep/rename `services/<svc>.yaml` and set the matching
  `*_ENABLED` env to `"True"`.
- **PR previews** (Step 1c) → **no** = omit `overlays/dev/pull-request-builds.yaml`
  *and* its entry in `overlays/dev/kustomization.yaml`; **yes** = keep it (control-plane
  overlay, not `deploy/overlays/dev/`), label-gated, plus `filters:` if they asked
  for branch scoping. Never decide this by inference — it's an explicit question
  with a cost attached: **each preview is a full stack** — every service, every
  backing service (Postgres/Redis/…), and every PVC in the tree, duplicated per
  open PR and live until it closes. Say that cost out loud when they choose.
- **Dev only** → omit the whole `deploy/overlays/live/` tree (add later on
  promotion); for prod, include it with the real host, TLS, and a pinned image
  tag.
- **Secrets are never trimmed.** Every app keeps
  `deploy/overlays/dev/secrets/{env.yaml,kustomization.yaml}` and the `envFrom`
  block in `deployment.yaml`, even with no secrets today — an app with an empty
  `encryptedData: {}` is valid, and adding the wiring back later is fiddly.

## Step 3b — Create the working branch in kubernetes-apps

**Do this before writing a single file into kubernetes-apps.** Everything from
Step 4 onward — the manifest tree, and the sealed `env.yaml` in Step 5d — lands
in this repo, and it all belongs on a dedicated branch that you will open a PR
from in Step 7.

**Branch name:** `deploy/agent-deploy-<login>-<MM>-<DD>-<YYYY>`

Resolve the two tokens rather than guessing:

```bash
gh api user --jq .login     # GitHub login, e.g. Jarred6068 — lowercase it
date +%m-%d-%Y              # zero-padded, e.g. 09-03-2026
```

If `gh` is unauthenticated or unavailable, fall back to `git config user.name`
lowercased with spaces replaced by hyphens (the convention this repo's other
skills use), and **say which source you used**. A branch named off the wrong
identity is confusing but harmless; silently guessing is not.

Then, in the kubernetes-apps checkout:

```bash
git -C <kubernetes-apps> fetch origin
git -C <kubernetes-apps> switch -c deploy/agent-deploy-<login>-<MM>-<DD>-<YYYY> origin/main
```

- **Branch from an up-to-date `origin/main`**, not from whatever the local
  checkout happens to be sitting on. A stale base produces a PR full of diff
  that isn't yours, and reviewers will bounce it.
- **Never commit to `main`.** If the repo is already on a non-`main` branch with
  uncommitted work in it, **stop and ask** — that's likely someone else's
  in-progress change, and branching on top of it drags their work into your PR.
- **If the branch already exists** (a second run the same day), don't force
  anything. Ask whether to continue on it — usually right, since it's the same
  user, same day, often the same app — or to start a fresh one with a `-2`
  suffix.
- **Only kubernetes-apps gets a branch.** App-repo edits (CI workflows,
  `Dockerfile`s, `scripts/`, `.gitignore`) stay on whatever branch the user has
  checked out, and you never commit them — see Step 8.

Nothing is committed here. This step only establishes where the writes land.

## Step 4 — Generate the manifest tree

Create, under `apps/rcds/apps/<app>/`, the same files as `deploy-template/`,
with every `🔧 SUBSTITUTE` token replaced. Keep the annotations light/relevant —
you don't need to carry over the teaching comments, but you must preserve the
exact structure and naming. Verify these invariants:

- `AppProject`, `Application`, `ImageUpdater` names all share the
  `apps-rcds-<app>` prefix and reference each other consistently
  (`Application.spec.project` == `AppProject.metadata.name`).
- `Application.spec.source.path` points at `.../deploy/base`; the dev overlay
  patch re-points it at `.../deploy/overlays/dev`.
- **Image match:** every `image:` in the Deployments and every `imageName:` in
  the ImageUpdater is *byte-for-byte* what the app repo's
  `build-<service>-container.yaml` publishes (`ghcr.io/<org>/<app>[-<service>]`).
  A mismatched suffix = ArgoCD pulls a non-existent image.
- One Deployment+Service per built service image. For a single-service app the
  Service/Deployment `app:` label and the Ingress backend `service.name` equal
  `<app>`; for multi-service, name them per service (e.g. `<app>-backend`).
- The standalone PVC `claimName` in the Deployment matches `storage-volume.yaml`.
- **Secret wiring** — the SealedSecret's `metadata.name` (`env`) matches the
  Deployment's `envFrom.secretRef.name`. A mismatch here fails the same way a bad
  image string does: pods never start, because the referenced Secret doesn't
  exist. The dev overlay's `kustomization.yaml` must still list `secrets`, and
  `secrets/kustomization.yaml` must still list `env.yaml` — both come free with
  the template copy, so **verify these rather than rewrite them**.
- Overlay relative paths to the shared `deploy-repo-secrets-*` dir have the right
  depth (`../../../../../../../` from `deploy/overlays/<env>`).
- Hosts: dev = `<app>.k8s-dev.hpc.uidaho.edu`; PR = `<app>-pr-{{.number}}...`.
- **PR previews, only if included** → every invariant in Step 4b.

## Step 4b — PR preview invariants (only if previews are included)

Previews fail in ways the normal deploy never does, and **every failure below is
silent** — valid YAML, green sync, wrong result. Walk this list explicitly.

- **AppProject namespace wildcard — the hard blocker.** Previews deploy to
  `apps-rcds-<app>-pr-<N>`, but `base/project.yaml` restricts destinations to the
  single namespace `apps-rcds-<app>`. It **must** carry a trailing `*`:

  ```yaml
  destinations:
    - name: "in-cluster"
      namespace: "apps-rcds-<app>*"   # * admits the -pr-<N> namespaces
  ```

  Without it ArgoCD rejects every preview Application as an off-project
  destination. Every PR-enabled app in the repo has the `*`; apps generated
  without previews often don't — so **if you are adding previews to an existing
  app, this is the first thing to check.**

- **Every service's image must build for every previewed PR.** The ApplicationSet
  pins *all* images to `pr-<N>-<sha>` at once. If any one
  `build-*-container.yaml` carries a `paths:` filter on its `pull_request`
  trigger, a PR touching none of those paths pushes no image for that service —
  and that Deployment alone lands in `ImagePullBackOff` while everything else
  comes up healthy. Either drop the `paths:` filter from `pull_request` on every
  service, or don't preview. (A `paths:` filter on `push` is fine and worth
  keeping.)

- **Short-SHA length is a cross-repo contract.** ArgoCD's `{{.head_short_sha}}` is
  **8** characters. `docker/metadata-action` defaults to **7** — a 7-vs-8
  mismatch means the tag ArgoCD asks for never exists. Each workflow must set:

  ```yaml
  env:
    DOCKER_METADATA_SHORT_SHA_LENGTH: 8   # match ArgoCD's head_short_sha
    DOCKER_METADATA_PR_HEAD_SHA: true     # PR head SHA, not the merge-commit SHA
  ```

  and emit `type=ref,event=pr,suffix=-{{sha}}`. `DOCKER_METADATA_PR_HEAD_SHA`
  matters just as much: without it the tag carries GitHub's synthetic merge commit
  SHA, which `head_short_sha` never equals. (ArgoCD also exposes
  `{{.head_short_sha_7}}` if a workflow is pinned to 7.)

- **Digest pins do not defeat the tag override — don't "fix" this.** Once
  argocd-image-updater has run, `deploy/overlays/dev/kustomization.yaml` holds both
  `newTag: main` *and* `digest: sha256:…`, which kustomize renders as
  `image:tag@digest` — and runtimes resolve by digest, ignoring the tag. It looks
  like previews must run `main`. They don't: ArgoCD applies its `kustomize.images`
  override by shelling out to `kustomize edit set image`, which **replaces the
  whole entry and drops the digest**, leaving a clean `:pr-<N>-<sha>`. Verified on
  kustomize v5.8.1. Leave the overlay alone.

- **Patch every host-specific ConfigMap value.** Anything in `deploy/base/config.yaml`
  holding the dev hostname — webhook/callback URLs, `CORS_ORIGINS`, OAuth redirect
  URIs, self-referential API base URLs — still points at the **shared dev
  deployment** inside a preview. A preview then invites an external service to post
  callbacks into shared dev: cross-environment traffic that looks like a preview
  bug and isn't. Grep the ConfigMap for the dev host and add a JSON-6902 patch per
  hit, alongside the Ingress host patch:

  ```yaml
  - target: { version: v1, kind: ConfigMap, name: <app>-config }
    patch: |-
      - op: replace
        path: /data/WEBHOOK_URL
        value: https://<app>-pr-{{.number}}.k8s-dev.hpc.uidaho.edu/api/v1/...
  ```

  (Only works because the ConfigMap is a plain resource with a stable name — a
  `configMapGenerator` would hash-suffix it and the patch target wouldn't match.)

- **The env SealedSecret must be `cluster-wide`.** A namespace-scoped SealedSecret
  will not decrypt in `apps-rcds-<app>-pr-<N>`, so every pod hangs waiting on a
  Secret that never materializes. Step 5's `--scope=cluster-wide` already gives you
  this — the point is **don't tighten it** to namespace scope on a preview-enabled
  app.

- **`appSecretName` matches the org:** `creds-github-ui-iids` or
  `creds-github-ui-insight`. Wrong org = the generator silently produces zero
  Applications (no error, just nothing).

- **Names stay under limits.** Namespace `apps-rcds-<app>-pr-<N>` and DNS label
  `<app>-pr-<N>` each cap at 63 chars. A long `<app>` can blow the label at
  two-digit PR numbers.

## Step 5 — Seal the dev env secrets

The app's env vars reach the pod via `envFrom: secretRef: name: env` in
`deploy/base/deployment.yaml`, supplied by the SealedSecret at
`deploy/overlays/dev/secrets/env.yaml`. The template ships that file empty
(`encryptedData: {}`); you replace it wholesale with the sealed output. The
kustomization wiring is inherited from the template copy — **there is nothing to
rewire**, only to verify (Step 4).

This step runs in four parts, **in this order**: gitignore gate → scaffold the
scripts → the owner fills in the values → seal and verify. The order is not
cosmetic. From Step 5b onward there is plaintext on disk in the app repo, so the
ignore rules must already be in place before anything creates a file.

### Step 5a — Gate: verify the gitignore covers the plaintext

**Do this before creating any file and before any seal attempt.** Two plaintext
artifacts live in the app repo during this workflow:

- `.env.secrets` (or `scripts/.env.secrets`) — the real dev values.
- `secrets.yaml` (or `scripts/secrets.yaml`) — the **unencrypted** intermediate
  that `create-credential-secrets.sh` writes between its two commands. If the
  `kubeseal` line fails, the file left sitting there is plaintext.

Verify rather than assume — the repo may already cover these with a broader rule,
and blindly appending duplicates is noise. `check-ignore -v` prints the rule that
matches each path and stays silent for paths nothing covers:

```bash
git -C <app-repo> check-ignore -v \
  .env.secrets secrets.yaml scripts/.env.secrets scripts/secrets.yaml
```

For every path that comes back **uncovered**, append it to the app repo's
`.gitignore`:

```
.env.secrets
secrets.yaml
scripts/.env.secrets
scripts/secrets.yaml
```

Then **re-run `check-ignore` and confirm all four are now covered.** If any path
is still unignored, **stop**: don't create `scripts/.env.secrets`, don't seal, and
tell the user why. The app template's `.gitignore` ships as unrelated boilerplate
and covers none of these, so on a fresh repo assume you are adding all four.

**Also check nothing is already committed** — `.gitignore` does not protect a
file that is already tracked:

```bash
git -C <app-repo> ls-files -- .env.secrets secrets.yaml scripts/
```

If a plaintext secrets file turns up in the index, **say so loudly and stop**:
those values are in the repo's history, must be treated as leaked, and need
rotating. Do not quietly `git rm --cached` it — that hides the problem without
fixing it, and the decision (rotate now? rewrite history?) is the owner's.

Do **not** add any of these to kubernetes-apps' `.gitignore`. The sealed
`env.yaml` there is encrypted and **must** be committed — ArgoCD can't apply what
isn't in the repo.

### Step 5b — Scaffold the sealing scripts (app repo)

Once the gate passes, create two files in the **app repo** so the sealing
workflow is reproducible by the owner without you:

`scripts/.env.secrets` — created **empty**, with a comment header only:

```
# Dev environment secrets for <app>, one KEY=value per line.
# No `export`, no surrounding quotes, no blank-line-separated sections.
# Gitignored — never commit this file.
```

You write this placeholder and then **never read it again**. Never populate it,
never copy values from a `.env.example`, never invent values.

`scripts/create-credential-secrets.sh` (make it executable, `chmod +x`):

```bash
#!/bin/bash
kubectl create secret generic env --dry-run=client -o yaml --from-env-file=.env.secrets > secrets.yaml

kubeseal --cert https://sealed-secrets.k8s-dev.hpc.uidaho.edu/v1/cert.pem -f secrets.yaml -w secrets.yaml --scope=cluster-wide
```

Document alongside it, in the handoff:

- It is run **from inside `scripts/`** (`cd scripts && ./create-credential-secrets.sh`)
  — both paths in it are relative to the script's own directory.
- It seals `secrets.yaml` **in place**: the file is plaintext after the first
  command and a SealedSecret after the second. Between them, plaintext is on
  disk — which is why Step 5a runs first.
- The sealed result must then be copied to
  `apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml` in kubernetes-apps.
- Neither command needs cluster credentials: `--dry-run=client` never contacts
  the cluster, and `--cert` fetches the public sealing cert over HTTPS.
- **Check `kind:` on `scripts/secrets.yaml` before committing anything**, and
  delete the file when done.

### Step 5c — The owner fills in the values

**Stop here and tell the user**: they must put the dev values into
`scripts/.env.secrets` — one `KEY=value` per line, no `export`, no surrounding
quotes — **before** anything is sealed. Never invent values, and never
reconstruct them from a `.env.example`.

Say what happens if they don't: sealing an empty file produces a perfectly valid
`SealedSecret` with **zero keys**. Nothing errors, ArgoCD syncs green, and the
pods boot blind against missing env vars — a failure that looks like an
application bug and costs an hour to trace back here.

### Step 5d — Seal and verify

**Never read, `cat`, `head`, `grep`, echo, or log `.env.secrets`, and never paste
any value from it into chat, a file, or a commit message.** You do not need to
see it to seal it.

**Where the file lives:** look in **`scripts/.env.secrets` first, then the repo
root**. New deploys get `scripts/`; plenty of already-deployed apps keep it at
the root and moving it is not this skill's job.

**Which command you run.** `create-credential-secrets.sh` is the owner-facing
shared artifact; *you* seal with the pipe below, which streams plaintext straight
into `kubeseal` and never puts it on disk at all:

```bash
cd <app-repo>/scripts && kubectl create secret generic env \
    --dry-run=client -o yaml --from-env-file=.env.secrets \
  | kubeseal \
      --cert https://sealed-secrets.k8s-dev.hpc.uidaho.edu/v1/cert.pem \
      -o yaml --scope=cluster-wide \
  > <kubernetes-apps>/apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml
```

(Drop the `scripts/` if the file is at the repo root.)

**Verify before trusting the output.** A failed cert fetch or a broken pipe can
leave a file that is empty, truncated, or the wrong kind — and the shell
redirect creates the file either way, so its existence proves nothing. Check:

- `kind:` is `SealedSecret`. **If it says `Secret`, plaintext just escaped into
  the manifest repo — delete the file immediately, tell the user, and stop.**
- `spec.encryptedData` is non-empty, with one key per line of `.env.secrets`
  (compare the key *count*; you may name keys, never values).
- `metadata.name` is `env`, matching `secretRef.name` in `deployment.yaml`.
- The `sealedsecrets.bitnami.com/cluster-wide: "true"` annotation is present on
  both `metadata` and `spec.template.metadata`.

If `kubeseal` isn't installed or the cert host is unreachable (it's behind the
campus network), don't improvise a fallback — leave `env.yaml` empty and hand the
sealing step back to the owner, who can run `scripts/create-credential-secrets.sh`
themselves once they're on the network.

Live is **not** sealed here: `deploy/overlays/live/secrets/env.yaml` stays empty
and is sealed at promotion against the live cert.

## Step 6 — Validate

Generating manifests is not deploying, and a green ArgoCD sync is not a working
app. **Do not report success until both halves of this step pass.** They happen
at different times: 6a runs now, before handoff; 6b can only run after the
kubernetes-apps PR is merged and ArgoCD has synced.

### Step 6a — The local container stack (always, before handoff)

If the stack doesn't run on the user's machine it will not run on the cluster,
and diagnosing it there costs an order of magnitude more.

Use the runtime resolved in **Step 0b** — `<RUNTIME>` below is `docker` or
`podman`, and the two are drop-in for everything here. If neither is installed,
skip this step, say so, and carry the gap into the handoff.

**If Step 2b already ran and passed** and nothing has changed since, don't
re-narrate the build — the layer cache will make it fast and there's nothing new
to report about it. Every *functional* check below still runs: Step 2b proved the
stack builds and answers on a port, not that it works. **This step is the gate
before the PR, and a failure here blocks Step 7 whatever Step 2b said.**

**Build every service the manifests reference.** A service that only ever builds
in CI is a service you have not validated.

```bash
# compose-based (docker-compose.yml / compose.yaml / compose.yml)
<RUNTIME> compose build && <RUNTIME> compose up -d
# otherwise, per service
<RUNTIME> build -f Dockerfile[.<service>] -t <app>[-<service>]:local .
```

If `podman compose` reports no provider, fall back to `podman-compose`; if
neither is available, build and run each service individually rather than
declaring failure.

**Then prove it functions, not just that it built:**

- Containers reach `running` and **stay** there — `<RUNTIME> compose ps` twice,
  ~15s apart. A container that is `running` on first look and `Restarting` on
  second is a crash loop, and it will be `CrashLoopBackOff` on the cluster.
- Each service answers on its mapped port: `curl -fsS localhost:<port>/<health-or-root>`.
  Use the same path the readiness probe in `deployment.yaml` uses — if they
  disagree, the probe is wrong and you've just found it before the cluster did.
- Backing services actually connect: if the app has Postgres/Redis/Mongo wired
  via `*_ENABLED`, exercise a path that touches them, not just the root route.
- The repo's own test suite passes, if it has one (`pytest`, `npm test`, …).

**On failure**, read the container logs (`<RUNTIME> compose logs <svc>
--tail=100`), fix the app-side problem, and re-run — do not proceed to generate
or hand off a deploy for a stack that doesn't start. If the failure is
environmental rather than real (a missing local `.env`, a port already bound, no
Docker daemon, a Podman machine that isn't started), say which it is instead of
declaring a pass.

Tear down when done (`<RUNTIME> compose down`) and report exactly what was built,
what was exercised, what passed, and **which runtime you used** — a pass on
Podman and a pass on Docker are not quite the same evidence, and the reader
should know which one they have.

### Step 6b — The dev cluster (after the Step 7 PR merges and ArgoCD syncs)

This is the one step that outlives the handoff: the namespace doesn't exist until
a human merges the PR you open in Step 7. Run it when they tell you they've
merged, or offer to on a later invocation — don't sit and poll for it.

Needs the context confirmed in **Step 0**. If access was declined or unavailable,
**skip this and say so explicitly** in the handoff, listing the checks that went
unverified — silence here reads as "verified".

Everything below is **read-only**. No `apply`, `delete`, `patch`, `exec`, or
`rollout restart`: when you find a problem you fix the *manifest* and let ArgoCD
converge. Patching the cluster produces drift ArgoCD will silently revert, and
you will have "fixed" it twice.

**1. Is it synced?**

```bash
<KUBECTL> get application -n argocd | grep <app>
```

Want `Synced` / `Healthy`. `OutOfSync` usually means the PR isn't merged or
ArgoCD hasn't polled yet — **wait, don't intervene**. `Unknown`/`Degraded` on the
Application itself points at a manifest or AppProject problem, not the workload.

**2. Pod status — hunt for the hung and unhealthy.**

```bash
<KUBECTL> get pods -n apps-rcds-<app>
```

Every pod `Running` **and** `Ready` (`1/1`, not `0/1`) with restarts at 0. Read
these states as diagnoses, not noise:

| State | What it almost always means |
|-------|-----------------------------|
| `ImagePullBackOff` / `ErrImagePull` | image string ≠ what CI builds, or CI never pushed (check 1 & 2 of the invariants) |
| `CreateContainerConfigError` | the `env` Secret doesn't exist — the SealedSecret never decrypted; check it was sealed `--scope=cluster-wide` |
| `Pending` | PVC unbound, or no node can schedule it (resources / node selector) |
| `CrashLoopBackOff` | app-level failure — go straight to logs, with `--previous` |
| `Running` but `0/1` | readiness probe failing: wrong port, wrong path, or the app is slower to start than the probe allows |
| `Init:*` stuck | an init container or a backing service it waits on isn't up |
| `Terminating` for minutes | stuck finalizer or a pod that ignores SIGTERM — note it, don't force-delete |

**3. Troubleshoot what you found.**

```bash
<KUBECTL> describe pod <pod> -n apps-rcds-<app>
<KUBECTL> get events -n apps-rcds-<app> --sort-by=.lastTimestamp | tail -30
```

`describe`'s Events section names the actual cause (`Failed to pull image …`,
`MountVolume.SetUp failed`, `secret "env" not found`). Map each finding back to
the invariant it violates, fix the manifest in kubernetes-apps, and let ArgoCD
re-sync.

**4. Fetch logs where necessary.**

```bash
<KUBECTL> logs <pod> -n apps-rcds-<app> --tail=200
<KUBECTL> logs <pod> -n apps-rcds-<app> --previous --tail=200   # crash loops
```

On a crash-looping pod, `--previous` is the one that matters — the current
container's logs are from the *next* doomed attempt, often empty. For multi-
container pods add `-c <container>`. **Never paste anything credential-shaped out
of logs** into chat, a file, or a commit; if a log line contains a secret,
summarize it ("the DB password is being read as empty") and say where to look.

**5. Test the dev site functionally.**

```bash
curl -fsSI https://<app>.k8s-dev.hpc.uidaho.edu
```

Confirm TLS is valid, the status is what the app should return, and you are not
looking at the ingress default backend (a 404 with no app headers means Ingress
routing or the Service name is wrong while the pods are perfectly healthy —
invariant 3). Then run real functionality:

- If the repo's test suite can target a base URL, run it against dev
  (`BASE_URL=https://<app>.k8s-dev.hpc.uidaho.edu <test command>`) and report
  pass/fail counts.
- Otherwise exercise the documented endpoints or main user path by hand, and
  **say that the coverage was manual** rather than implying a suite ran.
- Exercise at least one path that touches each backing service and any persistent
  storage — those are exactly the pieces that work locally and fail on the
  cluster.

**6. Report a pass/fail line per check** (sync, pods, logs clean, dev site,
functional tests). If anything failed, the deploy is not done: say what broke,
what you changed, and what still needs a re-sync. Green sync is not evidence the
app works — never let it stand in for this step.

## Step 7 — Commit, push, and open the PR

Run this once Step 6a passes. You commit and PR the **kubernetes-apps** work
yourself, on the Step 3b branch — that is the one place this skill writes to
GitHub on its own. You never merge it.

### Step 7a — The pre-commit gate

You are about to commit automatically, so the "check `kind:` before a file
moves" rule stops being advice and becomes a **stop condition**. A push is the
point of no return: after it, anything wrong is in the remote's history.

```bash
git -C <kubernetes-apps> status --short
git -C <kubernetes-apps> diff --cached --name-only
head -3 <kubernetes-apps>/apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml
```

Three things must hold before you commit:

1. **`secrets/env.yaml` says `kind: SealedSecret`.** If it says `Secret`,
   **do not commit** — delete the file, tell the user plaintext nearly escaped,
   and stop. This is the single most important check in the skill.
2. **Every staged path is under `apps/rcds/apps/<app>/`** (plus a
   `deploy-repo-secrets-<project>-live/` dir if promoting). Anything else —
   a sibling app, the template, a stray editor file — **unstage it and say why**.
   An automated commit is exactly where an accidental edit slips through.
3. **No `.env.secrets` or plaintext `secrets.yaml` is staged.** Those live in the
   app repo and have no business in kubernetes-apps at all; if one appears here,
   something went badly wrong upstream — stop and investigate.
4. **Local validation didn't fail.** A *skipped* build — declined, or no runtime
   — is fine: proceed and flag it in the PR body. A build that **ran and failed**
   is not: fix the app first. Don't open a PR proposing to deploy something you
   watched fail to start.

Stage explicitly by path — `git add apps/rcds/apps/<app>/` — **never `git add -A`
or `git add .`**, which is how unrelated working-tree state ends up in a PR.

### Step 7b — Commit and push

One commit, or a few logical ones. The message names the app and what was
generated, and carries whatever attribution footer this session is configured to
use. **Never put a secret value in a commit message**, including "helpful"
context like which key changed to what.

```bash
git -C <kubernetes-apps> commit -m "Add deploy manifests for <app>"
git -C <kubernetes-apps> push -u origin deploy/agent-deploy-<login>-<MM>-<DD>-<YYYY>
```

### Step 7c — Open the labeled PR

```bash
gh pr create --repo ui-iids/kubernetes-apps \
  --base main --head deploy/agent-deploy-<login>-<MM>-<DD>-<YYYY> \
  --title "Deploy <app> to k8s-dev" \
  --label coding-agent --label deploy-application \
  --body-file <body-file>
```

Base is **`main`** — ArgoCD syncs from it on merge.

**PR body** — what a reviewer needs, and nothing they must not see:

- what the app is, and whether this was Mode A (existing repo) or Mode B (new),
- the files added and the key substitutions: image string(s), ports, dev host,
  backing services, storage,
- the seal result: sealed, verified, and the **key count** — **never a value,
  and never a key-to-value mapping**,
- local validation: what built, what was exercised, what passed — and if the
  stack was **not** verified locally (Step 2b declined, or no container runtime),
  say so plainly, so the reviewer knows the ports and probe paths are inferred
  rather than observed,
- preview scope as an explicit arrow (`feature/* → test`), if previews were
  included,
- outstanding blockers: CI hasn't pushed an image yet, `scripts/.env.secrets`
  not yet filled in, `preview` label not created, live secrets unsealed,
- the attribution footer this session is configured to use.

**If labeling fails** — a label doesn't exist, or the user lacks permission — the
PR is still open and that is fine. Report it and hand over the fix:
`gh pr edit <N> --add-label coding-agent --add-label deploy-application`.
**Do not** close and recreate the PR to retry labels.

**Never merge it.** No `gh pr merge`, no `--admin`, no auto-merge, no push to
`main`. The merge is the human review gate this entire flow exists to preserve —
if the user asks you to merge, that's their call to make explicitly, not
something you infer from "it looks fine".

## Step 8 — Hand off

- **Lead with the action they owe you**, as its own line — not a footnote:

  > **Action required: review and merge the PR in `ui-iids/kubernetes-apps`** —
  > `<PR URL>`. Nothing is deployed until you merge it; ArgoCD syncs on merge.

  Then say what is blocked behind that merge: the Step 6b dev-cluster
  validation, preview environments becoming active at all, and the app-repo
  commit/PR they still owe (which you did **not** open for them).
- Summarize the files created and the key substitutions.
- Report the seal result: that dev `env.yaml` is sealed and verified, and how
  many keys it carries — **never which values**.
- Report the validation results: what the local stack did — including "not
  verified locally, because …" when Step 2b and Step 6a were both skipped — and
  either the dev-cluster findings or an explicit "not verified on the cluster,
  because …".
- Flag for later: `deploy/overlays/live/secrets/env.yaml` is still empty and
  needs its own seal against the live cert when the app is promoted.
- **If previews were included**, spell out how one is actually triggered — the
  manifests alone produce nothing:
  1. Merging to `kubernetes-apps` **activates** the ApplicationSet. No manual
     apply: the `apps-rcds` ApplicationSet runs a git directory generator over
     `apps/rcds/apps/*` sourcing each app's `overlays/dev`, so a new
     `pull-request-builds.yaml` there is picked up on its own.
  2. The PR must be **labeled `preview`** — `gh pr edit <N> --add-label preview`.
     Confirm the label was created in Step 1c; if it wasn't (no `gh` auth, no
     write access), list that as an **open blocker**, not a footnote — a missing
     label is indistinguishable from "previews are broken", since the generator
     matches nothing with no error anywhere.
  3. State the scope you configured as an arrow (`test → main`), the URL they'll
     get (`https://<app>-pr-<N>.k8s-dev.hpc.uidaho.edu`), and that it's pruned on
     PR close. Include the walkthrough from Step 1c if they asked for one.
- **If secrets are still unsealed** because the owner hasn't filled in
  `scripts/.env.secrets` (Step 5c), say so as a blocker and point them at
  `scripts/create-credential-secrets.sh` plus where to copy the sealed output.
- **The two repos are handled differently, and say so plainly.** kubernetes-apps
  is committed, pushed, and PR'd by you (Step 7). The **app repo** — CI
  workflows + Dockerfiles, the `scripts/` sealing files, and the `.gitignore`
  from Step 5a — is left uncommitted in their working tree and needs its own
  commit and PR **from them**. Don't commit or push in the app repo, and don't
  let "the PR is open" imply both repos are handled: they're separate repos and
  the app-repo half is still theirs to land.
- If the app repo's `build-*-container.yaml` hasn't run yet (no image in GHCR),
  call that out as a blocker — ArgoCD will fail to pull until CI builds it.

## Guardrails

- Two write locations only: (1) in kubernetes-apps, `apps/rcds/apps/<app>/` (plus,
  for a new live project, a `deploy-repo-secrets-<project>-live/` dir at the repo
  root if needed); (2) in the app repo, its `.github/workflows/`,
  `Dockerfile[.<service>]`, `scripts/` (the Step 5b sealing files), and
  `.gitignore` (Step 5a). Don't edit other apps or either template.
- **Three sanctioned GitHub-side writes, and no others:** the `preview` label in
  Step 1c (only after the user asks for previews and okays it), the
  kubernetes-apps branch in Step 3b, and the labeled PR in Step 7. Still
  forbidden: **merging** any PR, pushing to `main`, changing repo settings,
  branching or committing in the **app repo**, and any write at all to a repo
  outside `ui-iids`/`ui-insight`.
- Plaintext secrets are write-only to you: stream `.env.secrets` into `kubeseal`,
  never read it back, never echo a value, never let a `kind: Secret` file reach
  the manifest repo. If you're ever unsure whether a file is sealed, check
  `kind:` before it moves — not after.
- If the user asks to deploy something outside `ui-iids`/`ui-insight`, stop and
  confirm — that's outside our deploy pattern.
- When unsure about port, storage, domain, or backing services, ask rather than
  guess; these are baked into many files and tedious to fix after the fact.
- **The repo is the source of truth; the cluster is a last resort.** Nearly every
  convention question is answerable by grepping sibling apps, and that's the
  cheapest and most reliable check available:

  ```bash
  # what do apps that already have previews do?
  grep -rl "kind: ApplicationSet" apps/rcds/apps/
  # settle a convention by counting real usage rather than trusting the template
  cat apps/rcds/apps/<some-pr-enabled-app>/base/project.yaml
  ```

  The template is a *teaching* artifact and lags the fleet — where template and
  sibling apps disagree, the siblings win. Read at least one working PR-enabled
  app before writing a `pull-request-builds.yaml`.

- **Cluster access: read-only, through the Step 0 `<KUBECTL>`, never for Secret
  contents.** Cluster reads are a normal part of this skill — Step 6b's validation
  is built on them, as are one-off facts only the cluster has (the running ArgoCD
  version: does it support `filters`?, whether `creds-github-<org>` exists, a
  schema check via `kubectl apply --dry-run=server`, which validates against the
  real API server and persists nothing). The rules:
  - **Use exactly the access point resolved in Step 0** — `kubectl --context
    <ctx>` for a local kubeconfig, `ssh <sa>@<host> kubectl` for a service
    account. Never fall back to the ambient current context, and never reach for
    a kubeconfig the user didn't approve.
  - **No cross-mode fallback, ever.** If the chosen access point fails, stop and
    report — don't try the other one. They are different identities with
    different RBAC, so an answer from the wrong one isn't a workaround, it's a
    wrong answer delivered confidently.
  - **On a master node, read-only `kubectl` and nothing else** — no file edits,
    no `scp`, no installs, no exploring. SSH access is more power, not more
    permission, and the node is shared infrastructure.
  - If Step 0 found nothing or the user declined, **don't retry later** — proceed
    repo-only and report the checks you couldn't run.
  - Read-only means read-only: no `apply` (except `--dry-run=server`), `delete`,
    `patch`, `scale`, or `rollout restart`. Fix manifests, not clusters — ArgoCD
    reverts drift, so a live patch "works" exactly until the next sync.
  - Prefer existence checks that cannot leak: `kubectl get secret <name>
    --ignore-not-found`. **Never** read a Secret's `data`/`stringData` — and don't
    reach for `-o jsonpath` on a Secret even for a benign field; on Secrets the
    habit is the risk.
  - Never `exec` into `argocd-repo-server` (or any pod) to try things out. If you
    need a tool it has — e.g. the exact `kustomize` build it uses — fetch that
    version into a scratch dir locally and test there.
