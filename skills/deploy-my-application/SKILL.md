---
name: deploy-my-application
description: Deploy a University of Idaho RCDS/IIDS application to our Kubernetes cluster by generating ArgoCD + Kustomize deploy manifests in the `ui-iids/kubernetes-apps` GitOps repo, modeled on the canonical `apps/rcds/apps/deploy-template/` example. Handles two cases — (1) deploy an EXISTING repo from the `ui-iids` or `ui-insight` GitHub orgs, or (2) scaffold and deploy a NEW project (pick a namespace, generate the full manifest tree). Use when the user says "deploy my application", "deploy this repo", "deploy my app to the cluster", "set up deploy manifests", "create a namespace and deploy", "add my project to kubernetes-apps", or invokes /deploy-my-application.
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
- **Never** apply to a cluster, never hand-run `kubectl`. Produce files, then let
  the user commit/PR them for ArgoCD.
- **Never** put plaintext secrets in manifests. Env secrets are Bitnami
  `SealedSecret`s (ship them empty; the user seals real values with `kubeseal`).

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
| PR previews? | include the ApplicationSet or not |

## Step 3 — Decide what to include

Use the template as the maximal set and trim per the README's "Choosing what to
include":

- **Stateless, no deps** → drop `storage-volume.yaml`, `services/`, and the
  volume wiring in `deployment.yaml`; remove those from `deploy/base/kustomization.yaml`.
- **App-owned files** → keep `storage-volume.yaml` + the volume mount.
- **Needs a DB/cache** → keep/rename `services/<svc>.yaml` and set the matching
  `*_ENABLED` env to `"True"`.
- **No PR previews** → omit `overlays/dev/pull-request-builds.yaml`.
- **Dev only** → omit the whole `deploy/overlays/live/` tree (add later on
  promotion); for prod, include it with the real host, TLS, and a pinned image
  tag.

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
- Overlay relative paths to the shared `deploy-repo-secrets-*` dir have the right
  depth (`../../../../../../../` from `deploy/overlays/<env>`).
- Hosts: dev = `<app>.k8s-dev.hpc.uidaho.edu`; PR = `<app>-pr-{{.number}}...`.

## Step 5 — Hand off

- Summarize the files created and the key substitutions.
- Tell the user the two remaining manual steps you cannot/should not do:
  1. **Seal secrets** — populate `secrets/env.yaml` via `kubeseal` (it ships empty).
  2. **Commit + open a PR** to `ui-iids/kubernetes-apps`; ArgoCD deploys on merge.
- Do not commit or push unless the user explicitly asks. Note that the app repo
  CI (workflows + Dockerfiles) and the kubernetes-apps manifests are **separate
  repos** — each needs its own commit/PR.
- If the app repo's `build-*-container.yaml` hasn't run yet (no image in GHCR),
  call that out as a blocker — ArgoCD will fail to pull until CI builds it.

## Guardrails

- Two write locations only: (1) in kubernetes-apps, `apps/rcds/apps/<app>/` (plus,
  for a new live project, a `deploy-repo-secrets-<project>-live/` dir at the repo
  root if needed); (2) in the app repo, its `.github/workflows/` and
  `Dockerfile[.<service>]` when scaffolding/fixing CI. Don't edit other apps or
  either template.
- If the user asks to deploy something outside `ui-iids`/`ui-insight`, stop and
  confirm — that's outside our deploy pattern.
- When unsure about port, storage, domain, or backing services, ask rather than
  guess; these are baked into many files and tedious to fix after the fact.
