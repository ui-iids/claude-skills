---
name: deploy-my-application
description: Deploy a University of Idaho RCDS/IIDS application to our Kubernetes cluster by generating ArgoCD + Kustomize deploy manifests in the `ui-iids/kubernetes-apps` GitOps repo, modeled on the canonical `apps/rcds/apps/deploy-template/` example. Handles two cases — (1) deploy an EXISTING repo from the `ui-iids` or `ui-insight` GitHub orgs, or (2) scaffold and deploy a NEW project (pick a namespace, generate the full manifest tree). Also seals the app's dev env secrets into a Bitnami SealedSecret with `kubeseal`, piping `.env.secrets` through without ever reading it, and gitignores the plaintext in the app repo. Use when the user says "deploy my application", "deploy this repo", "deploy my app to the cluster", "set up deploy manifests", "create a namespace and deploy", "add my project to kubernetes-apps", "seal my secrets", "kubeseal my env file", or invokes /deploy-my-application.
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
  `--dry-run=client` seal touches no cluster at all. For the narrow read-only
  cases that are genuinely worth a cluster round-trip, see the cluster-access
  guardrail — and **announce before the first one**.
- **Never** put plaintext secrets in manifests, chat, or commit messages. Env
  secrets are Bitnami `SealedSecret`s: you seal the **dev** values yourself
  (Step 5) by piping `.env.secrets` through `kubeseal` without ever reading it.
  Sealed output is encrypted and safe to commit. **Live** secrets stay empty
  until promotion.

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

If they say yes, **do not wait** on the manual SonarQube/GitHub setup — it
happens in two web UIs and would block the deploy for no reason. Instead:

1. **Scaffold immediately.** Copy `.github/workflows/sonar-main.yaml` +
   `sonar-project.properties` from `ui-iids/deploy-template` into the app repo,
   leaving `sonar.projectKey` as a clearly-marked placeholder
   (`sonar.projectKey=TODO-paste-key-from-sonarqube`). Adjust
   `sonar.sources` / `sonar.tests` / language + coverage settings to the repo's
   layout, and the build/test steps in `sonar-main.yaml` to its stack.
2. **Keep going** with the rest of the deploy (manifests, etc.). The Sonar scan
   simply won't pass until the owner finishes the steps below — that's harmless.
3. **Hand the owner this checklist** at the end (only they can do these — you
   can't create the project or set GitHub secrets for them):
   - Create the project on our instance
     <https://sonarqube.k8s-dev.hpc.uidaho.edu/projects>, per
     <https://knowledgebase.k8s-dev.hpc.uidaho.edu/index.php/Adding_Sonarqube_to_a_repository>.
   - In the app repo's GitHub settings, add a repo **secret** `SONAR_TOKEN` (the
     analysis token) and a repo **variable** `SONAR_HOST_URL` =
     `https://sonarqube.k8s-dev.hpc.uidaho.edu`.
   - Paste the generated project key (UUID-suffixed, e.g.
     `ui-iids_my-app_c4489195-...`) into `sonar.projectKey` — or hand it to you
     and you'll set it.

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

**Ask the owner:**

1. *"Do you want PR preview builds — a temporary deploy spun up per open pull
   request labeled `preview`?"*
2. Only if yes: *"Which PRs should get one?"* **Do not offer `main` as a
   presumed default.** Check the app repo's actual branching model first
   (`gh repo view <org>/<app> --json defaultBranchRef -q .defaultBranchRef.name`,
   and skim recent PR targets) — plenty of our repos merge features into a
   long-lived `test` branch and use `test → main` only as a promotion PR. On such
   a repo, "previews for PRs into `main`" yields previews for exactly the
   promotion PR, which may or may not be what they meant. Restate their answer as
   an explicit arrow (`feature/* → test`, or `test → main`) and get confirmation
   before generating — "PRs into X" and "PRs from X" are opposite ends of the
   arrow and users mix them up.

**If they decline**, omit `overlays/dev/pull-request-builds.yaml` and its entry in
`overlays/dev/kustomization.yaml`. Nothing else changes. Mention in the Step 6
handoff that previews were skipped and can be added later.

**If they accept**, generate `pull-request-builds.yaml` from the template with the
generator pointed at `<org>/<app>`, the `preview` label gate kept, and the PR host
pattern `<app>-pr-{{.number}}.k8s-dev.hpc.uidaho.edu`.

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
| container port | each service's listen port (e.g. 8000) |
| dev host | `<app>.k8s-dev.hpc.uidaho.edu` |
| live host + TLS | only if promoting to prod (real domain) |
| backing services | none / mongodb / redis / postgres / mariadb |
| persistent storage | does the app need its own PVC? |
| PR previews? | **ask** (Step 1c) — include the ApplicationSet or not |
| PR scope | PR previews only — the confirmed source→target arrow (Step 1c). Label-gated by default; add `filters:` only if they asked for branch scoping. Never assume `main`. |

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

**Prerequisite:** the app repo has a `.env.secrets` holding the dev values, one
`KEY=value` per line, no `export`, no surrounding quotes. If it's missing, ask
the owner for it — never invent values, and never reconstruct it from a
`.env.example`.

**Never read, `cat`, `head`, `grep`, echo, or log `.env.secrets`, and never paste
any value from it into chat, a file, or a commit message.** You do not need to
see it to seal it. The command below is the only correct way to handle it:
plaintext is streamed straight into `kubeseal` and never touches disk.

Run from the app repo root (where `.env.secrets` lives):

```bash
kubectl create secret generic env --dry-run=client -o yaml \
    --from-env-file=.env.secrets \
  | kubeseal \
      --cert https://sealed-secrets.k8s-dev.hpc.uidaho.edu/v1/cert.pem \
      -o yaml --scope=cluster-wide \
  > <kubernetes-apps>/apps/rcds/apps/<app>/deploy/overlays/dev/secrets/env.yaml
```

`--dry-run=client` means no cluster contact; `--cert` fetches the public sealing
cert over HTTPS. Neither needs cluster credentials.

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
sealing step back to the owner with the command above.

Live is **not** sealed here: `deploy/overlays/live/secrets/env.yaml` stays empty
and is sealed at promotion against the live cert.

### Step 5b — Gitignore the plaintext (app repo only)

In the **app repo** — not kubernetes-apps — ensure `.gitignore` contains:

```
.env.secrets
secrets.yaml
```

`.env.secrets` is the real plaintext. `secrets.yaml` is the intermediate the
older two-step seal wrote to disk; the piped command never creates it, but ignore
it anyway so a hand-run of the old flow can't commit plaintext. Note the app
template's `.gitignore` ships as unrelated boilerplate and ignores neither, so
assume you must add them.

Do **not** add these to kubernetes-apps' `.gitignore`. The sealed `env.yaml`
there is encrypted and **must** be committed — ArgoCD can't apply what isn't in
the repo.

## Step 6 — Hand off

- Summarize the files created and the key substitutions.
- Report the seal result: that dev `env.yaml` is sealed and verified, and how
  many keys it carries — **never which values**.
- Tell the user the remaining manual step you should not do:
  1. **Commit + open a PR** to `ui-iids/kubernetes-apps`; ArgoCD deploys on
     merge. The sealed `env.yaml` is encrypted and belongs in that commit.
- Flag for later: `deploy/overlays/live/secrets/env.yaml` is still empty and
  needs its own seal against the live cert when the app is promoted.
- **If previews were included**, spell out how one is actually triggered — the
  manifests alone produce nothing:
  1. Merging to `kubernetes-apps` **activates** the ApplicationSet. No manual
     apply: the `apps-rcds` ApplicationSet runs a git directory generator over
     `apps/rcds/apps/*` sourcing each app's `overlays/dev`, so a new
     `pull-request-builds.yaml` there is picked up on its own.
  2. The PR must be **labeled `preview`** — `gh pr edit <N> --add-label preview`.
     The label frequently doesn't exist in a fresh repo yet, and a missing label
     is indistinguishable from "previews are broken": the generator just matches
     nothing, with no error anywhere. Tell them to create it if needed
     (`gh label create preview`).
  3. State the scope you configured as an arrow (`test → main`), the URL they'll
     get (`https://<app>-pr-<N>.k8s-dev.hpc.uidaho.edu`), and that it's pruned on
     PR close.
- Do not commit or push unless the user explicitly asks. Note that the app repo
  CI (workflows + Dockerfiles, plus the `.gitignore` from Step 5b) and the
  kubernetes-apps manifests are **separate repos** — each needs its own commit/PR.
- If the app repo's `build-*-container.yaml` hasn't run yet (no image in GHCR),
  call that out as a blocker — ArgoCD will fail to pull until CI builds it.

## Guardrails

- Two write locations only: (1) in kubernetes-apps, `apps/rcds/apps/<app>/` (plus,
  for a new live project, a `deploy-repo-secrets-<project>-live/` dir at the repo
  root if needed); (2) in the app repo, its `.github/workflows/`,
  `Dockerfile[.<service>]`, and `.gitignore` when scaffolding/fixing CI or
  Step 5b. Don't edit other apps or either template.
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

- **Cluster access: read-only, announced, never for Secret contents.** Occasionally
  a fact only the cluster has is worth it — the running ArgoCD version (does it
  support `filters`?), whether `creds-github-<org>` exists, or a schema check via
  `kubectl apply --dry-run=server` (validates against the real API server and
  persists nothing). These are legitimate, but:
  - **Say you're about to touch the cluster before you do it.** The user is
    thinking about files in a repo; unannounced `kubectl` against their live
    context is a surprise, and `--dry-run=server` still contacts the cluster.
  - Prefer existence checks that cannot leak: `kubectl get secret <name>
    --ignore-not-found`. **Never** read a Secret's `data`/`stringData` — and don't
    reach for `-o jsonpath` on a Secret even for a benign field; on Secrets the
    habit is the risk.
  - Never `exec` into `argocd-repo-server` (or any pod) to try things out. If you
    need a tool it has — e.g. the exact `kustomize` build it uses — fetch that
    version into a scratch dir locally and test there.
