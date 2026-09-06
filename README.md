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
announcements service. Both repos matter equally to the project, and both are kept to the
basics on purpose: enough Kubernetes here to be a real, properly automated platform, and
enough Spring Boot there to be a real, properly layered service, without either side
growing beyond what's needed to learn the fundamentals well. This platform builds the app,
deploys it, secures it, and monitors it.

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

## Why the manifests are heavily commented

The YAML in this repo carries more comments than production Kubernetes manifests usually
would, on purpose, for the same reason the Spring Boot app in the other repo does: this
project doubles as study material for me. Nearly every file explains the Kubernetes or
Flux concept it demonstrates, or the specific bug that led to a particular line existing,
right at the line it applies to, not just at the top of the file. A `dependsOn` gets a
comment saying which CRD it's actually waiting on and why; an `imagePullPolicy` gets a
comment saying which incident made it worth setting explicitly. Comment density on its
own isn't a fair way to judge either repo's day-to-day production style: the more useful
signal is the actual decisions underneath the comments, since those would still be there
with every comment stripped out. A handful of files are deliberately left alone: Flux's
own bootstrap output (`flux-system/gotk-*.yaml`) and the vendored upstream Tekton release
and catalog Task (`infrastructure/tekton/release.yaml`,
`tekton-pipeline/git-clone.yaml`), since those aren't authored here and get overwritten
wholesale on the next upgrade anyway.

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

### A monitoring rule matched nothing, and nothing complained

Prometheus needs to know which pods to scrape for metrics. In this cluster that's done
with a `ServiceMonitor`, a custom resource that a controller called the Prometheus
Operator watches. A `ServiceMonitor` points at a Kubernetes Service and says, in effect,
scrape whatever that Service sends traffic to.

I wrote one for demo-app and pointed it at the demo-app Service, which already routed
real traffic correctly. Nothing errored. Prometheus still never scraped it. I checked
Prometheus's own list of scrape targets directly, and demo-app wasn't there at all. Not
listed as failing, just absent, as if it had never been configured.

The cause: a Service actually has two different things that both sound like "which pods
does this apply to." One is `spec.selector`, which really does decide which pods get
traffic. The other is `metadata.labels`, plain labels on the Service object itself, with
no effect on routing at all. A `ServiceMonitor`'s own selector matches against a Service's
`metadata.labels`, not its `spec.selector`. My Service had `spec.selector` set, so routing
worked fine, but no `metadata.labels` of its own, so the `ServiceMonitor` had nothing to
actually attach to, even though it was pointed at the right Service.

I found this by reading Prometheus's own generated configuration for that specific job,
not by rereading the YAML files, which looked completely reasonable on their own. The
missing piece was a connection between two objects that neither file states directly.
Fixed by adding the missing labels straight onto the Service.

### Three theories for one flaky build failure, and only the third one held up

The CI pipeline, built with Tekton, shares one storage volume across its steps: clone the
code, build it, package it, push the image. Each step runs in its own pod, and that
shared volume is what lets one step's output become the next step's input.

Occasionally, the step that pushes the finished image would fail because it couldn't find
a file the previous step had just built seconds earlier.

My first theory was that the shared volume wasn't reliably available across different
nodes, since `kind` runs each Kubernetes node as its own separate container and this
storage type is tied to a specific node. I never actually checked that theory against a
real failing run, and it turned out to be wrong.

My second move was to try building a fix based on that theory: force every step of a
build onto the same node. That fix failed immediately to even apply, and the failure
revealed something important. Tekton already solves exactly this problem on its own. It
creates one small helper pod first, then pins every real step of that build to whichever
node the helper landed on. Node placement was never actually the issue. It was already
guaranteed correct the whole time.

The real cause turned up by comparing the start and end time of every failed build
against every other build running around the same time. Every single failure lined up
with a second build running at that exact moment. Both builds share the same volume, and
the very first step of a build always wipes that shared space clean before starting a
fresh checkout. When two builds overlap, one build's cleanup step deletes the other
build's in-progress files out from under it.

Fixed by having the automated trigger check whether a build is already running before
starting a new one, and skip that cycle if so.

### The full rebuild test caught a bug in the fix meant to make the rebuild test possible

To keep the whole environment reproducible, configuration files that matter are tracked
declaratively through Nix, a package and configuration tool, instead of sitting as loose
files that could quietly get lost or drift.

One such file tells the container tool how to reach a private image registry over plain
HTTP instead of the HTTPS it expects by default. I moved that file under Nix's
management. Under the hood, Nix does this by replacing the real file with a symlink
pointing into its own internal storage folder.

That broke image pulls the next time the cluster was rebuilt from scratch, and only
inside the containers acting as Kubernetes nodes. `kind` only gives each node visibility
into the one specific folder it's explicitly told to mount. The symlink itself lived
inside that mounted folder, but the actual file it pointed to, sitting in Nix's own
storage location, was outside it. From inside a fresh node, the file just wasn't there.

This only surfaced by actually deleting the cluster and rebuilding it from nothing. The
file looked completely correct sitting on the host machine, and would have looked correct
in a code review too. Fixed by also giving each node visibility into that Nix storage
location, not just the one config folder.

### A firewall rule was tested before it was trusted

Kubernetes has a resource called `NetworkPolicy` for restricting which pods can talk to
which, similar to a firewall rule between pods. Whether one actually does anything depends
entirely on the specific networking plugin the cluster uses to enforce it. Some plugins
have, at points in their history, accepted a `NetworkPolicy` without enforcing it at all,
silently. That's worse than having no rule, because it looks like protection that isn't
actually there.

So before writing any real policy for this cluster, I tested the mechanism itself first:
created a deny-everything rule in a spare, disposable namespace, then tried reaching a pod
from another pod and confirmed the connection was actually blocked, not just that the
policy object existed without error.

That confirmed this specific cluster's networking plugin enforces it correctly. The point
isn't really that one result. It's the habit: test that a security control actually does
something before depending on it, regardless of what worked on the last cluster or what
the documentation claims.

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
