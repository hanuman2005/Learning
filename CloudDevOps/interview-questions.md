# DevOps Interview Question Bank

A practice bank for the 30-day Cloud DevOps course in this folder. Use it after you finish each week.

## How to use this file

- Cover the answer. Say your own answer out loud first. Then compare.
- Each question has a level tag: **[Fresher]**, **[Mid]** or **[Senior]**.
- Each question ends with a "Revise" pointer to the day file that teaches the topic.
- Answers are short model answers. In a real interview, add one example from your own practice.
- Model answers show common behaviour. Tools change between versions. When unsure, say "it depends on the version" and name the doc you would check.

## Related notes outside this course

- Azure, nginx and pm2 are covered in `D:\LANDED\learning\16_nginx_pm2_Azure.md`.
- Linux in more depth is covered in `D:\LANDED\learning\03_Linux.md`.

## Contents

1. Top 15 must-know questions
2. Linux and shell (Q1 to Q10)
3. Networking (Q11 to Q16)
4. Git (Q17 to Q22)
5. Docker, Dockerfile and Compose (Q23 to Q35)
6. CI/CD, GitHub Actions, deployment and GitOps (Q36 to Q50)
7. AWS: foundations, IAM, VPC, core services, serverless, cost (Q51 to Q66)
8. Terraform (Q67 to Q74)
9. Kubernetes, Helm and scaling (Q75 to Q87)
10. Observability, alerting and SRE (Q88 to Q95)
11. Secrets and DevSecOps (Q96 to Q101)
12. Hands-on command questions (C1 to C20)
13. Common mistakes in DevOps interviews

---

## 1. Top 15 must-know questions

These come up in almost every DevOps interview. Learn them first.

### T1. What is DevOps, in your own words? [Fresher]
DevOps is a way of working where development and operations share one goal: ship software fast and safely. It uses automation, small changes, fast feedback and shared ownership. It is a culture first and a toolset second.
Revise: day-09-cicd-concepts.md

### T2. What is the difference between CI, continuous delivery and continuous deployment? [Fresher]
CI (continuous integration) means every change is built and tested automatically. Continuous delivery means every passing build is ready to release, but a human presses the button. Continuous deployment releases to production automatically after tests pass.
Revise: day-09-cicd-concepts.md

### T3. How does a container differ from a virtual machine? [Fresher]
A VM has its own guest operating system on top of a hypervisor (software that runs virtual machines). A container shares the host kernel and isolates processes with namespaces and cgroups. So containers start faster and are smaller. They give weaker isolation than a VM.
Revise: day-06-docker-fundamentals.md

### T4. What is the difference between an image and a container? [Fresher]
An image is a read-only template made of layers. A container is a running instance of an image, with a thin writable layer on top. You can start many containers from one image.
Revise: day-06-docker-fundamentals.md

### T5. Explain `git merge` and `git rebase`. When do you use each? [Mid]
Merge joins two branches and keeps the full history, often with a merge commit. Rebase replays your commits on top of another branch to make a straight history. Do not rebase commits that others already pulled, because it rewrites history.
Revise: day-04-git-core.md, day-05-git-collaboration.md

### T6. A service is slow or down. How do you start troubleshooting? [Mid]
Define the symptom and the time it began. Check recent changes first. Then check the layers in order: DNS, network, load balancer, app logs, resources (CPU, memory, disk), and dependencies. Use metrics and logs, not guesses. Roll back if a recent change is the likely cause.
Revise: day-03-networking-troubleshooting.md, day-27-alerting-sre.md

### T7. What is Infrastructure as Code and why use it? [Fresher]
IaC means you describe servers, networks and services in files, not by clicking in a console. The files live in Git. You get review, repeatability, history and fewer manual mistakes. Terraform is a common tool.
Revise: day-17-terraform-fundamentals.md

### T8. What is Terraform state and why is it sensitive? [Mid]
State is a file that maps your code to real resources. Terraform uses it to plan changes. It can contain secret values in plain text. So store it in a remote backend (a shared storage location) with encryption, access control and locking.
Revise: day-18-terraform-state-modules.md

### T9. Explain IAM users, roles and policies in AWS. [Fresher]
A user is a long-term identity for a person or app. A role is an identity that you assume for temporary credentials. A policy is a JSON document that allows or denies actions on resources. Prefer roles and least privilege (give only the access that is needed).
Revise: day-13-iam-security.md

### T10. What is the difference between a public and a private subnet? [Mid]
A public subnet has a route to an internet gateway. A private subnet does not. Private subnets reach the internet, if needed, through a NAT gateway (a service that lets private hosts make outbound calls). Put databases and app servers in private subnets.
Revise: day-14-cloud-networking-vpc.md

### T11. What are a Pod, a Deployment and a Service in Kubernetes? [Fresher]
A Pod is the smallest unit and holds one or more containers. A Deployment manages replicas of Pods and handles rolling updates. A Service gives a stable name and IP that load-balances to matching Pods.
Revise: day-20-kubernetes-fundamentals.md, day-21-kubernetes-workloads-networking.md

### T12. A pod is in CrashLoopBackOff. What do you do? [Mid]
See Q78 for the full steps. In short: run `kubectl describe pod`, read `kubectl logs --previous`, check the exit code, probes, config and resource limits.
Revise: day-21-kubernetes-workloads-networking.md

### T13. What are the three pillars of observability? [Mid]
Metrics (numbers over time), logs (event records) and traces (the path of one request across services). Together they show that something is wrong, where, and why.
Revise: day-25-metrics-monitoring.md, day-26-logging-tracing.md

### T14. What is an SLI, an SLO and an error budget? [Mid]
An SLI (service level indicator) is a measurement, such as the share of successful requests. An SLO (service level objective) is the target for it, such as 99.9 percent. The error budget is the allowed failure, 100 percent minus the SLO. It guides how much risk you can take.
Revise: day-27-alerting-sre.md

### T15. How do you keep secrets out of code and pipelines? [Mid]
Never commit them. Use a secrets manager, short-lived credentials and OIDC (a way for the pipeline to get temporary cloud access without stored keys). Scan for leaks, rotate keys, and grant least privilege.
Revise: day-28-secrets-management.md, day-29-devsecops.md

---

## 2. Linux and shell

### Q1. What does `chmod 640 file` mean? [Fresher]
The owner can read and write (6 = rw-). The group can read (4 = r--). Others have no access (0 = ---). Each digit adds read 4, write 2, execute 1.
Revise: day-01-linux-fundamentals.md

### Q2. What is the difference between `systemctl start` and `systemctl enable`? [Fresher]
`start` runs the service now. `enable` makes it start at boot. You often need both. `enable --now` does both in one command.
Revise: day-01-linux-fundamentals.md

### Q3. What is the difference between `kill -15` and `kill -9`? [Fresher]
`kill -15` sends SIGTERM (a polite request to stop). The process can clean up and exit. `kill -9` sends SIGKILL, which cannot be caught. Use -15 first. Use -9 only when the process ignores it.
Revise: day-01-linux-fundamentals.md

### Q4. A server shows "No space left on device" but `df -h` shows free space. Why? [Mid]
Inodes (the records that track files) may be full. Check `df -i`. Another cause is a deleted file still held open by a process. `lsof +L1` finds such files. Restarting that process frees the space.
Revise: day-01-linux-fundamentals.md

### Q5. The load average is high. How do you find the cause? [Mid]
Run `uptime` and `top` or `htop`. Check if CPU, swap or disk wait (`iostat`, the `wa` value in top) is high. Load counts waiting processes, so slow disks raise it too. Then find the top process with `ps aux --sort=-%cpu`.
Revise: day-01-linux-fundamentals.md

### Q6. What is a zombie process? [Mid]
A zombie has finished, but its parent has not read its exit status. It shows as `Z` in `ps`. You cannot kill it, because it is already dead. Fix or stop the parent, so init cleans it up.
Revise: day-01-linux-fundamentals.md

### Q7. What does `set -euo pipefail` do in a Bash script? [Mid]
`-e` stops the script when a command fails. `-u` treats unset variables as errors. `pipefail` makes a pipeline fail if any part fails. Together they catch many silent bugs. They have edge cases, so test your script.
Revise: day-02-shell-bash-scripting.md

### Q8. Why quote variables in shell, as in `"$var"`? [Fresher]
Without quotes, the shell splits the value on spaces and expands wildcards. A file named `my file` becomes two arguments. Quotes keep it as one value.
Revise: day-02-shell-bash-scripting.md

### Q9. Explain `grep`, `awk` and `sed` in one line each. [Mid]
`grep` finds lines that match a pattern. `awk` splits lines into fields and processes them, such as summing a column. `sed` edits a stream of text, such as replacing words. Example: `awk '{print $9}' access.log | sort | uniq -c`.
Revise: day-02-shell-bash-scripting.md

### Q10. What is the difference between `>`, `>>` and `2>&1`? [Fresher]
`>` writes output to a file and overwrites it. `>>` appends. `2>&1` sends errors (file descriptor 2) to the same place as normal output (descriptor 1). Use `cmd > out.log 2>&1` to capture both.
Revise: day-02-shell-bash-scripting.md

---

## 3. Networking

### Q11. What happens when you type a URL in a browser? [Fresher]
The browser resolves the name through DNS to an IP address. It opens a TCP connection, then a TLS handshake for HTTPS. It sends an HTTP request. The server replies, and the browser renders the page. A load balancer or proxy often sits in between.
Revise: day-03-networking-troubleshooting.md

### Q12. What is the difference between TCP and UDP? [Fresher]
TCP is connection-based and reliable. It orders data and retries lost packets. UDP is connectionless and has no delivery guarantee, but it is faster. DNS queries and video streams often use UDP.
Revise: day-03-networking-troubleshooting.md

### Q13. How do you debug "connection refused" versus "connection timed out"? [Mid]
Refused means the host answered, but nothing listens on that port, or a firewall rejected it. Timed out means packets got no answer, often a firewall drop or a routing problem. Check `ss -tlnp` on the server. Test from the client with `nc -vz host port` or `curl -v`.
Revise: day-03-networking-troubleshooting.md

### Q14. Scenario: users get 502 Bad Gateway after a deploy. What do you check? [Mid]
A 502 means the proxy or load balancer got a bad reply from the upstream app. 1) Is the app running and listening on the expected port? 2) Do health checks pass? 3) Check app logs for crashes or startup errors. 4) Check proxy upstream settings and timeouts. 5) Compare with the last release. Roll back if needed.
Revise: day-03-networking-troubleshooting.md, day-11-deployment-strategies-gitops.md

### Q15. What is DNS, and how do you debug a DNS problem? [Mid]
DNS turns names into IP addresses. Use `dig example.com` to see the answer, the TTL (how long it is cached) and the server that replied. Use `dig @8.8.8.8 name` to compare resolvers. Check for stale cache, wrong records and the `/etc/resolv.conf` file.
Revise: day-03-networking-troubleshooting.md

### Q16. What is the difference between a 4xx and a 5xx error? [Fresher]
4xx means the client made a mistake, such as 404 not found or 403 forbidden. 5xx means the server failed, such as 500 or 503. A 502 or 504 often points to a problem between proxy and backend.
Revise: day-03-networking-troubleshooting.md

---

## 4. Git

### Q17. What is the difference between `git fetch` and `git pull`? [Fresher]
`fetch` downloads new commits but does not change your working branch. `pull` is a fetch followed by a merge (or a rebase, if configured). Fetch first when you want to look before you merge.
Revise: day-05-git-collaboration.md

### Q18. How do you undo a commit that is already pushed? [Mid]
Use `git revert <sha>`. It adds a new commit that reverses the change and keeps history safe. Avoid `reset --hard` plus force push on shared branches, because it rewrites history for others.
Revise: day-04-git-core.md

### Q19. What do `git reset --soft`, `--mixed` and `--hard` do? [Mid]
All move the branch pointer. `--soft` keeps changes staged. `--mixed` (default) keeps changes but unstages them. `--hard` discards changes in the working tree. Be careful with `--hard`.
Revise: day-04-git-core.md

### Q20. You committed a secret by mistake. What do you do? [Mid]
Treat the secret as leaked. Revoke and rotate it first. Then remove it from history with a history-rewriting tool and coordinate a force push. Add a secret scan to CI. Deleting it in a new commit is not enough.
Revise: day-28-secrets-management.md, day-05-git-collaboration.md

### Q21. Explain a branching strategy such as trunk-based development. [Senior]
In trunk-based development, everyone merges small changes to one main branch often, behind feature flags if needed. It suits CI and fast delivery. Long-lived branches (as in Gitflow) give more control but cause big merges and slower feedback. Choose based on release needs.
Revise: day-05-git-collaboration.md

### Q22. How do you resolve a merge conflict? [Fresher]
Git marks conflict areas in the file with `<<<<<<<`, `=======` and `>>>>>>>`. Edit the file to the right result, remove the markers, run `git add`, then finish the merge or rebase. Run tests afterward.
Revise: day-04-git-core.md

---

## 5. Docker, Dockerfile and Compose

### Q23. What are namespaces and cgroups? [Mid]
Namespaces limit what a process can see: its own process list, network and file system view. Cgroups (control groups) limit what it can use: CPU, memory and I/O. Containers are made from these two Linux features.
Revise: day-06-docker-fundamentals.md

### Q24. Data disappears when a container is removed. How do you keep it? [Fresher]
The container writable layer is deleted with the container. Use a volume (storage managed by Docker) or a bind mount (a host folder) for data that must stay. Databases should use volumes.
Revise: day-06-docker-fundamentals.md

### Q25. What is the difference between a volume and a bind mount? [Mid]
A volume is managed by Docker and stored in a Docker area. It is the preferred way to persist data. A bind mount maps an exact host path into the container. It is handy for local development, but it depends on the host layout.
Revise: day-06-docker-fundamentals.md

### Q26. Scenario: a container exits right after start. How do you debug? [Mid]
Run `docker ps -a` to see the exit code. Read `docker logs <name>`. Exit code 137 often means it was killed (for example, out of memory). Run the image with `docker run -it --entrypoint sh image` to look inside. Check that the main process stays in the foreground.
Revise: day-06-docker-fundamentals.md

### Q27. `CMD` versus `ENTRYPOINT`: what is the difference? [Mid]
`ENTRYPOINT` sets the main program. `CMD` sets default arguments, or a default command if there is no entrypoint. Arguments given to `docker run` replace `CMD`. They do not replace `ENTRYPOINT` unless you use `--entrypoint`.
Revise: day-07-dockerfile-building-images.md

### Q28. What is `COPY` versus `ADD`? [Fresher]
`COPY` copies local files into the image. `ADD` does the same, and it can also unpack local tar files and fetch URLs. Best practice is to use `COPY` unless you need the extra features.
Revise: day-07-dockerfile-building-images.md

### Q29. How does Docker layer caching work, and how do you use it? [Mid]
Each instruction makes a layer. Docker reuses a layer if the instruction and its inputs did not change. Once one layer changes, all later layers rebuild. So copy dependency files first, install, then copy source code.
Revise: day-07-dockerfile-building-images.md

### Q30. What is a multi-stage build and why use it? [Mid]
You use several `FROM` stages in one Dockerfile. The first stage builds the app with compilers and tools. The final stage copies only the result. The final image is smaller and has fewer security risks.
Revise: day-07-dockerfile-building-images.md

### Q31. How do you make a Docker image smaller and safer? [Senior]
Use a small base image. Use multi-stage builds. Add a `.dockerignore` file. Run as a non-root user. Pin versions. Do not copy secrets into layers, because layers keep history. Scan the image for known vulnerabilities.
Revise: day-07-dockerfile-building-images.md, day-29-devsecops.md

### Q32. Why should containers not run as root? [Mid]
If an attacker escapes the app, root inside the container gives more power and may help a container breakout. A non-root user limits damage. Use the `USER` instruction and read-only file systems where possible.
Revise: day-07-dockerfile-building-images.md

### Q33. What does Docker Compose do? [Fresher]
Compose describes a multi-container app in one YAML file: services, networks, volumes and environment values. You start everything with `docker compose up`. It is good for local development and small setups.
Revise: day-08-docker-compose.md

### Q34. In Compose, how does one service reach another? [Mid]
Compose puts services on a shared network, and each service name works as a DNS name. The app connects to `db:5432`, not `localhost`. `localhost` inside a container means that same container.
Revise: day-08-docker-compose.md

### Q35. Does `depends_on` wait until the database is ready? [Mid]
By default it only controls start order. It does not wait for readiness. To wait for a healthy service, define a `healthcheck` and use `depends_on` with `condition: service_healthy`. Also make the app retry its connection.
Revise: day-08-docker-compose.md

---

## 6. CI/CD, GitHub Actions, deployment and GitOps

### Q36. What stages are in a typical CI/CD pipeline? [Fresher]
Checkout code, build, unit tests, lint and security scans, package (such as a container image), push to a registry, deploy to staging, run checks, then deploy to production. Fast stages run first, so failures show early.
Revise: day-09-cicd-concepts.md

### Q37. Scenario: the pipeline is slow. How do you speed it up? [Mid]
Measure which stage takes most time. Then: cache dependencies and Docker layers, run independent jobs in parallel, run only needed tests for the change, use faster runners, and fail fast. Split large test suites. Remove steps that add no value.
Revise: day-09-cicd-concepts.md, day-10-github-actions.md

### Q38. What is a flaky test and how do you handle it? [Mid]
A flaky test passes and fails without code changes. Causes are timing, shared state and outside services. Find and fix the root cause. Quarantine the test while you fix it, and track it. Do not just re-run until green.
Revise: day-09-cicd-concepts.md

### Q39. What is the difference between a build artifact and a deployment? [Fresher]
An artifact is the built output, such as an image or a package. A deployment puts that artifact into an environment. Build once, then deploy the same artifact to every environment, so what you tested is what you ship.
Revise: day-09-cicd-concepts.md

### Q40. Explain a GitHub Actions workflow: events, jobs, steps and runners. [Fresher]
A workflow is a YAML file in `.github/workflows`. An event, such as `push` or `pull_request`, triggers it. It has jobs, and each job runs on a runner (a machine). Jobs have steps that run commands or reuse actions. Jobs run in parallel unless you use `needs`.
Revise: day-10-github-actions.md

### Q41. How do you pass data and order between jobs? [Mid]
Use `needs` to set order. Use job `outputs` for small values. Use artifacts (upload and download actions) for files. Each job starts on a clean runner, so files do not carry over by themselves.
Revise: day-10-github-actions.md

### Q42. How do you use secrets safely in GitHub Actions? [Mid]
Store them as repository, environment or organization secrets, and read them with `${{ secrets.NAME }}`. GitHub masks secret values in logs, but masking is not perfect. Limit access with environments and approvals. Prefer OIDC to cloud providers over long-lived keys. Be careful with workflows from forks.
Revise: day-10-github-actions.md, day-28-secrets-management.md

### Q43. How do you cache dependencies in GitHub Actions? [Mid]
Use the cache action with a key built from the lock file hash, or a setup action that has built-in caching. When the lock file changes, the key changes and a new cache is saved. Cache has size and age limits, so check the current docs.
Revise: day-10-github-actions.md

### Q44. What is a matrix build? [Mid]
A matrix runs the same job with many value combinations, such as several language versions or operating systems. It saves repeated YAML. Use `fail-fast` and `max-parallel` to control behaviour and cost.
Revise: day-10-github-actions.md

### Q45. Compare rolling, blue-green and canary deployments. [Mid]
Rolling replaces instances a few at a time. Blue-green runs two full environments and switches traffic at once, so rollback is fast but cost is higher. Canary sends a small share of traffic to the new version first, then grows it if metrics look good.
Revise: day-11-deployment-strategies-gitops.md

### Q46. A release is bad in production. What is your rollback plan? [Mid]
Decide fast using metrics. Redeploy the previous known-good artifact, or switch traffic back in blue-green. For Kubernetes, use `kubectl rollout undo`. Think about database changes: use backward-compatible migrations so rollback is safe. Then write a post-incident review.
Revise: day-11-deployment-strategies-gitops.md

### Q47. What is GitOps? [Mid]
GitOps uses Git as the source of truth for the desired state of infrastructure and apps. An agent in the cluster pulls the state and keeps the cluster in sync. Changes happen by pull request. You get audit history and easy rollback by reverting a commit.
Revise: day-11-deployment-strategies-gitops.md, day-24-gitops-kubernetes.md

### Q48. What is the difference between push-based and pull-based deployment? [Senior]
In push-based CD, the pipeline holds cluster credentials and pushes changes in. In pull-based GitOps, an agent inside the cluster pulls from Git. Pull-based keeps credentials inside, and it corrects drift (manual changes) automatically.
Revise: day-24-gitops-kubernetes.md

### Q49. What are feature flags and how do they help? [Mid]
A flag turns a feature on or off without a deploy. It separates deploying code from releasing a feature. You can roll out to a few users, and switch off fast if there is a problem. Remove old flags to avoid clutter.
Revise: day-11-deployment-strategies-gitops.md

### Q50. How do you do database changes with zero downtime? [Senior]
Use the expand and contract pattern. First add new columns or tables in a backward-compatible way. Deploy code that works with both. Migrate data. Later, remove the old structure. Never ship a change that breaks the version still running.
Revise: day-11-deployment-strategies-gitops.md

---

## 7. AWS

### Q51. What are a Region and an Availability Zone? [Fresher]
A Region is a geographic area with several groups of data centers. An Availability Zone (AZ) is one or more separate data centers inside a Region. Spread your app over at least two AZs for high availability.
Revise: day-12-cloud-aws-foundations.md

### Q52. What is the shared responsibility model? [Fresher]
The cloud provider secures the cloud itself: hardware, facilities and core services. You secure what you put in the cloud: data, identities, configuration and your code. The split changes by service type.
Revise: day-12-cloud-aws-foundations.md

### Q53. IaaS, PaaS and serverless: what is the difference? [Fresher]
IaaS gives you virtual machines and networks, and you manage the OS. PaaS manages the OS and runtime, and you manage the app. Serverless runs your code on events and you do not manage servers at all.
Revise: day-12-cloud-aws-foundations.md, day-16-serverless-cost.md

### Q54. Explain IAM least privilege with an example. [Mid]
Give an identity only the actions it needs, on only the resources it needs. For an app that reads one S3 bucket, allow `s3:GetObject` on that bucket ARN (the unique resource name), not `s3:*` on `*`. Review and tighten over time.
Revise: day-13-iam-security.md

### Q55. How does an IAM policy evaluation work? [Senior]
By default, everything is denied. An explicit allow grants access. An explicit deny always wins over any allow. Other layers, such as permission boundaries and service control policies, can also limit the result.
Revise: day-13-iam-security.md

### Q56. An EC2 app needs to read S3. How do you give access safely? [Mid]
Attach an IAM role to the instance through an instance profile. The app gets temporary credentials from the instance metadata service. Do not store access keys on the box.
Revise: day-13-iam-security.md

### Q57. Scenario: an S3 bucket is found publicly exposed. What do you do? [Mid]
Act fast. Turn on S3 Block Public Access for the bucket and the account. Check the bucket policy and ACLs (access control lists). Review access logs and CloudTrail to see who read data. Decide if data was sensitive, rotate any exposed keys, and report per your process. Then add guardrails such as Config rules.
Revise: day-13-iam-security.md, day-15-core-aws-services.md

### Q58. What makes a VPC subnet public? [Mid]
Its route table has a route to an internet gateway, and instances have public IPs. Without both, it is not reachable from the internet. A private subnet has no such route and uses NAT for outbound traffic.
Revise: day-14-cloud-networking-vpc.md

### Q59. Security group versus network ACL? [Mid]
A security group works at the instance level and is stateful: return traffic is allowed automatically. It only has allow rules. A network ACL works at the subnet level, is stateless, and supports allow and deny rules in numbered order.
Revise: day-14-cloud-networking-vpc.md

### Q60. Scenario: an EC2 instance in a private subnet cannot reach the internet. What do you check? [Mid]
Check the route table has a default route to a NAT gateway. Check the NAT gateway sits in a public subnet with an internet gateway route. Check the security group and network ACL outbound rules. Check DNS settings. Test with `curl -v`.
Revise: day-14-cloud-networking-vpc.md

### Q61. What is VPC peering and what is a limit? [Senior]
Peering connects two VPCs over private IPs. CIDR ranges (IP address blocks) must not overlap. It is not transitive: if A peers with B and B with C, A cannot reach C. For many VPCs, consider a transit gateway.
Revise: day-14-cloud-networking-vpc.md

### Q62. What is an Application Load Balancer and what do target groups do? [Mid]
An ALB works at layer 7 (HTTP) and routes by path or host. It sends traffic to target groups, which hold instances, IPs or Lambda functions. Health checks decide which targets get traffic.
Revise: day-15-core-aws-services.md

### Q63. Explain S3 storage classes and versioning. [Mid]
Classes trade cost against access speed, such as Standard, Infrequent Access and Glacier types. Lifecycle rules move data between them. Versioning keeps old versions of objects, so you can recover from deletes and overwrites.
Revise: day-15-core-aws-services.md

### Q64. When would you use RDS Multi-AZ versus a read replica? [Senior]
Multi-AZ keeps a standby copy in another AZ for failover and high availability. A read replica serves read traffic to scale reads, and may lag. They solve different problems. Both can be used together.
Revise: day-15-core-aws-services.md

### Q65. When is AWS Lambda a good fit, and what are its limits? [Mid]
Good for event-driven, short tasks with uneven traffic: file processing, APIs, scheduled jobs. Limits include a maximum run time, cold starts (delay on first run) and less control over the runtime. Check current quotas in the docs.
Revise: day-16-serverless-cost.md

### Q66. How do you reduce an AWS bill? [Mid]
First find the cost with Cost Explorer and tags. Right-size instances, stop idle resources, delete unused volumes and snapshots, use autoscaling, and use savings plans or reserved capacity for steady load. Set budgets and alerts.
Revise: day-16-serverless-cost.md

---

## 8. Terraform

### Q67. Explain the Terraform workflow. [Fresher]
`terraform init` downloads providers and sets the backend. `terraform plan` shows what will change. `terraform apply` makes the change. `terraform destroy` removes resources. Always read the plan before you apply.
Revise: day-17-terraform-fundamentals.md

### Q68. What is a provider, a resource and a data source? [Fresher]
A provider is a plugin that talks to an API, such as AWS. A resource is something Terraform creates and manages. A data source only reads existing information, such as an AMI ID.
Revise: day-17-terraform-fundamentals.md

### Q69. What is the difference between variables, locals and outputs? [Fresher]
Variables are inputs from the user. Locals are named values computed inside the module. Outputs expose values to the user or to other modules. Use outputs to pass IDs between modules.
Revise: day-17-terraform-fundamentals.md

### Q70. Why use a remote backend with locking? [Mid]
A remote backend (for example S3) lets a team share one state. Locking stops two people from applying at the same time, which could corrupt state. Turn on encryption and versioning for the state storage. Check the current docs for the locking method your backend supports.
Revise: day-18-terraform-state-modules.md

### Q71. Scenario: Terraform says the state is locked, or two people changed state. What do you do? [Senior]
First check who holds the lock. Often a run is still in progress, or a CI job crashed. Wait, or confirm no one is applying. Only then use `terraform force-unlock <ID>`. If state is out of date, run `terraform plan` or `terraform refresh` style checks and compare. Use state versioning to restore a good copy. Prevent repeats by running apply only from CI.
Revise: day-18-terraform-state-modules.md, day-19-terraform-in-practice.md

### Q72. What is drift and how do you find it? [Mid]
Drift means real infrastructure differs from your code, often because someone changed it by hand. `terraform plan` shows the difference. Fix by reverting the manual change or by updating the code. Stop it with access controls.
Revise: day-19-terraform-in-practice.md

### Q73. What do `terraform import` and `terraform state mv` do? [Senior]
`import` brings an existing resource under Terraform control, so you must also write its configuration. `state mv` renames or moves a resource address in state without destroying it. Both change state, so back it up first.
Revise: day-18-terraform-state-modules.md

### Q74. How do you structure Terraform for many environments? [Senior]
Write reusable modules with clear inputs and outputs. Keep a separate state per environment, so a mistake in dev cannot touch production. Use separate folders or workspaces with care. Pin provider and module versions. Run plan in pull requests and apply after review.
Revise: day-18-terraform-state-modules.md, day-19-terraform-in-practice.md

---

## 9. Kubernetes, Helm and scaling

### Q75. What are the main parts of a Kubernetes cluster? [Fresher]
The control plane has the API server, etcd (the data store), the scheduler and controller managers. Worker nodes run the kubelet (the node agent), a container runtime and kube-proxy. You talk to the API server with `kubectl`.
Revise: day-20-kubernetes-fundamentals.md

### Q76. What is the difference between a Deployment and a StatefulSet? [Mid]
A Deployment is for stateless Pods that are interchangeable. A StatefulSet gives each Pod a stable name, stable storage and ordered start. Use it for databases and similar apps. Many teams use managed databases instead.
Revise: day-21-kubernetes-workloads-networking.md

### Q77. Explain Service types: ClusterIP, NodePort, LoadBalancer. [Mid]
ClusterIP is reachable only inside the cluster. NodePort opens a port on every node. LoadBalancer asks the cloud for an external load balancer. An Ingress routes HTTP traffic to many Services through one entry point.
Revise: day-21-kubernetes-workloads-networking.md

### Q78. Scenario: a pod is in CrashLoopBackOff. Walk me through it. [Mid]
CrashLoopBackOff means the container starts, fails and is restarted with growing delays.
1. `kubectl describe pod <pod>`: read Events, the last state and the exit code.
2. `kubectl logs <pod> --previous`: see the logs of the crashed run.
3. Exit code 137 often means OOMKilled (out of memory). Check limits.
4. Check config: wrong env vars, missing Secret or ConfigMap, bad command.
5. Check that the liveness probe is not killing a slow-starting app.
Fix, then watch with `kubectl get pods -w`.
Revise: day-21-kubernetes-workloads-networking.md, day-22-kubernetes-config-state.md

### Q79. Scenario: a pod is Pending. What do you check? [Mid]
Run `kubectl describe pod`. The Events often say why: not enough CPU or memory on nodes, node selectors or taints that do not match, or an unbound PersistentVolumeClaim. Fix the resource request, add nodes, or fix the scheduling rules.
Revise: day-21-kubernetes-workloads-networking.md

### Q80. Scenario: a pod shows ImagePullBackOff. Why? [Fresher]
Kubernetes cannot pull the image. Check the image name and tag for typos. Check the registry is reachable. For a private registry, check the image pull secret. `kubectl describe pod` shows the exact error.
Revise: day-21-kubernetes-workloads-networking.md

### Q81. Scenario: a Service has no traffic. How do you debug? [Mid]
Check that the Service selector matches the Pod labels. Run `kubectl get endpoints <svc>`. An empty list means no matching ready Pods. Check that readiness probes pass and the target port is right. Test from another Pod with `curl`.
Revise: day-21-kubernetes-workloads-networking.md

### Q82. Liveness, readiness and startup probes: what is the difference? [Mid]
Liveness decides if the container should be restarted. Readiness decides if the Pod should get traffic. Startup protects slow-starting apps from early liveness failure. A bad liveness probe can cause restart loops.
Revise: day-21-kubernetes-workloads-networking.md

### Q83. What are requests and limits? [Mid]
A request is the amount the scheduler reserves for the Pod. A limit is the maximum it may use. Going over a memory limit gets the container killed. Going over a CPU limit causes throttling (slowing down).
Revise: day-22-kubernetes-config-state.md, day-23-kubernetes-scaling-helm.md

### Q84. ConfigMap versus Secret? Are Secrets encrypted? [Mid]
Both inject configuration. A Secret is meant for sensitive data. By default, Secrets are only base64 encoded in etcd, which is not encryption. Turn on encryption at rest, limit access with RBAC (role-based access control), or use an external secrets tool.
Revise: day-22-kubernetes-config-state.md, day-28-secrets-management.md

### Q85. What is a PersistentVolume and a PersistentVolumeClaim? [Mid]
A PersistentVolume (PV) is a piece of storage. A PersistentVolumeClaim (PVC) is a Pod's request for storage. A StorageClass can create PVs on demand. Data survives Pod restarts.
Revise: day-22-kubernetes-config-state.md

### Q86. How does the Horizontal Pod Autoscaler work? [Mid]
The HPA watches metrics, such as CPU use against the requested amount, and changes the replica count to hit a target. It needs a metrics source and resource requests to be set. Cluster autoscaling adds nodes when Pods cannot fit.
Revise: day-23-kubernetes-scaling-helm.md

### Q87. What is Helm and why use it? [Mid]
Helm is a package manager for Kubernetes. A chart holds templates and default values. You install a release with your own values. You can upgrade and roll back a release (`helm rollback`). It avoids copying YAML for each environment.
Revise: day-23-kubernetes-scaling-helm.md

---

## 10. Observability, alerting and SRE

### Q88. What is Prometheus and how does it collect data? [Mid]
Prometheus is a metrics system. It mainly pulls (scrapes) metrics from HTTP endpoints at set intervals and stores them as time series. You query with PromQL. Grafana is often used to draw dashboards.
Revise: day-25-metrics-monitoring.md

### Q89. What are the golden signals? [Mid]
Latency, traffic, errors and saturation. They give a quick health view of any service. The RED method (rate, errors, duration) suits services. The USE method (utilization, saturation, errors) suits resources.
Revise: day-25-metrics-monitoring.md, day-27-alerting-sre.md

### Q90. Why use percentiles, such as p99, and not the average? [Senior]
The average hides slow requests. If 1 in 100 users waits 10 seconds, the average still looks fine. p99 shows the slow tail that real users feel. Use histograms to compute percentiles.
Revise: day-25-metrics-monitoring.md

### Q91. What is structured logging and why does it help? [Mid]
Structured logs are written as key-value data, often JSON, not free text. Tools can filter and count them by field, such as `user_id` or `status`. Add a request or trace ID so you can follow one request.
Revise: day-26-logging-tracing.md

### Q92. What is distributed tracing? [Mid]
A trace follows one request through many services. It is made of spans, each with a start time and a duration. Trace context is passed in headers. It shows which service adds delay. OpenTelemetry is a common standard.
Revise: day-26-logging-tracing.md

### Q93. What makes a good alert? [Senior]
It is urgent, it needs human action, and it points to user impact. Alert on symptoms, such as error rate, more than on causes. Add a link to a runbook. Too many noisy alerts cause alert fatigue, and people then miss real problems.
Revise: day-27-alerting-sre.md

### Q94. What is a blameless post-incident review? [Mid]
After an incident, the team writes a timeline, the impact, the root causes and follow-up actions. It focuses on systems and process, not on blaming a person. The goal is learning and fewer repeats.
Revise: day-27-alerting-sre.md

### Q95. What is toil, and how do SREs handle it? [Senior]
Toil is manual, repetitive work that grows with the service and has no lasting value. SREs automate it. They also use error budgets: if the budget is spent, focus moves from new features to reliability.
Revise: day-27-alerting-sre.md

---

## 11. Secrets and DevSecOps

### Q96. Where should secrets be stored? [Mid]
In a secrets manager, such as AWS Secrets Manager, Parameter Store or Vault. Apps fetch secrets at run time using an identity, not a stored password. Avoid secrets in Git, images and plain environment files. Audit access.
Revise: day-28-secrets-management.md

### Q97. Why rotate secrets and use short-lived credentials? [Mid]
If a secret leaks, a short life limits the damage. Rotation replaces old values on a schedule or after an incident. Dynamic secrets are created per request and expire. Roles and OIDC avoid long-lived keys.
Revise: day-28-secrets-management.md

### Q98. What does "shift left" mean in DevSecOps? [Fresher]
It means finding security problems early in the lifecycle, such as in the editor, in the pull request and in CI, when fixes are cheap. It does not replace later checks. It adds checks earlier.
Revise: day-29-devsecops.md

### Q99. Name the common security scans in a pipeline. [Mid]
SAST (static analysis of source code), dependency or SCA scanning (known bad libraries), secret scanning, container image scanning, and IaC scanning for Terraform. DAST (testing a running app) comes later. Tune results to reduce false positives.
Revise: day-29-devsecops.md

### Q100. What is a software bill of materials (SBOM)? [Senior]
An SBOM is a list of all components inside your software, with versions. If a new vulnerability appears, you can quickly see which apps are affected. Signing images adds proof of where an image came from.
Revise: day-29-devsecops.md

### Q101. Describe your capstone project in two minutes. [Mid]
Use this shape: the problem, the architecture, the pipeline, the infrastructure code, the deployment, the monitoring, and one thing that went wrong and how you fixed it. Be honest about what you built yourself. Practise with a timer.
Revise: day-30-capstone-interview.md

---

## 12. Hands-on command questions

Say the command, then say what you expect to see. Option names can differ slightly by version.

### C1. Which command shows which process listens on which port? [Fresher]
```
ss -tlnp
```
`-t` is TCP, `-l` is listening, `-n` is numbers, `-p` is process (needs root for others' processes).
Revise: day-03-networking-troubleshooting.md

### C2. Which command shows disk space and inode use? [Fresher]
```
df -h
df -i
```
Revise: day-01-linux-fundamentals.md

### C3. Which command shows the last 100 lines of a service log from systemd and follows it? [Fresher]
```
journalctl -u nginx -n 100 -f
```
Revise: day-01-linux-fundamentals.md

### C4. Which command shows the top 10 memory-using processes? [Fresher]
```
ps aux --sort=-%mem | head -n 11
```
The first line is the header, so use 11.
Revise: day-01-linux-fundamentals.md

### C5. Which command checks an HTTP endpoint with headers, status and timing? [Mid]
```
curl -sv -o /dev/null -w "%{http_code} %{time_total}s\n" https://example.com/health
```
Revise: day-03-networking-troubleshooting.md

### C6. Which command shows the DNS answer for a name using a chosen resolver? [Mid]
```
dig @8.8.8.8 example.com A +short
```
Revise: day-03-networking-troubleshooting.md

### C7. Which command shows a short, graphical Git history of all branches? [Fresher]
```
git log --oneline --graph --all
```
Revise: day-04-git-core.md

### C8. Which command shows who changed each line of a file? [Fresher]
```
git blame path/to/file
```
Revise: day-04-git-core.md

### C9. Which command finds the commit that introduced a bug? [Senior]
```
git bisect start
git bisect bad
git bisect good <known-good-sha>
```
Git then checks out middle commits. You mark each as good or bad until it finds the first bad one. End with `git bisect reset`.
Revise: day-04-git-core.md

### C10. Which command shows running and stopped containers, and then the logs of one? [Fresher]
```
docker ps -a
docker logs --tail 100 -f <container>
```
Revise: day-06-docker-fundamentals.md

### C11. Which command opens a shell in a running container? [Fresher]
```
docker exec -it <container> sh
```
Use `bash` if the image has it.
Revise: day-06-docker-fundamentals.md

### C12. Which command shows image layers and their sizes? [Mid]
```
docker history <image>
```
Revise: day-07-dockerfile-building-images.md

### C13. Which command shows the Terraform plan and saves it to apply later? [Mid]
```
terraform plan -out=tfplan
terraform apply tfplan
```
Revise: day-17-terraform-fundamentals.md

### C14. Which command lists resources in Terraform state? [Mid]
```
terraform state list
terraform state show <address>
```
Revise: day-18-terraform-state-modules.md

### C15. Which command shows which AWS identity your CLI is using? [Fresher]
```
aws sts get-caller-identity
```
It shows the account ID, the user or role ARN and the user ID. Use it first when you get "access denied".
Revise: day-13-iam-security.md

### C16. Which command lists your EC2 instances in a clear table? [Mid]
```
aws ec2 describe-instances \
  --query "Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]" \
  --output table
```
Revise: day-15-core-aws-services.md

### C17. Which command shows why a Kubernetes pod is not healthy? [Fresher]
```
kubectl describe pod <pod>
kubectl logs <pod> --previous
```
Read the Events section at the end of `describe`.
Revise: day-21-kubernetes-workloads-networking.md

### C18. Which command shows Pods with their node and IP, in all namespaces? [Fresher]
```
kubectl get pods -A -o wide
```
Revise: day-20-kubernetes-fundamentals.md

### C19. Which commands roll out, check and undo a Deployment change? [Mid]
```
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
```
Revise: day-21-kubernetes-workloads-networking.md

### C20. Which commands show Pod CPU and memory use, and list Helm releases? [Mid]
```
kubectl top pods
helm list -A
```
`kubectl top` needs the metrics server installed.
Revise: day-23-kubernetes-scaling-helm.md

---

## 13. Common mistakes in DevOps interviews

### Answer mistakes

- **Listing tools with no reasons.** Say why you chose a tool and what you gave up.
- **Skipping the basics.** Many questions test Linux, networking and Git, not only fancy tools.
- **Jumping to a fix.** In troubleshooting, first say how you will look at the problem. Then name the tests.
- **No example.** Give one real case from your labs. Use "I did" for what you did, and "we could" for ideas.
- **Claiming skills you do not have.** Interviewers ask follow-ups. It is fine to say "I have not used this, but I would start by reading ...".
- **Saying "it depends" and stopping.** Say what it depends on, and then pick an option.
- **Confusing terms.** Examples: image and container, CI and CD, Deployment and Service, security group and network ACL.
- **Forgetting security and cost.** Mention least privilege, secrets and cost when you design anything.
- **Ignoring rollback.** Every deploy answer should include how you go back.
- **Blaming people.** In incident stories, talk about process and system fixes.

### Habit mistakes

- **Talking too long.** Give a short answer first. Then offer more detail.
- **Not asking questions.** For design questions, ask about size, users, budget and downtime limits.
- **Memorising commands without understanding.** Know what a command changes before you run it.
- **Running risky commands in the example.** Do not suggest `rm -rf`, `--force` or `terraform destroy` without a safety step.
- **Not practising out loud.** Practise with a friend or a timer. Record yourself.
- **Not preparing your own project.** Use day-30-capstone-interview.md to prepare a clear story.

### Last-week checklist

1. Re-read the Self-Check sections in each day file.
2. Redo three labs from memory: a Dockerfile, a pipeline and a small Terraform stack.
3. Practise the 15 must-know questions out loud.
4. Prepare two stories: a failure you fixed, and a thing you improved.
5. Prepare three questions to ask the interviewer.

This bank is a study aid. It does not predict what any interviewer will ask, and it cannot promise any result.
