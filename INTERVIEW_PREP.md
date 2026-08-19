# KodeKloud — Interview Prep Master Table

Covers all 197 labs across `100DaysofDevops` (100), `100DaysofK8s` (15), `100DaysofTerraform` (35), `100DaysofAzure` (47).

**How to use:** Each row is one interview answer. Read *Concept* → say the *Real-time use* → drop the *Command* → close with the *Architecture* line. That last column is what separates a junior answer from a senior one.

---

## 1. Linux Administration & Troubleshooting — `100DaysofDevops/01–20`

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 01 | User creation | Onboarding app/service accounts; non-login users for daemons | `useradd -m -s /bin/bash user`, `useradd -r -s /sbin/nologin svc` | Identity layer. In real estates this is LDAP/AD/IAM-backed, not local — local users are the fallback for break-glass and service identities |
| 02 | Account expiry | Contractor/vendor access that must auto-revoke; compliance (SOX/ISO) | `chage -E 2026-12-31 user`, `chage -l user` | Time-bound access = least privilege. Same principle as STS tokens in AWS and PIM in Azure AD |
| 03 | SSH hardening | Disable root login, enforce key-only auth on every fleet node | `sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config`, `systemctl restart sshd` | First control in the CIS benchmark. In production this is enforced by Ansible/config-mgmt drift detection, not hand edits |
| 04 | File permissions | Fixing "permission denied" on deploy artifacts and web roots | `chmod 755 dir`, `chmod 644 file`, `chown user:group path` | rwx/octal model underpins container `USER` directives, K8s `securityContext`, and volume mount failures |
| 05 | SELinux | Web server can't read files despite correct chmod — the classic MAC trap | `getenforce`, `setenforce 0`, `semanage fcontext -a -t httpd_sys_content_t`, `restorecon -Rv` | Mandatory Access Control layer *above* DAC. Disabling it is a finding in every audit — relabel instead |
| 06 | Cron scheduling | Log rotation, backups, cleanup jobs on a single host | `crontab -e`, `crontab -l -u user`, `*/5 * * * * /script.sh` | The pre-orchestrator scheduler. Scales up to Jenkins cron triggers → K8s CronJobs → EventBridge/Logic Apps |
| 07 | Passwordless SSH | Prereq for Ansible, Jenkins agents, scp-based deploys | `ssh-keygen -t rsa -b 4096`, `ssh-copy-id user@host` | Trust bootstrap for all agentless automation. Public key on target, private key on controller — never the reverse |
| 08 | Ansible install | Building the control node for config management | `yum install -y ansible-core`, `ansible --version` | Control-node/managed-node topology. Push model, agentless — contrast with Puppet/Chef pull-agents |
| 09 | MariaDB troubleshooting | DB won't start after a disk/permission event — highest-severity incident type | `systemctl status mariadb`, `journalctl -xeu mariadb`, `chown -R mysql:mysql /var/lib/mysql`, `ss -tlnp \| grep 3306` | The four-layer debug ladder: service → logs → filesystem/ownership → port binding. Applies to any stateful service |
| 10 | Bash scripting | Wrapping repeatable ops into idempotent, re-runnable scripts | `#!/bin/bash`, `set -euo pipefail`, loops over host lists | The bridge from manual toil to automation. Every Ansible module started life as someone's bash script |
| 11 | Tomcat install | Deploying a Java app server + WAR on a fixed port | `yum install tomcat`, `systemctl enable --now tomcat`, WAR → `/var/lib/tomcat/webapps/ROOT.war` | App-server tier of the classic 3-tier stack. Containerisation replaces this — same WAR, now in an image |
| 12 | Network troubleshooting | "App unreachable" — is it the process, port, firewall, or route? | `ss -tulnp`, `curl -Iv http://host:port`, `iptables -L -n -v`, `systemctl status` | The universal L3→L7 walk: listener exists? → local firewall? → network path? → app responding? Identical logic for K8s Services and Azure NSGs |
| 13 | iptables | Host-level ingress control, port allow/deny, NAT rules | `iptables -A INPUT -p tcp --dport 80 -j ACCEPT`, `iptables -L -n --line-numbers`, `service iptables save` | Host firewall. Its cloud equivalents are AWS Security Groups and Azure NSGs — and `kube-proxy` writes iptables rules for every K8s Service |
| 14 | Process troubleshooting | Runaway CPU/memory, zombie processes, service flapping | `ps aux --sort=-%cpu`, `top`, `journalctl -u svc -f`, `kill -9 PID` | Feeds capacity planning and alerting thresholds. Same signals Prometheus scrapes as node metrics |
| 15 | Nginx + SSL | Terminating HTTPS with a cert on the edge server | `openssl req -x509 -newkey rsa:2048`, `ssl_certificate` in server block, `nginx -t`, `systemctl reload nginx` | TLS termination point. In cloud this moves to ALB/App Gateway/Ingress — the cert lifecycle problem stays identical |
| 16 | Nginx load balancer | Distributing traffic across app servers | `upstream {}` block, `proxy_pass`, `nginx -t`, `curl` to verify round-robin | L7 reverse proxy. Same role as AWS ALB, Azure App Gateway, K8s Ingress Controller (which *is* Nginx, usually) |
| 17 | PostgreSQL DB creation | Provisioning a database + owner role for a new app | `sudo -u postgres psql`, `CREATE USER ... WITH PASSWORD`, `CREATE DATABASE ... OWNER` | Data tier. Managed equivalents: RDS, Azure SQL — the role/grant model does not change |
| 18 | DB configuration | Enabling remote connections, tuning, binding to a non-loopback address | `bind-address` / `listen_addresses`, `pg_hba.conf`, `systemctl restart` | Default-deny networking on the DB. Modern equivalent: private endpoints + firewall rules instead of open binds |
| 19 | Web app install | Full stack deploy — code, permissions, service, port | `scp` artifact, `chown`, `systemctl enable --now`, `curl` verify | The manual version of a CD pipeline. Every step here becomes a pipeline stage later |
| 20 | Nginx + PHP-FPM | LEMP stack — dynamic content behind a web server | `yum install nginx php-fpm`, `fastcgi_pass 127.0.0.1:9000`, `systemctl enable --now php-fpm nginx` | Web-server/app-runtime separation over a socket. Same split as Nginx→uWSGI, or a sidecar container pair in K8s |

---

## 2. Git & Version Control — `100DaysofDevops/21–34`

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 21 | Git server setup | Self-hosted bare repo as team origin | `git init --bare`, `git clone /opt/repo.git` | Origin is just a repo with no working tree. GitHub/GitLab add auth, PRs, and CI on top of exactly this |
| 22 | Clone | Getting a working copy on a build/deploy server | `git clone /path/repo.git /usr/src/app` | Entry point of every CI pipeline — the first stage of a Jenkins job |
| 23 | Fork | Contributing without write access to upstream | `git clone fork`, `git remote add upstream`, `git push origin branch` | Fork-and-PR model powers open source and enforces review-before-merge in regulated environments |
| 24 | Checkout / branching | Feature isolation; never work on master directly | `git checkout -b feature`, `git branch -a` | Branching strategy = release strategy. GitFlow vs trunk-based determines your deploy cadence |
| 25 | Merge | Integrating a feature into master with history preserved | `git checkout master`, `git merge feature`, `git push origin master` | Merge commits keep true history; the PR merge button is this command server-side |
| 26 | Remotes | Pointing a repo at a new origin after migration | `git remote add/set-url origin`, `git remote -v`, `git ls-remote` | Repo→server mapping. Critical during Bitbucket→GitHub migrations and mirror setups |
| 27 | Revert | Undoing a bad commit on a **shared** branch safely | `git revert <sha>`, `git log --oneline` | Forward-only correction — creates a new commit. The only safe undo for pushed history |
| 28 | Cherry-pick | Hotfix: pull one specific commit into release without merging the branch | `git cherry-pick <sha>` | Selective promotion between environments. Core of hotfix/backport workflows |
| 29 | Pull | Syncing local with remote before work | `git pull origin master`, `git fetch` + `merge` | `pull` = `fetch` + `merge`. Knowing the split is what lets you avoid surprise merge commits |
| 30 | Hard reset | Discarding local commits, resetting to a known-good SHA | `git reset --hard <sha>`, `git push --force` | Destructive and history-rewriting. Acceptable on private branches, forbidden on shared ones — protected-branch rules exist for this |
| 31 | Stash | Parking WIP to switch to an urgent fix | `git stash`, `git stash list`, `git stash pop` | Context-switch tool. Signals you should have been on a separate branch |
| 32 | Rebase | Linear history; replaying your commits on top of updated master | `git fetch`, `git rebase origin/master` | Merge vs rebase is a team-policy decision: readable linear log vs true history. Never rebase shared branches |
| 33 | Merge conflicts | Two people touched the same lines — resolve and continue | `git status`, edit markers, `git add`, `git commit` | Conflict frequency is a proxy for architecture: constant conflicts = files/modules too coarse |
| 34 | Git hooks | Enforcing policy at commit/push time — tag on merge, block secrets | `.git/hooks/post-merge`, `chmod +x`, `git show-ref`, `git tag` | Shift-left enforcement. Client hooks are advisory; server hooks and CI gates are authoritative |

---

## 3. Docker & Containers — `100DaysofDevops/35–47`

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 35 | Docker install | Preparing a container host | `yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin`, `systemctl enable --now docker` | Client → daemon → containerd → runc. Knowing this chain explains K8s dropping dockershim for containerd |
| 36 | Run container | Launching a service with a port map | `docker run -d --name nginx -p 8080:80 nginx:latest`, `docker ps` | Port publishing is a NAT/iptables rule. Same concept as a K8s NodePort |
| 37 | Copy to/from container | Pulling logs out or pushing a config in during an incident | `docker cp file container:/path`, `docker exec -it container bash` | Debug-only escape hatch. Anything copied by hand is lost on restart — containers are immutable, config belongs in mounts/env |
| 38 | Pull & tag images | Promoting images between environments by tag | `docker pull image:tag`, `docker tag src dest`, `docker images` | Tags are mutable pointers; digests are immutable. Production should pin digests |
| 39 | Commit image | Snapshotting a modified container into an image | `docker commit container image:tag` | Anti-pattern for production — undocumented layers. Dockerfile is the reproducible path; know why |
| 40 | Docker service/exec | Verifying processes inside a running container | `docker exec container ps`, `docker ps --format` | One container = one main process (PID 1). Signal handling and restarts hang off this |
| 41 | Build from Dockerfile | Baking app + deps into a versioned artifact | `docker build -t app:v1 .`, `docker run -d -p 80:80 app:v1` | Layer caching drives build speed; ordering instructions correctly is the main optimisation lever |
| 42 | Docker networking | Container-to-container name resolution on a user-defined bridge | `docker network create --driver bridge net1`, `docker network inspect` | Bridge/host/overlay/none. User-defined bridges give embedded DNS — the ancestor of K8s Service discovery |
| 43 | Pull + run image | Standing up a service from a registry image | `docker pull`, `docker run -d -p host:container` | Registry → host pull path. In K8s this is the kubelet's job, with imagePullSecrets |
| 44 | Docker Compose | Declaring a multi-container stack in one file | `docker compose up -d`, `docker compose ps`, `docker compose down` | Declarative multi-container spec — the conceptual predecessor of K8s manifests. Single-host only |
| 45 | Docker troubleshooting | Build/run failures: bad base image, port clash, missing file in context | `docker build` output, `docker logs`, `docker images` | Failure taxonomy: build-time vs runtime vs network. Different logs answer each |
| 46 | App deploy with Compose | Web + DB stack with volumes and dependencies | compose `services`, `volumes`, `depends_on`, `docker compose up -d` | Stateful+stateless together. Volumes are the durability boundary — the container is disposable, the volume is not |
| 47 | Python app container | Containerising a Flask/Python app with a custom Dockerfile | `FROM python`, `COPY`, `RUN pip install -r requirements.txt`, `CMD`, `docker build`/`run` | Language-runtime image pattern. Multi-stage builds shrink final image and cut CVE surface |

---

## 4. Kubernetes — `100DaysofDevops/48–67` + `100DaysofK8s/01–14`

| Ref | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| D48, K01 | Pod | Smallest deployable unit; one-off debug pods | `kubectl run pod --image=nginx`, `kubectl apply -f pod.yaml`, `kubectl get pods -o wide` | Pod = shared network namespace + shared volumes across containers. Bare pods aren't self-healing — that's why Deployments exist |
| D49, K02 | Deployment | Standard way to run stateless apps with self-healing | `kubectl create deployment nginx --image=nginx`, `kubectl get deploy` | Deployment → ReplicaSet → Pod. Each spec change mints a new RS — that's what makes rollback possible |
| K03 | Namespace | Multi-tenant separation: dev/qa/prod, per-team quotas | `kubectl create ns dev`, `kubectl get pods -n dev` | Logical isolation + scope for RBAC, ResourceQuota, and NetworkPolicy. Not a security boundary on its own |
| D50, K04 | Resource requests/limits | Preventing noisy-neighbour and OOMKills | `resources.requests/limits` in spec, `kubectl describe pod` | Requests drive scheduling, limits drive throttling/kill. QoS class (Guaranteed/Burstable/BestEffort) sets eviction order |
| D51, K05 | Rolling update | Zero-downtime image upgrade | `kubectl set image deploy/app c=nginx:1.19`, `kubectl rollout status deploy/app` | maxSurge/maxUnavailable control blast radius. Readiness probes are what make it truly zero-downtime |
| D52, K06 | Rollback | Reverting a bad release in seconds | `kubectl rollout history deploy/app`, `kubectl rollout undo deploy/app --to-revision=N` | Revision history retained on old ReplicaSets. MTTR-driven — roll back first, debug after |
| K07 | ReplicaSet | Maintaining N healthy replicas | `kubectl apply -f rs.yaml`, `kubectl get rs` | Reconciliation loop: desired vs actual. Manage via Deployment, not directly |
| K08, K09 | Job / CronJob | Batch work, backups, scheduled reports | `kubectl get jobs`, `kubectl get cronjobs`, `schedule: "*/5 * * * *"` | Run-to-completion vs run-forever. `backoffLimit`, `completions`, `concurrencyPolicy` are the interview details |
| D53 | Volume mounts | Serving files into a container at a mount path | `volumeMounts` + `volumes`, `kubectl cp`, `kubectl exec` | Mount path shadows existing image content — the #1 cause of "my files vanished" |
| D54 | Shared volumes | Two containers exchanging data in one pod | `emptyDir` volume mounted into both containers | Pod as the sharing boundary. Basis of sidecar patterns |
| D55 | Sidecar | Log shipper / proxy alongside the app container | multi-container `containers:` list, `kubectl logs pod -c sidecar` | Separation of concerns without changing app code. Service meshes (Istio/Envoy) are sidecars at scale |
| D56 | Nginx web server deploy | Standard stateless web tier | `kubectl apply -f deploy.yaml`, `curl` via service | Deployment + Service + Ingress is the canonical web-tier triple |
| D57 | Env variables | Injecting config without rebuilding the image | `env:` / `envFrom:`, `kubectl logs pod` | 12-factor config. ConfigMap for plain, Secret for sensitive |
| D58 | Grafana deploy | Running the observability stack in-cluster | `kubectl apply`, NodePort service, `curl` | Prometheus + Grafana + Alertmanager. Metrics → dashboards → alerts → on-call |
| D59, D64, K11 | Deployment troubleshooting | ImagePullBackOff, CrashLoopBackOff, Pending | `kubectl describe pod`, `kubectl logs --previous`, `kubectl get events --sort-by=.lastTimestamp`, `kubectl edit deploy` | Triage ladder: events → describe → logs → exec. Pending = scheduling/resources; CrashLoop = app; ImagePull = registry/auth |
| D60 | PersistentVolume / PVC | Durable storage that survives pod restarts | `kubectl apply -f pv.yaml pvc.yaml`, `kubectl get pv,pvc` | PV (cluster resource) ↔ PVC (namespaced claim) ↔ StorageClass (dynamic provisioning). Access modes + reclaim policy matter |
| D61 | Init containers | Wait-for-dependency, schema migration, file pre-seed | `initContainers:`, `kubectl logs pod -c init` | Ordered, run-to-completion before app start. Keeps startup ordering out of app code |
| D62 | Secrets | DB passwords, API keys injected safely | `kubectl create secret generic`, `kubectl describe secret`, mount or `envFrom` | Base64 ≠ encrypted. Production needs etcd encryption-at-rest + External Secrets/Vault/Key Vault CSI |
| D63 | Multi-tier app deploy | Web + DB with services, secrets, PVCs together | `kubectl apply -f`, `kubectl -n ns get all`, `curl` NodePort | Full app topology in one namespace. This is the "design a K8s app" whiteboard answer |
| D65 | Redis deploy | Cache/session store in-cluster | `kubectl apply`, `kubectl exec -- redis-cli ping` | Cache tier. ConfigMap-driven config; often a StatefulSet when persistence is needed |
| D66 | MySQL deploy | Stateful DB with PVC + Secret | `kubectl apply`, `kubectl exec -- mysql -u root -p` | Stateful workloads need stable storage + stable identity — the argument for StatefulSet over Deployment |
| D67 | StatefulSet | Ordered, identity-stable pods (DBs, queues, clusters) | `kubectl apply -f sts.yaml`, `kubectl get sts` | Stable network ID (`pod-0`, `pod-1`), ordered rollout, per-pod PVC via volumeClaimTemplates + headless Service |
| K10 | ConfigMap | Non-secret config as files or env | `kubectl apply -f cm.yaml`, `kubectl exec -- cat /config/file` | Config decoupled from image. Note: mounted CMs update live; env-injected ones need a restart |
| K12 | Scale + update together | Replica bump plus image change in one release | `kubectl apply`, `kubectl rollout status`, `kubectl scale` | Manual scale is the baseline; HPA on CPU/custom metrics is the production answer |
| K13 | Service / NodePort | Exposing pods behind a stable virtual IP | `kubectl apply -f svc.yaml`, `kubectl get svc`, `kubectl describe svc` | ClusterIP → NodePort → LoadBalancer → Ingress. Selector-to-label match is the #1 "service has no endpoints" bug |
| K14 | *(empty file)* | Volume mount issues — worth filling in | `kubectl describe pod`, check `volumeMounts` vs `volumes` names | Gap in your notes |

---

## 5. Jenkins & CI/CD — `100DaysofDevops/68–81`

| # | Concept | Real-time use | Key commands / UI path | Architecture (bigger scope) |
|---|---|---|---|---|
| 68 | Jenkins install | Standing up the CI controller | `yum install jenkins java-17`, `systemctl enable --now jenkins`, unlock via `initialAdminPassword` | Controller/agent architecture. Controller schedules, agents execute — never build on the controller |
| 69 | Plugin management | Git, Pipeline, SSH Agent, Credentials Binding plugins | Manage Jenkins → Plugins | Plugin sprawl is the top Jenkins fragility and CVE source. Pin versions, review before upgrade |
| 70 | User management | Per-user accounts instead of shared admin logins | Manage Jenkins → Users | Auditability — every build traceable to a human. Real setups delegate to LDAP/SSO |
| 71 | Package/artifact job | Freestyle job producing a build artifact | Build step: shell; SSH keys to targets | Artifact as the unit of promotion — build once, deploy many |
| 72 | Parameterised builds | One job serving multiple envs/branches | Job → "This project is parameterized" → String/Choice params | Reusable pipelines over copy-pasted jobs. Parameters are the interface |
| 73 | Cron/scheduled jobs | Nightly builds, periodic housekeeping | Build Triggers → Build periodically `H/5 * * * *` | `H` spreads load across the hour — real answer to "why H instead of 0" |
| 74 | DB backup job | Scheduled DB dump pushed to a backup host | shell step + `scp`, key-based SSH | CI server as an ops scheduler. Fine at small scale; backups belong in a dedicated tool at scale |
| 75 | Agent/node setup | Distributing builds to labelled agents | Manage Jenkins → Nodes; SSH launch method + `ssh-keygen` | Horizontal scale + build isolation. Labels route workloads to the right OS/toolchain |
| 76 | Security & authorization | Matrix/role-based permissions per job | Configure Global Security → Matrix-based security | Least privilege in CI. A CI system with prod creds is a top-tier breach target |
| 77 | Pipeline deploy | Declarative pipeline: checkout → build → deploy | `Jenkinsfile`, `pipeline { stages { } }`, `git` + `sshagent` | Pipeline-as-code lives with the app, is versioned and reviewed. The core CI/CD interview answer |
| 78 | Conditional pipeline | Branch-based routing (develop → dev, master → prod) | `when { branch 'master' }` | Environment promotion logic in code. Basis of trunk-based delivery |
| 79 | Push-triggered build | Auto-deploy on `git push` | Poll SCM `* * * * *` / webhook | Poll is pull-based (lag, load); webhook is push-based (instant). Know the trade-off |
| 80 | Chained builds | Job A deploys, then triggers job B to restart httpd | Post-build → Build other projects; `credentials-binding` | Job orchestration DAG. Modern equivalent: pipeline stages or Argo Workflows |
| 81 | Multi-stage pipeline | Deploy + test in one declarative pipeline | `stages { stage('Deploy') stage('Test') }`, `git push` from pipeline | Quality gates between stages. Failed test → no promotion, which is what CD actually means |

---

## 6. Ansible — `100DaysofDevops/82–93`

| # | Concept | Real-time use | Key commands / modules | Architecture (bigger scope) |
|---|---|---|---|---|
| 82 | Inventory | Defining managed hosts and connection vars | `inventory` file, `ansible-inventory --list`, `ansible all -m ping` | Static vs dynamic inventory. Cloud estates use dynamic plugins keyed off tags — inventory becomes generated, not written |
| 83 | Troubleshooting / ansible.cfg | Fixing host-key, privilege, and inventory-path failures | `ansible.cfg` with `host_key_checking=False`, `inventory=`, `remote_user=` | Config precedence: `ANSIBLE_CFG` env → cwd → `~/.ansible.cfg` → `/etc/ansible`. Classic gotcha question |
| 84 | Copy module | Pushing config/content files to many hosts | `ansible.builtin.copy: src= dest= owner= mode=` | Idempotency: copies only on checksum change. Idempotence is Ansible's whole value proposition |
| 85 | File module | Creating dirs/files with exact ownership and perms | `ansible.builtin.file: path= state=touch/directory owner= mode=` | Declarative desired state — you describe the end state, not the steps |
| 86 | Ping / connectivity | Verifying SSH trust before any playbook run | `ssh-keygen`, `ssh-copy-id`, `ansible all -m ping` | Agentless over SSH. `ping` here is an Ansible module round-trip, not ICMP — a favourite interview trick |
| 87 | Package module | Installing packages fleet-wide, distro-agnostically | `ansible.builtin.yum/dnf/package: name= state=present` | `package` abstracts the distro's package manager. Abstraction over portability |
| 88 | blockinfile | Managing a marked block inside an existing config file | `ansible.builtin.blockinfile: path= block= marker=` | Safe partial-file edits — markers make it re-runnable. Full-file `template` is cleaner when you own the file |
| 89 | Service module | Ensuring services are started and enabled | `ansible.builtin.service: name= state=started enabled=yes` | Convergence to desired state, plus handlers for restart-on-change |
| 90 | ACLs | Fine-grained access beyond owner/group/other | `ansible.posix.acl`, `getfacl`, `ansible-galaxy collection install` | POSIX ACLs when standard perms can't express the requirement. Collections = modular content distribution |
| 91 | lineinfile | Changing a single directive in a config file | `ansible.builtin.lineinfile: path= regexp= line=` | Regex-based idempotent edit. Regex accuracy is the failure mode — anchor your patterns |
| 92 | Jinja2 templates | Per-host config rendered from variables | `ansible.builtin.template: src=x.j2 dest=`, `{{ var }}`, `{% if %}` | One template, N hosts. Same idea as Helm charts for K8s and Terraform interpolation |
| 93 | Conditionals | Branching on OS family, facts, or variables | `when: ansible_facts['os_family'] == 'RedHat'`, `register` + `when` | Facts gathering drives conditional logic. Playbooks become portable across heterogeneous fleets |

---

## 7. Terraform / IaC — `100DaysofDevops/94–100` + `100DaysofTerraform/01–35`

### 7a. Core workflow & language

| Ref | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| all | Core workflow | Every change goes init → validate → plan → apply | `terraform init`, `terraform validate`, `terraform plan`, `terraform apply -auto-approve`, `terraform destroy` | Plan is the review artifact and the safety gate — "plan in PR, apply on merge" is the standard pipeline |
| all | State | Source of truth mapping config → real resources | `terraform state list`, `terraform state show <addr>`, `terraform state rm` | Local state doesn't scale. Remote backend (S3+DynamoDB lock / Azure Storage) gives sharing, locking, versioning. State holds secrets in plaintext — encrypt it |
| T25, T25.1 | Lifecycle meta-arguments | Protecting prod resources; avoiding downtime on replace | `lifecycle { create_before_destroy, prevent_destroy, ignore_changes }` | Governs the destroy/create ordering that Terraform would otherwise pick. `ignore_changes` reconciles Terraform with out-of-band drift |
| T25 | Resource replacement | Why a type change forces destroy+create | `terraform plan` shows `-/+ must be replaced` | ForceNew attributes in the provider schema. Knowing which attributes force replacement prevents accidental prod outages |
| D98–100 | Outputs & validation | Surfacing IDs/IPs to operators and downstream modules | `terraform output`, `terraform validate` | Outputs are a module's public API — how modules compose via `remote_state`/inputs |
| T35 | *(empty file)* | Variables & tfvars — worth filling in | `variable {}`, `-var-file=prod.tfvars`, `TF_VAR_x` | Gap in your notes. Precedence order (CLI > env > tfvars > default) is a frequent question |
| T30–33 | Targeted destroy | Removing one resource but keeping the code | `terraform destroy -target=aws_instance.ec2 -auto-approve` | `-target` is a break-glass tool — it partially applies the graph. Correct long-term answer is `count = 0` or removing the block |
| T29 | Provisioners / null_resource | Running CLI steps Terraform has no resource for (S3 sync then delete) | `resource "null_resource"` + `local-exec` | Last resort — provisioners break the declarative model and aren't tracked in state. Prefer native resources or a pipeline step |

### 7b. Resource catalogue

| Ref | Concept | Real-time use | Key resources | Architecture (bigger scope) |
|---|---|---|---|---|
| T01 | Key pair | SSH access to EC2, key generated by Terraform | `tls_private_key`, `aws_key_pair`, `local_file` | Private key lands in state **and** on disk — real answer is Secrets Manager or a pre-created key |
| T02, D95 | Security group | Instance-level firewall with ingress/egress rules | `aws_security_group`, `data "aws_vpc"` | Stateful firewall at the ENI. SG-referencing-SG is the pattern for tier-to-tier rules (web→app→db) |
| T03, T04, D94 | VPC & CIDR | Network foundation and address planning | `aws_vpc { cidr_block }` | CIDR sizing is one-way — over-provision. Non-overlapping ranges are mandatory for future peering/VPN |
| T05 | *(empty)* IPv6 VPC | — | `assign_generated_ipv6_cidr_block = true` | Gap in your notes |
| T06 | *(empty)* Elastic IP | — | `aws_eip` | Gap in your notes (covered in T26) |
| T07, D96 | EC2 instance | Compute with AMI, type, key, and SG attached | `aws_instance`, `data "aws_security_group"` | Compute tier. AMI + user_data = immutable infrastructure; instead of patching, rebuild and replace |
| T08 | AMI from instance | Golden image capture for autoscaling | `aws_ami_from_instance`, `data "aws_instance"` | Baking (Packer/AMI) vs bootstrapping (user_data/Ansible). Baking gives faster, deterministic scale-out |
| T09 | EBS volume | Block storage for a DB or app data disk | `aws_ebs_volume { size, type, availability_zone }` | AZ-bound — volume and instance must share an AZ. gp3 vs io2 is a cost/IOPS decision |
| T10 | EBS snapshot | Point-in-time backup before a risky change | `aws_ebs_snapshot`, `data "aws_ebs_volume"` | Incremental, S3-backed, cross-region copyable. Backbone of backup/DR and RPO planning |
| T11, D100 | CloudWatch alarm | Alerting on CPU/status-check thresholds | `aws_cloudwatch_metric_alarm` | Metric → threshold → SNS → action. Alarms should drive auto-remediation, not just email |
| T12 | Public S3 bucket | Static site or public asset hosting | `aws_s3_bucket`, `_acl`, `_ownership_controls`, `_public_access_block`, `_policy` | Public buckets are the classic breach headline. Post-2023 AWS blocks public access by default — you must explicitly unwind four controls |
| T15, T28 | Private S3 + versioning | Default-secure bucket with object history | `aws_s3_bucket_versioning`, `_public_access_block` | Versioning protects against overwrite/delete and is a prerequisite for replication and object lock |
| T34 | S3 object upload | Shipping a file into a bucket via IaC | `aws_s3_object { bucket, key, source, etag }` | IaC managing data, not just infra — fine for small config/static assets, wrong for app data |
| T13 | IAM user | Human/service identity with access keys | `aws_iam_user`, `_access_key`, `_login_profile`, `_policy_attachment` | Long-lived keys are a liability. Roles + STS/OIDC federation is the modern answer |
| T14 | IAM group | Permissions assigned by role, not per person | `aws_iam_group`, `_membership`, `_policy_attachment` | Group-based RBAC scales; per-user policies do not |
| T16, D97 | IAM policy | Least-privilege JSON permission document | `aws_iam_policy { policy = jsonencode(...) }` | Effect/Action/Resource/Condition. Deny beats Allow; explicit Deny is unoverridable |
| T27, D99 | Policy attachment | Binding a policy to user/group/role | `aws_iam_user_policy_attachment`, `_role_policy_attachment` | Separating policy definition from binding lets one policy serve many principals |
| T32 | IAM role | Assumable identity for services and cross-account | `aws_iam_role` + `assume_role_policy`, `_role_policy_attachment` | Trust policy (who can assume) vs permission policy (what they can do). The distinction is a standard interview question |
| T17 | DynamoDB | NoSQL key-value store; also the TF state-lock table | `aws_dynamodb_table { hash_key, billing_mode }` | Partition key design determines throughput distribution. On-demand vs provisioned is a cost model choice |
| T18 | Kinesis | Real-time streaming ingest | `aws_kinesis_stream { shard_count, retention_period }` | Shards = throughput units. Streaming pipeline: producer → stream → consumer → sink |
| T19 | SNS | Pub/sub fan-out for alerts and events | `aws_sns_topic`, `_subscription`, `_topic_policy` | Decouples producers from consumers. SNS (push, fan-out) + SQS (pull, buffered) is the durable pattern |
| T20 | SSM Parameter Store | Config and secrets without hardcoding | `aws_ssm_parameter { type = "SecureString" }`, `data "aws_ssm_parameter"` | Externalised config with KMS encryption and IAM-scoped reads. Cheaper alternative to Secrets Manager |
| T24 | Secrets Manager | Credentials with rotation and versioning | `aws_secretsmanager_secret`, `_secret_version` | Native rotation + fine-grained IAM. Secrets still land in TF state — pull at runtime instead of at plan time |
| T21 | CloudWatch Logs | Centralised log aggregation with retention | `aws_cloudwatch_log_group`, `_log_stream`, `_metric_filter`, `_subscription_filter` | Logs → metric filters → alarms; subscription filters ship to Kinesis/Lambda/OpenSearch. The observability pipeline |
| T22 | CloudFormation via Terraform | Wrapping a CFN stack inside Terraform | `aws_cloudformation_stack { template_body }` | Migration/coexistence pattern for estates that still run CFN. Two state systems — know which owns what |
| T23 | OpenSearch | Search and log analytics backend | `aws_opensearch_domain { cluster_config, ebs_options }` | ELK/OpenSearch stack. Sizing (data vs master nodes) and access policy are the operational levers |
| T26 | Elastic IP association | Stable public IP attached to an instance | `aws_eip`, `aws_eip_association` | Static IPs are an anti-pattern for scale — DNS + load balancer is the elastic answer. EIPs remain for allow-listing |
| D98 | Private VPC infra | Private subnet, no public IP, controlled egress | `aws_vpc`, subnets, route tables, `aws_instance` | Public/private subnet split = the standard secure topology. Private egress via NAT Gateway; access via bastion/SSM |

---

## 8. Azure — `100DaysofAzure/01–47`

### 8a. Compute & VM lifecycle

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 01 | VM creation | Provisioning the base compute unit | `az vm create -g RG -n vm --image Ubuntu2204 --size Standard_B1s`, `az vm show -d` | A "VM" is really 5 resources: VM + NIC + Public IP + NSG + Disk. Explaining that is a strong interview answer |
| 02 | VM resize | Right-sizing for cost or performance | `az vm list-vm-resize-options`, `az vm resize --size Standard_B2s` | Vertical scaling requires a reboot and is capped by the host SKU family. Horizontal (VMSS) is the cloud-native answer |
| 03 | Attach managed disk | Adding data capacity to a running VM | `az vm disk attach --vm-name vm --name disk`, `az disk show` | Managed disks decouple storage lifecycle from VM lifecycle. OS vs data vs temp disk — temp is ephemeral |
| 05 | Attach NIC | Multi-NIC VMs for segmented traffic | `az vm deallocate`, `az vm nic add`, `az vm start` | NIC changes need deallocation. Multi-NIC = management/data plane separation, common in NVAs |
| 08 | Tagging | Cost allocation, ownership, environment tracking | `az resource tag --tags env=prod owner=team`, `az resource list -g RG` | Tags drive chargeback, Azure Policy enforcement, and automated cleanup. Untagged resources are unowned spend |
| 09, 10 | SSH keys | Key-based VM auth and key distribution | `ssh-keygen -t rsa -b 4096`, `az vm create --ssh-key-values`, `az vm show -d --query publicIps` | Azure stores only the public key. Lost private key = lost access unless you use VM Access Extension or Bastion |
| 11 | Managed disk create | Pre-provisioning storage for later attach | `az disk create -g RG -n disk --size-gb 30 --sku Standard_LRS` | SKU = redundancy + performance tier (Standard_LRS → Premium_SSD → Ultra). LRS/ZRS/GRS is the durability axis |
| 18, 21 | VM + SSH provisioning | Full VM build with key auth and open ports | `az vm create --generate-ssh-keys`, `az vm open-port --port 80`, `az vm delete`, `az disk delete` | `az vm delete` leaves the disk, NIC, and IP behind — orphaned-resource cost is a real finding |
| 19, 20 | VM + Nginx (cloud-init) | Bootstrapping software at first boot | `az vm create --custom-data cloud-init.txt`, `az vm run-command invoke --command-id RunShellScript` | cloud-init = declarative bootstrap; `run-command` = imperative post-hoc fix. Prefer bootstrap for reproducibility |
| 22 | Disk expand + attach | Growing the OS disk and adding a data disk | `az vm deallocate`, `az disk update --size-gb`, `az vm disk attach`, then in-guest `growpart` + `resize2fs` | Two-layer resize: cloud disk **and** guest filesystem. Forgetting the guest step is the classic mistake |
| 25, 31 | VM troubleshooting | Web app unreachable on a public VM | `az vm run-command invoke`, `az network nsg rule list`, `az network nic show`, `az network route-table route list` | Ordered path check: NSG (subnet+NIC) → route table → NIC/IP association → guest service. Both NSG layers must allow |
| 34 | VM ↔ MySQL connectivity | App VM reaching a database VM privately | `az vm create`, `az network nsg rule create --destination-port-ranges 3306`, `az vm list-ip-addresses` | Tier-to-tier rules scoped to source subnet, never `0.0.0.0/0`. Private-only DB traffic |

### 8b. Networking

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 04 | Public IP → NIC | Making a VM internet-reachable | `az network nic ip-config update --public-ip-address`, `az network public-ip show` | Public IP binds to an ipconfig on a NIC, not to the VM. Static vs dynamic matters for DNS and allow-lists |
| 06 | Public IP allocation | Reserving a static address | `az network public-ip create --allocation-method Static --sku Standard` | Standard SKU is zone-redundant and secure-by-default (deny inbound unless an NSG allows). Basic is retired |
| 07 | VNet & subnet | Network segmentation per tier | `az network vnet create --address-prefix 10.0.0.0/16`, `az network vnet subnet create --address-prefix 10.0.1.0/24` | Address planning must anticipate peering and on-prem VPN. Azure reserves 5 IPs per subnet |
| 12 | NSG rules | Stateful allow/deny at subnet and NIC level | `az network nsg create`, `az network nsg rule create --priority 100 --destination-port-ranges 22`, `az network nic update --network-security-group` | Lowest priority number wins; default rules allow intra-VNet and deny internet inbound. Both NSG layers evaluate — the top source of "why is it still blocked" |
| 23, 24 | Public/private VNet deploy | Bastion-plus-private-workload topology | `az network vnet create`, `az vm create --public-ip-address ""`, `az network nsg rule` | Hub-and-spoke. Private subnets egress via NAT Gateway or firewall; access via Bastion/Just-In-Time (file 24 is empty) |
| 30 | Load balancer | Distributing traffic across VM backends | `az network lb create`, `az network lb probe create`, `az network lb rule create` | L4 (Azure LB) vs L7 (App Gateway). Health probe defines what "healthy" means — probe misconfiguration = silent outage |
| 32 | VNet peering | Connecting two VNets privately | `az network vnet peering create --remote-vnet --allow-vnet-access` | Peering is **non-transitive** and must be created on both sides. Overlapping CIDRs block it permanently |
| 40, 47 | Application Gateway | L7 routing, WAF, path/host-based rules | `az network application-gateway show`, `... show-backend-health`, `az deployment group create` | L7 with TLS termination, WAF, and cookie affinity. Needs a dedicated subnet. Backend health is the first thing to check on a 502 |

### 8c. Storage

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 13, 14 | Storage account + container | Object storage for artifacts, backups, media | `az storage account create --sku Standard_LRS`, `az storage container create --public-access off/blob`, `az storage account keys list` | Account → container → blob. Replication (LRS/ZRS/GRS/RA-GRS) is a DR decision; access tier (Hot/Cool/Archive) is a cost decision |
| 15, 29 | Blob upload / copy | Moving artifacts and backups between containers | `az storage blob upload`, `az storage blob copy start`, `az storage blob list`, `az storage blob exists` | Server-side copy is async and avoids egress through the client. Always verify with `exists`/`show` before deleting the source |
| 16 | Change container access | Locking down an accidentally public container | `az storage container set-permission --public-access off` | Public blob access is a top audit finding. Prefer SAS tokens or Entra ID RBAC over anonymous access |
| 33 | Lifecycle management | Auto-tiering and expiring old blobs | `az storage account management-policy create --policy @policy.json` | Policy-driven cost control: Hot → Cool → Archive → delete. The single biggest storage-cost lever |
| 35 | VM → Blob upload | App on a VM writing backups to object storage | `az storage blob upload`, account key or Managed Identity | Managed Identity beats account keys — no secret to rotate or leak. Key-based access is the legacy path |
| 36 | Static website hosting | Hosting a SPA/static site directly from storage | `az storage blob service-properties update --static-website --index-document index.html` | Serverless hosting: no VM, no patching. Front with CDN/Front Door for TLS, custom domain, and caching |
| 38 | Table storage | NoSQL key-attribute store for a to-do style app | `az storage table create`, `az storage entity insert/query/show` | PartitionKey + RowKey = the entire index. Cheap and fast for known-key access; Cosmos DB is the richer successor |
| 39 | Blob backup & cleanup | Download-then-delete retention workflow | `az storage blob download-batch`, `az storage container delete`, `az storage container exists` | Verify-before-delete is the rule. Soft delete and versioning are the real safety nets |

### 8d. Platform services

| # | Concept | Real-time use | Key commands | Architecture (bigger scope) |
|---|---|---|---|---|
| 17 | ARM template deploy | Declarative, repeatable infra deployment | `az deployment group create --template-file azuredeploy.json --parameters @params.json` | Azure's native IaC. Idempotent, supports what-if. Bicep is the readable DSL; Terraform is the multi-cloud alternative |
| 26, 45 | Azure Container Registry | Private image registry feeding VMs/AKS | `az acr create --sku Basic`, `az acr login`, `az acr build`, `az acr repository list`, `az acr repository show-tags` | Registry sits between CI and runtime. Managed Identity/AcrPull for auth, not admin credentials. Enable content trust + vulnerability scanning |
| 27, 44 | Azure SQL | Managed relational DB with firewall rules and export | `az sql server create`, `az sql db create`, `az sql server firewall-rule create`, `az sql db export --storage-uri` | PaaS DB: HA, patching, and backup managed for you. `db export` to a bacpac in blob is the portable backup/migration path |
| 28 | App Service / Web App | PaaS app hosting with no VM management | `az appservice plan create`, `az webapp create --runtime`, `az webapp config appsettings set` | Plan = compute, webapp = workload; many apps per plan. App settings inject config as env vars — the 12-factor path |
| 37 | Key Vault | Central secrets/keys with encrypt-decrypt operations | `az keyvault create`, `az keyvault key create --kty RSA`, `az keyvault key encrypt/decrypt`, `az keyvault set-policy` | Keys never leave the vault — you send data to it, not the key to you (HSM-backed). Access via RBAC/policy + Managed Identity |
| 41, 43 | Event Hubs | High-throughput event/log ingestion and capture to blob | `az eventhubs namespace create`, `az eventhubs eventhub create --partition-count`, `az eventhubs namespace authorization-rule keys list`, `az monitor metrics list` | Kafka-equivalent ingest. Partitions = parallelism; consumer groups = independent readers. Capture writes straight to blob for cheap replay |
| 42 | AKS | Managed Kubernetes control plane | `az aks create --node-count --enable-managed-identity`, `az aks get-credentials`, `az aks update`, `az aks disable-addons` | Managed control plane, you own the node pools. Integrates ACR, Key Vault CSI, Entra RBAC, and Azure CNI vs kubenet networking |
| 46 | Storage + Nginx static content | VM serving content pulled from blob storage | `az storage account create/update`, `az storage blob exists`, `az network nsg rule create` | Content-origin separation: storage is the source of truth, the VM is a cache/serving tier. Front Door/CDN replaces this at scale |

---

## 9. Cross-cutting architecture story (the "bigger scope" answer)

```
Developer → Git (branch/PR)
   → Jenkins (build, test, artifact/image)
       → Registry (ACR / Docker Hub)
           → Terraform / ARM  (provision infra: VNet, VM/AKS, DB, storage, IAM)
               → Ansible      (configure OS + app on VMs)
               → Kubernetes   (schedule containers: Deployment → Service → Ingress)
                   → Monitoring (CloudWatch / Prometheus + Grafana → alerts)
                       → Feedback loop back to Git
```

**The four boundaries interviewers probe:**

| Boundary | Question they ask | Your answer |
|---|---|---|
| Provisioning vs configuration | "Terraform or Ansible?" | Terraform provisions immutable infra (declarative, state-tracked). Ansible configures inside it (idempotent, agentless). They compose; they don't compete |
| Mutable vs immutable | "How do you patch 200 servers?" | Ansible for mutable fleets; rebuild from a golden image/container for immutable. Immutable removes drift entirely |
| Push vs pull | "Webhook or poll?" | Push (webhook, Ansible) = instant, needs reachability. Pull (Poll SCM, GitOps/Argo) = resilient, eventually consistent |
| Stateless vs stateful | "Deployment or StatefulSet?" | Stateless → Deployment + HPA, any pod is disposable. Stateful → StatefulSet + PVC + headless Service for stable identity and ordered rollout |

**Secrets, across every layer:** never in Git → Jenkins Credentials → Ansible Vault → K8s Secret (+ etcd encryption) → SSM/Secrets Manager/Key Vault → Managed Identity (best: no secret exists at all).

**Troubleshooting ladder that works everywhere:**
`Is the process running?` → `Is it listening on the port?` → `Does the firewall allow it?` (host iptables / SG / NSG) → `Is the route/DNS correct?` → `Is the app returning errors?` (logs)

---

## 10. Gaps in your notes (worth filling before interviews)

| File | Missing topic | Why it matters |
|---|---|---|
| `100DaysofK8s/14.K8s_volume_mountissues.md` | Volume mount troubleshooting | Mount-path shadowing and volume-name mismatch are common interview scenarios |
| `100DaysofTerraform/05.Terrafrom_vpc_ipv6.md` | IPv6-enabled VPC | `assign_generated_ipv6_cidr_block`, dual-stack subnets |
| `100DaysofTerraform/06.Terrafrom_elastic_ip.md` | Elastic IP creation | Partly covered by `26.Terraform_elsaticip.md` |
| `100DaysofTerraform/35.Terraform_variablesetup.md` | Variables, tfvars, precedence | Asked in nearly every Terraform interview |
| `100DaysofAzure/24.vm_deploy_private_vn.md` | Private VNet VM deploy | The secure-topology counterpart to file 23 |
