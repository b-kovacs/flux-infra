# flux-infra

A Kubernetes platform managed with GitOps. GitOps means the Git repository is the single
source of truth: instead of running `kubectl apply` by hand, a controller called Flux
watches this repo and keeps the cluster matching it automatically.

It runs on a local multi-node `kind` cluster (`kind` runs real Kubernetes nodes as
containers, so you get a proper multi-node cluster without needing real servers). The
platform is security hardened, observable with Prometheus and Grafana, and has a full
CI/CD pipeline: push code, and it builds, tests, packages, and deploys itself with no
manual step. I proved the whole thing rebuilds from this repo alone by deleting the
cluster completely and recreating it from scratch.

The application it runs lives in a separate repo,
[springboot-demo](https://github.com/b-kovacs/springboot-demo). That's a small internal
announcements service, kept deliberately simple so the learning effort could go into the
platform here instead of into the app's own business logic. Both repos matter equally to
the project. This platform builds the app, deploys it, secures it, and monitors it.

The full build log is in [`learning/`](https://github.com/b-kovacs/springboot-demo/tree/main/learning)
inside the app repo. It documents every bug I found, every wrong theory I had and later
corrected, and why each design decision was made. This README is the short tour. That
folder is the detailed, honest version.

## Architecture

```
Flux (GitOps CD): reconciles everything below from this repo, continuously
│
├── infrastructure/        MetalLB (gives Services real LoadBalancer IPs), Envoy Gateway
│                          (the Kubernetes Gateway API, for routing traffic), and the
│                          local container registry's Service
├── metallb-config/        MetalLB's IP address pool, kept as its own layer, applied only
│                          after infrastructure is ready (explained below)
├── apps/                  demo-app and Postgres: Deployments, Services, routing rules,
│                          NetworkPolicies (firewall rules between pods),
│                          PodDisruptionBudgets (protect pods from careless eviction),
│                          a Postgres backup job, and secrets encrypted with SOPS and age
│                          so nothing sensitive is ever stored in plain text
├── tekton/, tekton-pipeline/   the CI pipeline: clone the app repo, build and test it
│                          with Maven, build a container image with Kaniko, push a unique
│                          tag. A CronJob polls the app repo for new commits, because a
│                          real GitHub webhook can't reach this machine (no inbound path
│                          through WSL and NAT)
├── image-automation/      Flux watches the registry, notices the new image tag, and
│                          commits the deployment update back into this same repo,
│                          closing the loop from code change to running pod
└── observability/         Prometheus, Grafana, and Alertmanager, scraping real metrics
    observability-config/  from demo-app, not just infrastructure metrics
```

Some of these layers are deliberately split in two, like `metallb-config` next to
`infrastructure`. The reason: a CRD is a custom resource type a controller adds to
Kubernetes. If you ask Flux to install a controller and use its custom resource in the
same step, the check Flux does before applying anything (a dry run) fails, because the
custom resource type doesn't exist yet. Splitting the config into its own layer that
waits for the installer to finish first avoids that. It's a small thing, but getting it
wrong was one of the actual lessons from building this.

## What this shows

- **GitOps that actually works, not just GitOps that's configured.** I deleted the
  cluster entirely and rebuilt it from this repo alone, twice. The second time was after
  fixing a bug the first attempt exposed. "It works right now" and "it rebuilds itself
  from Git" are different claims, and only that kind of destructive test proves the
  second one.
- **Secrets handled properly.** Everything sensitive is encrypted with SOPS and age
  before it goes into Git. There's one exception in the history: an early demo password
  that was committed in plain text before the encryption was set up. I left it visible in
  the history instead of rewriting it away, since it's a throwaway password and hiding it
  would be more dishonest than useful.
- **CI/CD that's actually automatic.** A code change goes from `git push` to a running
  pod through build, test, image push, and redeploy, with nobody touching `kubectl` in
  between.
- **Security hardening that's verified, not assumed.** Default-deny network policies,
  containers running as non-root users, least-privilege permissions, encrypted secrets.
  Before trusting the network policy, I tested that the cluster's networking plugin
  actually enforces it. More on that below.
- **Observability that measures the application, not just the cluster.** Prometheus
  scrapes real metrics from demo-app itself. Getting that working exposed a genuine
  Kubernetes gotcha, which I found and fixed.
- **Debugging habits that scale past reading code.** Several of the bugs below were found
  by reading a controller's own logs or generated config, not by staring at YAML files.
  That's the same instinct needed when a production system misbehaves for a reason the
  configuration doesn't mention.

## A few of the actual bugs (full detail in `learning/`)

**A monitoring rule matched nothing, and nothing complained.** Prometheus never scraped
demo-app. No error anywhere, the target just wasn't in the list. The cause turned out to
be a mismatch between two different fields that look similar: the rule telling Prometheus
what to scrape matches a Service's labels, not the selector that controls which pods it
routes to. Two objects that look correctly connected, but aren't. I found it by reading
Prometheus's own generated configuration for that specific job, not by rereading the
manifests.

**Three theories for one flaky build failure, and only the third one held up.** A build
would occasionally fail because a later step couldn't find a file an earlier step had
just built. My first guess was that the shared storage wasn't reliably available across
different nodes. I never actually checked that, and it turned out to be wrong. My second
attempt was to build a fix for that theory, which failed immediately and revealed that the
tool already has a built-in feature guaranteeing the thing I thought was missing. I
retracted the theory instead of guessing again. The real answer came from comparing the
timestamps of every failed run against every other run running at the same time: two
builds were sharing one fixed storage volume with no protection against both writing to
it at once, so a later build's cleanup step deleted an earlier build's files while it was
still running. Fixed by having the automated trigger check whether a build is already
running before starting a new one.

**The full rebuild test caught a bug in the fix meant to make the rebuild test possible.**
I made a config file declarative through Nix (my system configuration tool) to close a gap
where it wasn't tracked in version control. That change turned the file into a symlink
into the Nix store. That symlink broke inside a freshly created Kubernetes node, because
the node only had access to the specific folder it was told to mount, not the store path
the symlink pointed to. Only actually running the full destroy-and-rebuild test caught
this. Reading the code change would not have.

**A firewall rule was tested before I trusted it.** Kubernetes network policies (rules
that control which pods can talk to which) are only as good as the network plugin
enforcing them, and some plugins have, at points in their history, silently accepted the
rule without enforcing it at all. That's worse than having no rule, because it looks safe
and isn't. Before writing a real policy, I deployed a simple deny-all rule in a throwaway
namespace just to confirm this specific cluster's networking actually blocks traffic when
told to.

## Built with Claude Code

I did the infrastructure work, the debugging, the full-rebuild testing, and this README
together with Claude Code, Anthropic's coding agent, working directly against the live
cluster and both repos with real `kubectl`, `flux`, `git`, and `gh` access. I used it
because it let me iterate on real infrastructure faster, with every fix checked against
the running system instead of assumed to work. The `learning/` folder records that
process honestly, including the times a diagnosis was wrong and got corrected.

## Repo map

```
clusters/kind/
├── flux-system/           Flux's own bootstrap files, generated by Flux, not hand-edited
├── infrastructure.yaml, infrastructure/       MetalLB, Envoy Gateway, registry Service
├── metallb-config.yaml, metallb-config/       MetalLB's IP pool, its own layer (see above)
├── apps.yaml, apps/                           demo-app, Postgres, network policies,
│                                               disruption budgets, backups, secrets
├── tekton.yaml, tekton-pipeline.yaml,
│   tekton-pipeline/                           CI tasks, the pipeline, the polling job
├── image-automation.yaml, image-automation/   Flux's image automation resources
└── observability.yaml, observability/,
    observability-config.yaml,
    observability-config/                      Prometheus stack and demo-app's scrape rule
```
