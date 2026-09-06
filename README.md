# flux-infra

A GitOps-managed Kubernetes platform, built and hardened from scratch on a local `kind`
cluster: multi-node, security-hardened, observable, with fully automated CI/CD from a
`git push` to a running pod — and the whole thing rebuilds itself from this repository
alone, verified by actually destroying and recreating the cluster.

The companion application repo is [springboot-demo](https://github.com/b-kovacs/springboot-demo)
— a small Spring Boot service this platform builds, deploys, secures, and monitors.

**The full build log — every bug found, every wrong theory caught and corrected, every
design decision and why — is in [`learning/`](https://github.com/b-kovacs/springboot-demo/tree/main/learning)
in the app repo.** This README is the tour; that folder is the detailed, honest record.

## Architecture

```
Flux (GitOps CD) — reconciles everything below from this repo, continuously
│
├── infrastructure/        MetalLB (LoadBalancer IPs), Envoy Gateway (Gateway API),
│                          the local container registry's Service/EndpointSlice
├── metallb-config/        IPAddressPool — its own layer, dependent on infrastructure
│                          (avoids a CRD dry-run deadlock — see the lessons below)
├── apps/                  demo-app + Postgres: Deployments, Services, Gateway/HTTPRoute,
│                          NetworkPolicies, PodDisruptionBudgets, a Postgres backup CronJob,
│                          secrets encrypted with SOPS+age (nothing plaintext, ever)
├── tekton/, tekton-pipeline/   In-cluster CI: clone → Maven build+test → Kaniko image
│                          build → push a unique, chronologically-sortable tag.
│                          Triggered by a CronJob polling the app repo (a real GitHub
│                          webhook is impossible here — no inbound path through WSL/NAT)
├── image-automation/      Flux watches the registry, picks the newest tag, auto-commits
│                          the Deployment update back into THIS repo — closing the loop
└── observability/         kube-prometheus-stack (Prometheus, Grafana, Alertmanager),
    observability-config/  scraping demo-app's real JVM/HTTP metrics
```

Every layer above is its own Flux `Kustomization`, several deliberately split from their
own CRD installer (`metallb-config` from MetalLB's HelmRelease, `observability-config` from
the Prometheus Operator's HelmRelease) — bundling a CRD's config with the thing that
installs the CRD causes a dry-run deadlock the first time Flux applies it. Getting this
ordering right, and knowing why it matters, is one of the platform's actual lessons, not
just a structural choice.

## What this demonstrates

- **GitOps done for real, not just configured**: the cluster was fully destroyed
  (`kind delete cluster`) and rebuilt from this repository alone — twice, the second time
  after fixing a bug the first attempt exposed. "It works" and "it rebuilds from Git" are
  different claims; only the acid test proves the second one.
- **Secrets management**: SOPS + age, zero plaintext secrets in git history (bar one
  trivial pre-migration demo password, kept visible rather than scrubbed — see the
  postgres-secret history if curious).
- **CI/CD, fully automated, no manual triggers**: a real code change flows from `git push`
  through build, test, image push, and a Flux-driven redeploy with no human touching
  `kubectl` in between.
- **Security hardening applied and *verified*, not assumed**: default-deny
  `NetworkPolicy` (tested against the actual CNI before being trusted — see below),
  non-root containers, least-privilege RBAC, encrypted secrets.
- **Observability**: real application metrics (not just infrastructure metrics) scraped
  into Prometheus, with the actual Kubernetes-specific gotcha that broke it the first time
  found and fixed.
- **Debugging discipline that scales to systems, not just code**: several of the incidents
  below were found by reading a controller's own generated config/logs, not by re-reading
  YAML — the same instinct that matters when a production system misbehaves for a reason
  the manifest doesn't state.

## A few of the engineering stories (the honest, detailed versions are in `learning/`)

**A ServiceMonitor matched nothing, silently.** Prometheus never scraped `demo-app` — no
error anywhere, the target simply didn't exist in the scrape target list. The cause:
`ServiceMonitor.spec.selector` matches a Service's own `metadata.labels`, not its
`spec.selector` (which only controls pod routing) — a distinction neither object's YAML
states. Found by reading Prometheus's own generated scrape config for the specific job, not
by re-reading the manifests.

**Three theories for one intermittent Tekton failure, each retired only by evidence.** A
build would occasionally fail because a later step couldn't find a file an earlier step had
just built. Theory one — `local-path` storage not reliably shared across nodes — was never
actually checked and turned out wrong. Theory two — building a pod-affinity "fix" for that
failed immediately, revealing Tekton already has a built-in Affinity Assistant that
guarantees co-scheduling by design; that diagnosis was retracted rather than left on record,
honestly leaving the real cause unstated rather than guessing again. The actual cause,
found by finally correlating every failed run's timestamps against every *other* run's: two
`PipelineRun`s sharing one fixed-name workspace PVC with zero mutual exclusion — a later
run's cleanup step was deleting an earlier run's `pom.xml`/`src`/`target` mid-build. Fixed
by having the automated trigger check for an in-flight run before starting a new one.

**The acid test caught a bug in its own prerequisite fix.** Making a config file
declarative via Nix (to close a "not committed to git" gap) turned it into a symlink into
`/nix/store` — which broke inside a fresh Kubernetes node's mount namespace, since `kind`'s
`extraMounts` only bind-mounted the directory, not the store path the symlink pointed into.
Only running the actual destroy-and-rebuild test surfaced this; reading the diff would not
have.

**A NetworkPolicy was tested before being trusted.** `kindnet` has, at points in its
history, silently not enforced `NetworkPolicy` at all — the most dangerous kind of security
control, one that looks configured and isn't. Before writing any real policy, a throwaway
deny-all rule was deployed in a scratch namespace specifically to confirm this cluster's
CNI actually blocks traffic, rather than assuming it from general Kubernetes knowledge.

## Built with Claude Code

The infrastructure debugging, the manifest authoring, the acid-test verification, and this
README were done working with Claude Code (Anthropic's CLI coding agent) as a pair
programmer with direct `kubectl`/`flux`/`git`/`gh` access to the live cluster and both
repos — used deliberately as a way to iterate on real infrastructure faster, with every fix
verified live rather than assumed. The `learning/` folder in the app repo is, among other
things, a record of that collaboration actually working end to end, including the times it
found a wrong diagnosis and self-corrected.

## Repo map

```
clusters/kind/
├── flux-system/           Flux's own bootstrap manifests (generated, not hand-edited)
├── infrastructure.yaml, infrastructure/       MetalLB, Envoy Gateway, registry Service
├── metallb-config.yaml, metallb-config/       IPAddressPool (separate layer - see above)
├── apps.yaml, apps/                           demo-app, Postgres, NetworkPolicies, PDBs,
│                                               backups, secrets (SOPS-encrypted)
├── tekton.yaml, tekton-pipeline.yaml,
│   tekton-pipeline/                           CI: Tasks, Pipeline, the polling CronJob
├── image-automation.yaml, image-automation/   Flux image automation (ImageRepository,
│                                               ImagePolicy, ImageUpdateAutomation)
└── observability.yaml, observability/,
    observability-config.yaml,
    observability-config/                      kube-prometheus-stack + demo-app's
                                                ServiceMonitor (separate layer - see above)
```
