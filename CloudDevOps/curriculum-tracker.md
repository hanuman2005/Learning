# ☁️ Cloud & DevOps Engineering — 30-Day Curriculum Tracker

> **Goal:** Master the real toolchain a working SDE/DevOps engineer uses to carry code from laptop → production. Hands-on, job-ready, product-company grade.
> **Rule:** 3 hrs/day = ~1.5h learn + ~1.5h **hands-on lab** (DevOps is learned by *doing*, not reading).
> **Cloud:** AWS-primary, with GCP/Azure equivalents noted.
> **Teaching style (same as SystemDesign/AIENGINEER):** every concept starts with an **In plain words** primer → Why/What/When/How/Where → 🤔 misconception hook → ✅ good / ❌ bad scenario → When NOT to.
> **Landscape note (2026):** Terraform has an open-source fork **OpenTofu**; GitHub Actions dominates CI/CD; ArgoCD/Flux lead GitOps. Verify tool versions when you build.

**Progress: S1 ⬜ · S2 ⬜ · S3 ⬜ · S4 ⬜ · S5 ⬜ · S6 ⬜ · S7 ⬜ · S8 ⬜ · S9 ⬜**

---

## The mental model (read first)
Every DevOps tool sits somewhere on the path **code → users**:

`Editor → Git → Docker → CI/CD → Cloud(AWS) → Terraform → Kubernetes → Observability → Security`

Inner loop = what you do locally (edit, commit, containerize). Outer loop = what carries it safely to real users (CI/CD → cloud → orchestration → monitoring). DevOps is mostly mastering the outer loop.

---

## Stage 1 — Linux & Shell (Days 1–3)

### Day 1 — Linux Fundamentals
- [ ] Why Linux runs ~all servers; filesystem hierarchy (/etc, /var, /home, /proc)
- [ ] Navigation & files (ls, cd, cp, mv, rm, find, cat, less, grep)
- [ ] Permissions (rwx, chmod, chown, sudo) — the model behind most "permission denied"
- [ ] Processes (ps, top, htop, kill, signals), systemd services (systemctl)
- [ ] Package managers (apt/yum), env variables, PATH
- [ ] **Lab:** spin up a Linux box (WSL/VM/cloud), navigate, manage a service

### Day 2 — Shell / Bash Scripting
- [ ] Variables, quoting, exit codes, pipes & redirection (| > >> 2>&1)
- [ ] Conditionals, loops, functions, arguments ($1, $@)
- [ ] Text tools: grep, sed, awk, cut, sort, uniq, xargs
- [ ] Writing robust scripts (set -euo pipefail, trap)
- [ ] **Lab:** write a backup/log-parsing script

### Day 3 — Networking & Troubleshooting on Linux
- [ ] TCP/IP refresher, ports, DNS resolution on a box
- [ ] Tools: curl, wget, ss/netstat, ping, dig, nc, traceroute
- [ ] SSH deeply (keys, config, tunnels), scp/rsync
- [ ] Logs & debugging (journalctl, /var/log, tail -f)
- [ ] **Lab:** SSH into a server, diagnose a "service unreachable" problem

---

## Stage 2 — Git & Version Control (Days 4–5)

### Day 4 — Git Core
- [ ] The mental model (working dir → staging → commit → history; Git tracks snapshots)
- [ ] Everyday flow (add, commit, status, log, diff, .gitignore)
- [ ] Branching & merging, merge vs rebase, resolving conflicts
- [ ] Remotes (clone, push, pull, fetch), HEAD, detached HEAD
- [ ] Undo toolkit (reset, revert, restore, reflog, stash)
- [ ] **Lab:** simulate a conflict and resolve it; rebase a feature branch

### Day 5 — Git Collaboration (GitHub/GitLab)
- [ ] Pull/Merge Requests, code review workflow
- [ ] Branching strategies (trunk-based vs GitFlow — trade-offs)
- [ ] Tags, releases, semantic versioning; protected branches
- [ ] Hooks, conventional commits, monorepo vs polyrepo (brief)
- [ ] **Lab:** open a PR, review, merge; set a branch protection rule

---

## Stage 3 — Docker & Containers (Days 6–8)

### Day 6 — Docker Fundamentals
- [ ] The "works on my machine" problem → containers vs VMs
- [ ] Images vs containers; the Docker architecture (daemon, registry)
- [ ] Core commands (run, ps, exec, logs, stop, rm, images, pull)
- [ ] Volumes (persist data) & networking (ports, bridge)
- [ ] **Lab:** run a database + app container, connect them

### Day 7 — Building Images (Dockerfile)
- [ ] Dockerfile instructions (FROM, RUN, COPY, CMD, ENTRYPOINT, ENV, WORKDIR)
- [ ] Layer caching & why order matters; image size optimization
- [ ] **Multi-stage builds** (build vs runtime); distroless/slim bases
- [ ] Tags, registries (Docker Hub, ECR), .dockerignore, security basics
- [ ] **Lab:** write a multi-stage Dockerfile, push to a registry

### Day 8 — Docker Compose & Local Dev
- [ ] Multi-container apps with docker-compose.yml
- [ ] Services, networks, volumes, depends_on, healthchecks, env files
- [ ] Dev vs prod compose; when Compose ends and Kubernetes begins
- [ ] **Lab:** compose a full stack (app + DB + cache) with one command

---

## Stage 4 — CI/CD (Days 9–11)

### Day 9 — CI/CD Concepts
- [ ] What CI vs CD (continuous integration / delivery / deployment) actually mean
- [ ] The pipeline: build → test → scan → package → deploy
- [ ] Why automate: catch bugs early, ship safely, remove human error
- [ ] Artifacts, environments (dev/staging/prod), promotion
- [ ] **Lab:** design a pipeline on paper for a web app

### Day 10 — GitHub Actions (hands-on CI/CD)
- [ ] Workflows, jobs, steps, runners, triggers (on: push/PR)
- [ ] Actions marketplace, secrets, caching, matrix builds
- [ ] Build+test+Docker-build+push pipeline; environments & approvals
- [ ] Jenkins / GitLab CI equivalents (brief comparison table)
- [ ] **Lab:** write a real workflow that tests + builds + pushes an image

### Day 11 — Deployment Strategies & GitOps intro
- [ ] Rolling, blue-green, canary, feature flags — trade-offs
- [ ] Rollbacks & deployment safety; DORA metrics (deploy freq, MTTR, etc.)
- [ ] GitOps concept (Git as source of truth) — ArgoCD/Flux preview
- [ ] **Lab:** add a deploy stage with a manual approval gate

---

## Stage 5 — Cloud & AWS Core (Days 12–16)

### Day 12 — Cloud & AWS Foundations
- [ ] Why cloud (capex→opex, elasticity, managed services); regions/AZs
- [ ] Shared responsibility model; the AWS console/CLI/SDK
- [ ] Compute options: EC2, containers (ECS/EKS/Fargate), Lambda — when each
- [ ] GCP/Azure equivalents noted
- [ ] **Lab:** launch an EC2 instance, SSH in, run your container

### Day 13 — IAM & Security Foundations ⭐
- [ ] Identity & access: users, groups, roles, policies (least privilege)
- [ ] Why most cloud breaches are IAM misconfigurations
- [ ] Roles vs keys, instance profiles, assume-role, MFA
- [ ] Policy structure (effect/action/resource/condition)
- [ ] **Lab:** write a least-privilege policy; give an EC2 role S3 access (no keys)

### Day 14 — Cloud Networking (VPC)
- [ ] VPC, subnets (public/private), route tables, IGW, NAT
- [ ] Security groups vs NACLs; the "why can't it connect?" debugging flow
- [ ] Load balancers (ALB/NLB), Route 53 (DNS), CloudFront (CDN)
- [ ] **Lab:** build a VPC with public+private subnets; put a DB in private

### Day 15 — Core AWS Services
- [ ] Storage: S3 (object), EBS (block), EFS (file) — when each
- [ ] Databases: RDS, DynamoDB, ElastiCache (managed = less ops)
- [ ] Messaging: SQS, SNS, EventBridge (reuse your System Design queues)
- [ ] **Lab:** host static site on S3+CloudFront; provision an RDS instance

### Day 16 — Serverless & Cost
- [ ] Lambda (event-driven compute), API Gateway, when serverless wins/loses
- [ ] Cost model & FinOps basics (tagging, budgets, right-sizing, spot)
- [ ] The "surprise $10k bill" failure mode and how to prevent it
- [ ] **Lab:** deploy a Lambda triggered by an S3 upload

---

## Stage 6 — Infrastructure as Code / Terraform (Days 17–19) ⭐

### Day 17 — IaC & Terraform Fundamentals
- [ ] Why IaC: reproducible, reviewable, version-controlled infra (no console clicking)
- [ ] Terraform vs OpenTofu (the 2026 fork); vs CloudFormation/Pulumi
- [ ] Providers, resources, HCL syntax, terraform init/plan/apply/destroy
- [ ] **Lab:** provision an EC2 + security group in Terraform

### Day 18 — Terraform State, Variables & Modules
- [ ] State (what/why), remote state (S3+DynamoDB lock), the state-corruption danger
- [ ] Variables, outputs, locals, data sources
- [ ] **Modules** (reusable infra), the registry, DRY infra
- [ ] **Lab:** refactor Day 17 into a reusable module with remote state

### Day 19 — Terraform in Practice
- [ ] Workspaces & environments (dev/staging/prod)
- [ ] Plan in CI, apply gating, drift detection, import existing resources
- [ ] Secrets in Terraform (don't commit them!), Terragrunt (brief)
- [ ] **Lab:** run terraform plan in a GitHub Actions pipeline

---

## Stage 7 — Kubernetes (Days 20–24)

### Day 20 — Kubernetes Fundamentals
- [ ] The problem k8s solves (orchestrate/scale/heal many containers)
- [ ] Architecture: control plane (API server, etcd, scheduler) + nodes (kubelet)
- [ ] Pods, the desired-state/reconciliation model
- [ ] kubectl basics; managed k8s (EKS/GKE/AKS)
- [ ] **Lab:** run a local cluster (kind/minikube), deploy your first pod

### Day 21 — Workloads & Networking
- [ ] Deployments, ReplicaSets, rolling updates & rollbacks
- [ ] Services (ClusterIP/NodePort/LoadBalancer), Ingress, DNS
- [ ] Labels, selectors, namespaces
- [ ] **Lab:** deploy an app with a Deployment + Service + Ingress

### Day 22 — Config, State & Storage
- [ ] ConfigMaps & Secrets (and why k8s Secrets aren't truly secret)
- [ ] Volumes, PersistentVolumes/Claims, StatefulSets (databases)
- [ ] Resource requests/limits, liveness/readiness probes
- [ ] **Lab:** externalize config; add probes + resource limits

### Day 23 — Scaling, Packaging & Operations
- [ ] Autoscaling (HPA), node scaling; scheduling (taints/tolerations, affinity)
- [ ] **Helm** (the k8s package manager) — charts, values, releases
- [ ] Health, debugging (kubectl describe/logs/exec), common failures (CrashLoopBackOff)
- [ ] **Lab:** package your app as a Helm chart; set up autoscaling

### Day 24 — GitOps with Kubernetes
- [ ] GitOps done properly: ArgoCD/Flux syncing Git → cluster
- [ ] Infra GitOps (Terraform/OpenTofu) vs app GitOps (ArgoCD) — the clean split
- [ ] Progressive delivery (canary via Argo Rollouts)
- [ ] **Lab:** install ArgoCD, sync an app from a Git repo

---

## Stage 8 — Observability & Monitoring (Days 25–27)

### Day 25 — Metrics & Monitoring
- [ ] The 3 pillars (metrics, logs, traces) — reuse System Design Day 20
- [ ] Prometheus (scraping, PromQL), Grafana (dashboards)
- [ ] RED & USE methods; the four golden signals; why p99 not average
- [ ] **Lab:** scrape app metrics with Prometheus, chart in Grafana

### Day 26 — Logging & Tracing
- [ ] Centralized logging (ELK / Loki), structured logs, log levels
- [ ] Distributed tracing (OpenTelemetry, Jaeger) — follow a request across services
- [ ] What NEVER to log (secrets/PII — reuse System Design/AI security)
- [ ] **Lab:** ship container logs to a central store; add a trace

### Day 27 — Alerting & SRE
- [ ] SLI / SLO / error budgets; alert on symptoms not causes; alert fatigue
- [ ] On-call, runbooks, incident response, blameless postmortems
- [ ] PagerDuty/Opsgenie; Datadog as all-in-one (build vs buy)
- [ ] **Lab:** define an SLO + alert rule for your app

---

## Stage 9 — Security & Secrets (Days 28–30)

### Day 28 — Secrets Management
- [ ] The "secrets in Git" disaster; where secrets should live
- [ ] Vault / AWS Secrets Manager / SSM Parameter Store; rotation
- [ ] Injecting secrets into containers/k8s/CI safely
- [ ] **Lab:** store a DB password in Secrets Manager, inject at runtime

### Day 29 — DevSecOps
- [ ] Shift-left security; image scanning (Trivy), dependency scanning (Snyk)
- [ ] Supply-chain security (SBOM, signing), least-privilege everywhere
- [ ] Secret scanning in CI, IaC scanning (tfsec/Checkov)
- [ ] **Lab:** add image + IaC scanning to your CI pipeline

### Day 30 — Capstone + Interview Prep
- [ ] **Capstone:** take an app → Dockerize → Terraform infra → deploy to k8s → CI/CD pipeline → monitoring + secrets. The whole outer loop, end to end.
- [ ] DevOps interview prep: "walk me through what happens when you `git push`", "design a CI/CD pipeline", "how do you do zero-downtime deploys", debugging scenarios
- [ ] Gap analysis (topic → confidence 1-5 → action)
- [ ] Beyond: service mesh (Istio), multi-region, platform engineering, FinOps deep-dive

---

## 📁 File Index — all 30 days created ✅

| Day | File | Studied? |
|---|---|---|
| 1 | `day-01-linux-fundamentals.md` | ⬜ |
| 2 | `day-02-shell-bash-scripting.md` | ⬜ |
| 3 | `day-03-networking-troubleshooting.md` | ⬜ |
| 4 | `day-04-git-core.md` | ⬜ |
| 5 | `day-05-git-collaboration.md` | ⬜ |
| 6 | `day-06-docker-fundamentals.md` | ⬜ |
| 7 | `day-07-dockerfile-building-images.md` | ⬜ |
| 8 | `day-08-docker-compose.md` | ⬜ |
| 9 | `day-09-cicd-concepts.md` | ⬜ |
| 10 | `day-10-github-actions.md` | ⬜ |
| 11 | `day-11-deployment-strategies-gitops.md` | ⬜ |
| 12 | `day-12-cloud-aws-foundations.md` | ⬜ |
| 13 | `day-13-iam-security.md` | ⬜ |
| 14 | `day-14-cloud-networking-vpc.md` | ⬜ |
| 15 | `day-15-core-aws-services.md` | ⬜ |
| 16 | `day-16-serverless-cost.md` | ⬜ |
| 17 | `day-17-terraform-fundamentals.md` | ⬜ |
| 18 | `day-18-terraform-state-modules.md` | ⬜ |
| 19 | `day-19-terraform-in-practice.md` | ⬜ |
| 20 | `day-20-kubernetes-fundamentals.md` | ⬜ |
| 21 | `day-21-kubernetes-workloads-networking.md` | ⬜ |
| 22 | `day-22-kubernetes-config-state.md` | ⬜ |
| 23 | `day-23-kubernetes-scaling-helm.md` | ⬜ |
| 24 | `day-24-gitops-kubernetes.md` | ⬜ |
| 25 | `day-25-metrics-monitoring.md` | ⬜ |
| 26 | `day-26-logging-tracing.md` | ⬜ |
| 27 | `day-27-alerting-sre.md` | ⬜ |
| 28 | `day-28-secrets-management.md` | ⬜ |
| 29 | `day-29-devsecops.md` | ⬜ |
| 30 | `day-30-capstone-interview.md` | ⬜ |

---

## Topics kept LIGHT (know they exist)
- **Ansible / config management** — fading vs immutable infra + IaC; awareness only.
- **Service mesh (Istio/Linkerd)** — advanced; introduced Day 30.
- **Platform engineering / IDPs (Backstage)** — the emerging layer above; Day 30 mention.

## Daily Ritual
- [ ] Every day ends with a **working lab**, not just reading — DevOps is muscle memory.
- [ ] Explain the day's tool out loud in 2 min (Feynman test).
- [ ] Commit lab work to git: `git commit -m "devops-day XX: <tool>"`.
