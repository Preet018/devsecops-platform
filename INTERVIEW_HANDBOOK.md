# DevSecOps Platform — Interview Preparation & Revision Handbook

> Built from the **actual repository** (source code and configs are the source of truth), not from the notes PDF.
> Where the notes and the code disagree, this handbook follows the code and says so.

**How to use this file**
1. Read **Part 0** first. It tells you what you can and cannot claim.
2. Read Parts 1–9 in order for depth (full format: *Question → Interview Answer → Deep Dive → Project Connection → Follow-ups → Quick Revision*).
3. Use **Part 10** (cheat sheet) for the last 30–45 minutes before an interview.
4. Do **Part 11** (mock interview) out loud, with someone or a recorder.

**Legend:** ✅ implemented in code · 🟡 partially implemented · 📝 proposed / not in code · ⚠️ weakness an interviewer may find

---

# PART 0 — The truth about your implementation

## 0.1 What the project is
A **multi-tenant DevSecOps deployment platform** (mini internal developer platform). A user logs into a React dashboard, creates a project, and the Express backend provisions Kubernetes resources for it, triggers a Jenkins pipeline (Vault → build → Docker → Trivy → deploy), and the dashboard shows status, live URL, security report links, CPU/memory (Prometheus) and logs.

## 0.2 Repository map
| Path | Contents |
|---|---|
| `frontend/` | React + Vite dashboard; `Dockerfile` (multi-stage: node build → nginx) |
| `backend/server.js` | The whole API (457 lines): auth, projects, webhook, monitoring, logs |
| `backend/templates/deployment-template.yaml` | Per-project Deployment + NodePort Service template |
| `backend/models/` | `User.js`, `Project.js` — **not imported by `server.js`** |
| `Jenkinsfile` | Declarative pipeline (6 stages + `post`) |
| `sonar-project.properties` | SonarCloud project config |
| `infrastructure/docker-compose.yml` | Jenkins, Vault (dev mode), MongoDB |
| `infrastructure/jenkins/Dockerfile` | Jenkins LTS + docker.io + kubectl + trivy |
| `infrastructure/deploy-monitoring.sh`, `values-monitoring.yaml` | Helm install of kube-prometheus-stack |
| `monitoring/` | Compose stack: Prometheus (self-scrape only), Loki, Promtail, Grafana + their configs |
| `k8s/` | `backend.yaml`, `frontend.yaml`, `hpa.yaml`, `promtail.yaml` |
| `ansible/` | Playbook + role `k8s-deploy` |

## 0.3 Implementation truth table

### Platform and CI/CD
| Feature | Status | Evidence / what is really true |
|---|---|---|
| JWT auth + bcrypt | ✅ | `jwt.sign(..., {expiresIn:'8h'})`, `bcrypt.hash(pw, 10)` |
| Per-user project isolation (data) | ✅ | `Project.find({owner: req.user.id})`; delete checks owner |
| Per-project K8s Deployment + Service | ✅ | Template → `kubectl apply -n devsecops-prod` |
| Live URL via NodePort | ✅ | `kubectl get svc -o json` → nodePort + `minikube ip` |
| Namespace-per-tenant | 📝 | All projects in `devsecops-prod`. `kubernetesNamespace` exists only in unused `models/User.js` |
| What the project pods run | 🟡 | Template image is **`nginxdemos/hello`**, not the user's code |
| Pipeline builds user's repo | 📝 | `checkout scm` = the platform's own repo; user's `repo` URL is stored but never used by Jenkins |
| Jenkins triggered by backend | ✅ | `buildWithParameters` with `PROJECT_ID` |
| GitHub push → Jenkins webhook | 📝 | Not configured in code |
| Jenkins → backend status webhook | ✅ | `post {}` → `host.docker.internal:5000/api/webhook/jenkins` |
| Docker build + tag (build no. + latest) + push | ✅ | `Containerize` stage |
| Deploy stage | ✅ | `kubectl apply` of backend/frontend/hpa YAML |
| HPA | ✅ | CPU-only, 2–5 replicas, 70% target, for platform backend/frontend (not user projects) |
| Ansible | 🟡 | Playbook/role exist; Jenkins doesn't use them; `__DOCKER_USER__` placeholder no longer in YAML |

### Security
| Feature | Status | Evidence / what is really true |
|---|---|---|
| Vault stores SonarCloud token | ✅ | `vault kv put secret/sonarcloud token=…` (per your notes) |
| Jenkins fetches it via `withVault` | ✅ | KV v2, path `secret/sonarcloud`, key `token` → `SONAR_TOKEN` |
| Vault persistent / production mode | 📝 | `vault server -dev` (in-memory, auto-unsealed, root token in compose) |
| Vault least-privilege auth (AppRole) | 📝 | Jenkins uses the **root token** (`vault-root-token` credential) |
| **SonarCloud SAST scan** | 🟡 | `sonar-project.properties` exists, but the Jenkinsfile only **echoes** the token. **No `sonar-scanner` call.** ⚠️ Also prints the token to the log |
| Quality gate enforcement | 📝 | None |
| Per-project Sonar keys | 📝 | One hardcoded key `chahak02_devsecops-platform` |
| **Trivy image scan** | 🟡 | Runs on the **backend image only**, **after push**, **never fails the build**, HTML report to `/reports` |
| Trivy `--exit-code` / severity gate | 📝 | Not present |
| Real scan numbers on dashboard | 📝 | Webhook payload has **hardcoded** `bugs:2, vulnerabilities:0, codeSmells:15, critical:0, high:1, medium:4` and a fixed `liveUrl` |
| SBOM, image signing, DAST, secret scanning | 📝 | None |
| Resource requests/limits | ✅ | In template and platform YAML |
| Probes, securityContext, NetworkPolicy, RBAC, Pod Security | 📝 | None in any manifest |
| helmet, CORS | 🟡 | `helmet()` ✅; `cors()` with **no origin restriction** ⚠️ |
| Rate limiting / brute-force protection | 📝 | None on `/api/auth/login` |
| Webhook authentication | 📝 | **None** — anyone reaching :5000 can set any project "Deployed" |

### Monitoring
| Feature | Status | Evidence / what is really true |
|---|---|---|
| kube-prometheus-stack (Prometheus, Grafana, node-exporter, kube-state-metrics, Alertmanager) | ✅ | `deploy-monitoring.sh`, `monitoring` namespace |
| Backend queries Prometheus (PromQL) | ✅ | `container_memory_usage_bytes`, `rate(container_cpu_usage_seconds_total[1m])` |
| Grafana embedded in dashboard (iframe, `var-pod`) | ✅ | `Monitoring.jsx` |
| Grafana security | ⚠️ | Anonymous **Admin**, login form disabled (`values-monitoring.yaml`); standalone Grafana `admin/admin` |
| Standalone `monitoring/prometheus.yml` | 🟡 | Scrapes **only itself** (your own comment says it's wrong) |
| Loki + Promtail (Compose) | ✅ | Promtail reads `/var/lib/docker/containers/*/*.log`, label `job=varlogs`, pushes to `loki:3100` |
| Loki config | 🟡 | `auth_enabled:false`, **filesystem under `/tmp/loki`**, in-memory ring → logs lost if container is recreated without a volume |
| Promtail DaemonSet in K8s (`k8s/promtail.yaml`) | 🟡 | Loki URL is a **placeholder** (`<YOUR_GRAFANA_LOKI_URL>`); `serviceAccountName: default` with no RBAC defined → pod discovery would be forbidden; hostPath mounts |
| Dashboard Logs page | ✅ | `kubectl logs --tail=50` via backend — **not Loki** |
| Custom alert rules / Alertmanager receivers | 📝 | None in repo (chart defaults only) |
| Tracing | 📝 | None |
| Monitoring/logs API authenticated | ⚠️ | **No** `authenticateToken`; `projectName` is interpolated into a shell command |

## 0.4 Important code facts to know cold
- `backend/.env` contains **only `PORT`**. So `JWT_SECRET` and `MONGO_URI` are **not set** → the **hardcoded fallbacks are the real values in use** (`'devsecops-super-secret-key'`, `mongodb://localhost:27017/devsecops`). `.env` is not git-ignored and there is no root `.gitignore`.
- Backend runs **on the host** (`node server.js`), Prometheus/Grafana are reached via **kubectl port-forward** to `localhost`.
- `/api/stats` adds fixed numbers (`+145`, `+2`, `securityAlerts: 3`, `clusterHealth: '98%'`).
- Port allocation (`findOne().sort('-port')`) is a race condition; the actual access port is the Kubernetes nodePort.
- Jenkins container: `privileged: true`, `user: root`, `docker.sock` mounted, **CSRF protection disabled**, `runSetupWizard=false` (verify that authentication/authorization is actually enabled in your Jenkins — the compose file alone doesn't prove it).
- Reports are served from a **hardcoded absolute path** `/Users/veeralagarwal/...` in `server.js`.
- The Trivy report is named by `PROJECT_ID`, so each run overwrites the previous one for that project.

## 0.5 Your individual contribution — fill this in honestly
The files don't record who wrote what. Complete this table **before** any interview; interviewers ask "what exactly did *you* do?".

| Area | What I personally did | Evidence I can show |
|---|---|---|
| Vault integration | _fill in_ | `withVault` block, compose service |
| SonarCloud | _fill in_ | `sonar-project.properties` |
| Trivy | _fill in_ | Trivy stage, Jenkins Dockerfile |
| Prometheus/Grafana | _fill in_ | `deploy-monitoring.sh`, `values-monitoring.yaml`, `Monitoring.jsx`, `/api/monitoring` |
| Loki/Promtail | _fill in_ | `monitoring/`, `k8s/promtail.yaml` |
| Other | _fill in_ | |

**Never claim** something in the ✅ column you can't explain line by line, and never claim 📝 items as done.

## 0.6 How to phrase gaps (3-part formula)
> **What it does now → why (a conscious trade-off) → what I'd do next.**
> *"The Trivy stage generates an HTML report but doesn't block the build; I focused first on getting the report onto the dashboard. Next I'd scan before the push with `--exit-code 1 --severity CRITICAL,HIGH`."*

---

# PART 1 — DevSecOps fundamentals

> Compact format here (Answer + Project link). Full format begins in Part 2.

### Q1. What is DevOps?
**Answer:** A culture and set of practices that unites development and operations so software is built, tested, released and operated through **automation and fast feedback**, instead of throwing code over a wall. Core ideas: CI/CD, infrastructure as code, monitoring, collaboration.
**Project:** Git → Jenkins → Docker → Kubernetes, with Prometheus/Grafana feedback.

### Q2. What is DevSecOps? DevOps vs DevSecOps?
**Answer:** DevSecOps embeds **security into every stage** of the DevOps lifecycle and makes it everyone's job, automated in the pipeline, rather than a manual review at the end.
| | DevOps | DevSecOps |
|---|---|---|
| Goal | Speed + reliability | Speed + reliability + **security** |
| Security | Late, separate team | Continuous, automated, shared |
| Pipeline | build → test → deploy | + SAST, SCA, secrets, image scan, policy gates |
| Failure | Tests fail | **Vulnerabilities fail the build** (when gated) |
**Project:** Vault (secrets), SonarCloud (SAST, partial), Trivy (image scan, partial). ⚠️ Gates not enforced yet.

### Q3. Why integrate security into CI/CD?
Cost of a fix rises sharply the later a bug is found; manual reviews don't scale with deploy frequency; automated checks are consistent; it creates an audit trail.

### Q4. What is Shift-Left Security?
Moving security checks **earlier** (left on the timeline: code → build → test → deploy → run): IDE linting, pre-commit secret scanning, SAST on PRs, dependency scans at build, image scans before push.
**Follow-up — "Is your pipeline shifted left?"** Partly: Sonar (planned) is early; Trivy runs *after* the push → should move left of the push.

### Q5. Security by Design?
Security considered during architecture, not bolted on: least privilege, secure defaults, minimal attack surface, defence in depth, fail-safe defaults, secrets out of code.
**Project (honest):** good — secrets in Vault, requests/limits, bcrypt/JWT. Gaps — unauthenticated webhook, default JWT secret, root Vault token.

### Q6. SDLC and secure SDLC stages
| SDLC stage | Security activity | Project example |
|---|---|---|
| Requirements | Security requirements, abuse cases | "Users must only see their own projects" |
| Design | Threat modelling (STRIDE), architecture review | Tenant isolation, secrets handling |
| Implementation | Secure coding, SAST, secret scanning | SonarCloud |
| Testing | SCA, DAST, pen test | Trivy (fs/image); DAST not done |
| Deployment | Hardened configs, signed images, IaC scan | Requests/limits; no securityContext |
| Operations | Monitoring, patching, incident response | Prometheus/Grafana/Loki |

### Q7. CI vs Continuous Delivery vs Continuous Deployment
| | Meaning | Human approval before prod? |
|---|---|---|
| **CI** | Every commit is built and checked automatically | n/a |
| **Continuous Delivery** | Every passing build is *releasable* | Yes (button) |
| **Continuous Deployment** | Every passing build is *released automatically* | No |
**Project:** CI + continuous deployment (the Deploy stage runs automatically).

### Q8. What are security gates?
Automated pass/fail checkpoints that **block** progression when a policy is violated (e.g. "no CRITICAL CVEs", "Sonar quality gate = Passed"). They convert *reporting* into *enforcement*.
**Project:** ⚠️ none enforced. Sonar stage is `catchError → UNSTABLE`; Trivy never exits non-zero.

### Q9. Vulnerability, threat, risk
| Term | Meaning | Example |
|---|---|---|
| **Vulnerability** | A weakness | Webhook has no authentication |
| **Threat** | Something that can exploit it | An attacker on the network |
| **Risk** | Likelihood × impact | Fake "Deployed" status → misleading dashboard (medium) |
**Formula:** risk = threat × vulnerability × impact.

### Q10. Principle of least privilege
Each user/tool gets **only** the access it needs, for as long as it needs it.
**Project:** ✅ per-container CPU/memory limits; ✅ owner-scoped queries. ⚠️ Jenkins uses Vault **root token**; Jenkins has `docker.sock`; Grafana anonymous = Admin. **Fix:** AppRole + read-only policy on `secret/data/sonarcloud`.

### Q11. Zero Trust
"Never trust, always verify": no implicit trust based on network location; every request is authenticated, authorised and encrypted; continuous verification; micro-segmentation.
**Project:** opposite in places — Jenkins→backend webhook and backend→Prometheus trust the network. **Zero-Trust moves:** webhook HMAC, NetworkPolicies, mTLS (service mesh), short-lived credentials.

### Q12. Defence in depth
Multiple independent layers so one failure doesn't mean compromise.
| Layer | Project control | Gap |
|---|---|---|
| Code | SAST (partial) | no gate |
| Dependencies/images | Trivy | after push, non-blocking |
| Secrets | Vault | dev mode, root token |
| Runtime | requests/limits | no securityContext |
| Network | none | no NetworkPolicy |
| Detection | Prometheus/Loki | no alerts |
| Identity | JWT/bcrypt | no rate limit, default secret |

### Q13. OWASP Top 10 (2021) mapped to your project
| OWASP | Where it shows up in your code |
|---|---|
| A01 Broken Access Control | `/api/monitoring/:p`, `/api/logs/:p`, webhook have no auth/ownership check |
| A02 Cryptographic Failures | Hardcoded fallback JWT secret; HTTP everywhere; token in `localStorage` |
| A03 Injection | `execSync(\`… grep "^${projectName}-"\`)` → **command injection** |
| A04 Insecure Design | Single namespace "multi-tenancy"; webhook trust |
| A05 Security Misconfiguration | Grafana anon Admin; Jenkins CSRF off/privileged; open CORS; Vault dev |
| A06 Vulnerable Components | Why Trivy/`npm audit` matter; `:latest` images |
| A07 Identification & Auth Failures | No rate limit/lockout; 8h non-revocable JWT |
| A08 Software & Data Integrity | Unsigned images, `:latest`, no webhook signature |
| A09 Logging & Monitoring Failures | No failed-login logging/alerts |
| A10 SSRF | Not obvious in this code (n/a) |

### Q14. CVE vs CVSS
- **CVE:** an **ID** for a publicly known vulnerability (`CVE-2024-12345`).
- **CVSS:** a **score 0–10** of severity (None 0, Low 0.1–3.9, Medium 4–6.9, High 7–8.9, Critical 9–10), from metrics: attack vector, complexity, privileges, user interaction, scope, CIA impact.
- CVSS ≠ risk: it ignores *your* context (is the package reachable? internet-exposed?). Complement with **EPSS** (exploit likelihood) and **CISA KEV** (known-exploited list).

### Q15. False positives vs false negatives
| | Meaning | Cost |
|---|---|---|
| **False positive** | Tool flags something that isn't a real issue | Wasted time, alert fatigue, devs ignore tool |
| **False negative** | Tool misses a real issue | False sense of security — worse |
**Handle:** triage with justification, suppress per finding (`.trivyignore` with expiry, Sonar "won't fix"/"false positive"), tune rules, combine tools. Never blanket-disable a gate.

### Q16. SAST vs DAST vs SCA vs container scanning vs IaC vs secret scanning
| Type | Looks at | When | Finds | Tools | Yours |
|---|---|---|---|---|---|
| SAST | Source code (static) | Early | Injection patterns, bugs, code smells | SonarCloud, Semgrep, CodeQL | 🟡 |
| SCA | 3rd-party libs | Build | Known-vulnerable deps, licences | npm audit, Snyk, Trivy fs | 📝 (Trivy fs not used) |
| Container scan | Image: OS pkgs + libs | After build, before push | CVEs in layers | Trivy, Grype | 🟡 |
| DAST | Running app | Test env | XSS, auth flaws, misconfig | OWASP ZAP, Burp | 📝 |
| IaC scan | YAML/Dockerfile/Terraform | PR/build | Missing limits, root, privileged | Trivy config, Checkov, kube-linter | 📝 |
| Secret scan | Repo + history | Pre-commit/PR | Committed keys | gitleaks, Trivy secret | 📝 |
**One-liner:** SAST reads the code; DAST attacks the running app; SCA reads the dependency list; container scanning reads the whole image.

### Q17. SBOM, supply-chain security, image signing
- **SBOM:** inventory of every component in an artifact (CycloneDX/SPDX). Lets you answer "are we affected by CVE-X?" in seconds. `trivy image --format cyclonedx -o sbom.json <img>`; `syft`.
- **Supply-chain security:** protecting everything between a developer's keyboard and production: source, dependencies, build system, registry, deployment. (SolarWinds, Log4Shell, `event-stream`, Codecov.)
- **Image signing:** cryptographically sign an image digest (cosign/Sigstore) and verify at admission (Kyverno / Sigstore policy-controller) so only trusted builds run.
**Project:** 📝 none implemented; `:latest` tags + no signing = weak supply-chain posture. SLSA levels are the framework to cite.

### Q18. Vulnerability remediation order
1. Triage: reachable? exposed? exploit available (KEV/EPSS)?
2. Prefer **upgrading** the package / **rebuilding on a newer base image**.
3. If no fix: mitigate (config, WAF, remove feature) and **accept risk with an expiry date**.
4. Re-scan, track as ticket with SLA (e.g. Critical 7 days, High 30).
5. Add a regression gate so it can't recur.

---

# PART 2 — CI/CD pipeline and Jenkins security

## 2.1 Your actual pipeline (from `Jenkinsfile`)
```
parameters: PROJECT_ID      tools: nodejs 'NodeJS 20'      agent any
1 Checkout          checkout scm                                   (platform repo)
2 Build Frontend    cd frontend; npm install; npm run build
3 Security Scan     catchError(SUCCESS/UNSTABLE){ withVault → SONAR_TOKEN; echo }   ⚠️ no scanner
4 Containerize      docker build/tag/push  backend+frontend  (:BUILD_NUMBER and :latest)
5 Trivy Image Scan  trivy image --format template … -o /reports/trivy-<PROJECT_ID>.html  (backend only)
6 Deploy to K8s     kubectl apply -f k8s/backend.yaml, frontend.yaml, hpa.yaml
post success        curl POST /api/webhook/jenkins  (HARDCODED sonar/trivy numbers + liveUrl)
post failure        curl POST … { status: "Failed" }
```

### Question 2.1: Walk me through your pipeline.
**Interview Answer:** "It's a declarative Jenkins pipeline with six stages, triggered by my backend through the `buildWithParameters` API with the project's MongoDB ID. Checkout pulls the code. Build Frontend runs npm install and build as an early check. The Security Scan stage fetches the SonarCloud token from Vault using the Vault plugin. Containerize builds the backend and frontend images, tags them with the build number and latest, and pushes to Docker Hub. The Trivy stage scans the backend image and writes an HTML report into a bind-mounted folder that my backend serves. Deploy applies the Kubernetes manifests. The post block calls my backend's webhook so the dashboard status updates. In the current version the Sonar stage is a placeholder and Trivy reports without blocking; I'd reorder so the scan runs before the push and fails on critical findings."

**Technical Deep Dive:** declarative pipeline = fixed structure (`pipeline/agent/stages/steps/post`) parsed and validated up-front; steps run on an executor on the controller (`agent any` → your single Jenkins container). `post` conditions (`success/failure/always/unstable`) run after stages. `parameters` populate `params.*`. `catchError(buildResult:'SUCCESS', stageResult:'UNSTABLE')` swallows a failure: stage turns yellow, build stays green.

**Project Connection:** exact stage list above. Also `withCredentials` for Docker Hub, `${BUILD_NUMBER}` env var for immutable tags.

**Possible Follow-ups**
- *Is the order right?* No — push-before-scan lets a vulnerable image reach Docker Hub. Order should be build → scan (gate) → push → deploy.
- *Does the pipeline build the user's repo?* No. `checkout scm` fetches the Jenkinsfile's own repo; the user's URL is stored but unused. Fix: pass `REPO_URL` as a parameter and `git url: params.REPO_URL`.
- *What does "Deployed" really mean?* Jenkins finished; it doesn't verify rollout. Add `kubectl rollout status deployment/… --timeout=120s` before reporting success.
- *Why do the Sonar numbers look constant?* They're hardcoded in `post`. Real numbers should come from the SonarCloud API (`/api/measures/component`) and Trivy JSON (`--format json`, parsed with `jq`).

**Quick Revision:** 6 stages + post · backend triggers, webhook reports · Trivy after push (wrong order) · Sonar is placeholder · numbers hardcoded.

---

### Question 2.2: Jenkins architecture — controller vs agent
**Interview Answer:** "The controller schedules jobs, stores configuration and credentials, and serves the UI. Agents (nodes) execute the builds. In my setup there's a single container, so the controller also runs builds via `agent any`. In production I'd keep the controller free of builds and use ephemeral agents — containers or Kubernetes pods — so a compromised build can't touch the controller or its secrets."

**Deep Dive:** controller ↔ agent via JNLP/WebSocket (port 50000 published in your compose) or SSH. Builds run in **executors**. `JENKINS_HOME` (your `jenkins_home` named volume) holds jobs, plugins, `credentials.xml`, `secrets/master.key`. Number of executors on controller should be 0 in production.

**Project Connection:** ports `8080` (UI), `50000` (agents); `jenkins_home` volume persists jobs/plugins/credentials across container recreation.

**Follow-ups**
- *Why is it dangerous that builds run on the controller?* A pipeline step can read `JENKINS_HOME` — including `secrets/master.key` and `credentials.xml` — i.e. decrypt every stored credential.
- *Why ephemeral agents?* Clean state per build, no cross-build contamination, easy scaling, destroyed after use.
- *What does `agent any` mean?* Any available executor, with no isolation guarantee.

**Quick Revision:** controller = brain, agents = muscle · executors · JENKINS_HOME · 0 executors on controller · ephemeral agents.

---

### Question 2.3: Declarative vs Scripted; Pipeline as Code
| | Declarative | Scripted |
|---|---|---|
| Syntax | Structured `pipeline { stages { … } }` | Free-form Groovy `node { … }` |
| Validation | Up-front, clear errors | Runtime errors |
| Flexibility | Lower (use `script {}` escape hatch) | Very high |
| Yours | ✅ (with `script {}` in Containerize, Trivy, post) | — |
**Pipeline as Code:** the pipeline lives in `Jenkinsfile` in Git → versioned, reviewable, reproducible, auditable, restorable.
**Follow-up — *Security risk of pipeline as code?*** Whoever can change the Jenkinsfile can run arbitrary code with the pipeline's credentials. Mitigate: protected branches + required review, separate trusted pipeline library, restrict who can edit/replay, don't run untrusted PR builds with secrets.

---

### Question 2.4: Credentials and secret exposure
**Interview Answer:** "Docker Hub credentials are in the Jenkins credentials store and injected with `withCredentials`, and I pipe the password via `--password-stdin` so it isn't on the command line. The SonarCloud token lives in Vault and `withVault` injects it as an environment variable only inside that block. One thing I'd fix: my Security Scan stage echoes the token into the build log, which defeats the purpose."

**Deep Dive:** Jenkins **masks** values of bound credentials in console output (`****`), but masking is string matching — it fails if the value is transformed (base64, split) or written to files/artifacts. Never `echo`, `set -x` with secrets, or pass them as `docker build --build-arg` (stays in image history). Single-quote `sh '…$SECRET…'` so Groovy doesn't interpolate secrets into the command (leaks into process list / logs): your `sh 'echo $DOCKER_PASS | …'` is correct; `sh "…${SECRET}…"` is not.

**Project Connection:** `withCredentials([usernamePassword(credentialsId:'docker-hub-credentials',…)])`; `withVault([…vaultCredentialId:'vault-root-token'…])`. ⚠️ Jenkins API token and username hardcoded in `server.js`.

**Follow-ups**
- *Where are Jenkins credentials stored and how are they encrypted?* `credentials.xml` in JENKINS_HOME, encrypted with keys in `secrets/` (AES; `master.key` + `hudson.util.Secret`). Anyone with the volume or script-console access can decrypt.
- *Jenkins "secret leaked in log" — what do you do?* See Scenario 9.2.
- *Credentials store vs Vault?* Store = simple, built in, per-Jenkins; Vault = central, policy + audit, dynamic secrets, reusable by apps. Your pipeline uses both (Jenkins needs a credential to reach Vault).

**Quick Revision:** `withCredentials`/`withVault` scope secrets · masking is not security · single-quoted `sh` · `--password-stdin` · rotate leaked secrets.

---

### Question 2.5: Jenkins hardening (RBAC, agents, supply chain)
**Interview Answer:** "My Jenkins is a local convenience setup: privileged, root, docker.sock mounted, CSRF protection disabled and setup wizard skipped. In production I'd enable security realm + matrix/role-based authorization (Role Strategy plugin) so developers can build but not edit credentials or system config, turn CSRF back on and use API tokens/crumbs, run agents as ephemeral unprivileged pods, build images rootlessly with Kaniko or BuildKit instead of mounting docker.sock, pin and update plugins, and keep Jenkins behind HTTPS and SSO."

**Deep Dive**
| Control | What / why |
|---|---|
| Authentication | Security realm (LDAP/OIDC/GitHub); no anonymous access |
| Authorization (RBAC) | Matrix Auth / Role Strategy: Admin, Developer, Viewer; folder-level roles; least privilege |
| CSRF | Crumb issuer on; your `DISABLE_CSRF_PROTECTION=true` allows forged cross-site build triggers |
| Agent security | Agent→controller access control; ephemeral agents; no builds on controller |
| Script Security | Sandbox for Groovy; approve scripts deliberately (sandbox escapes are a known class of vuln) |
| Plugins | Few, updated; Jenkins plugin CVEs are frequent |
| Secrets | Credentials store/Vault; folder-scoped credentials; no secrets in env of untrusted jobs |
| Network | TLS, firewall, no public exposure; restrict `/script` console |
| docker.sock | Equivalent to root on host: `docker run -v /:/host …` mounts the whole host filesystem |

**Project Connection:** compose: `privileged: true`, `user: root`, `/var/run/docker.sock`, `-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true`, `-Djenkins.install.runSetupWizard=false`.

**Follow-ups**
- *"An attacker compromises a Jenkins agent — impact?"* → Scenario 9.5.
- *How would you stop unauthorized pipeline modification?* Branch protection + CODEOWNERS on `Jenkinsfile`; multibranch pipeline from trusted branch only; restrict "Replay" and job-configure permission; Shared Library from a locked repo; audit log plugin.
- *How do you secure the CI/CD supply chain?* Pin base images by digest, verify dependencies (lockfiles, `npm ci`), scan, sign artifacts (cosign), generate SBOM/provenance (SLSA), isolate builders, least-privilege registry tokens, protect the Jenkinsfile.

**Quick Revision:** enable authN+RBAC · CSRF on · no builds on controller · rootless builds · docker.sock = root · protect the Jenkinsfile.

---

### Question 2.6: Webhooks and triggers
**Interview Answer:** "A webhook is an HTTP POST a system sends when an event happens. I use two: my backend triggers Jenkins through its REST API when a project is created, and Jenkins calls my backend's `/api/webhook/jenkins` when a build finishes. A GitHub push webhook to Jenkins would be the standard trigger; I haven't configured it. Polling SCM is the alternative, but it wastes requests and adds latency."

**Deep Dive:** GitHub webhook → `POST /github-webhook/` on Jenkins (GitHub plugin) with `X-Hub-Signature-256` HMAC header computed from a shared secret. **Verify the signature** (constant-time compare) before trusting the payload.
**⚠️ Your webhook:** `/api/webhook/jenkins` has no auth. Fix:
```js
const sig = req.headers['x-signature'];
const expected = crypto.createHmac('sha256', process.env.WEBHOOK_SECRET)
                       .update(JSON.stringify(req.body)).digest('hex');
if (!sig || !crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expected))) return res.sendStatus(401);
```
**Follow-ups**
- *What if the webhook is lost?* Project stays "Building" forever. Add timeout/reconciliation job and retries (`curl --retry`).
- *Idempotency?* Replays should not corrupt state; include build number and ignore stale updates.

---

### Question 2.7: Failure handling — what should happen when a scan fails?
**Interview Answer:** "It depends on severity and policy. For CRITICAL or HIGH findings with a fix available, the pipeline must fail and block deployment — that's the point of a security gate. Lower severities should be reported and tracked with an SLA, not block. Today my pipeline reports but doesn't block: the Sonar stage is wrapped in `catchError` and Trivy has no exit code. I'd change Trivy to `--exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed` before the push, and use the Sonar quality gate (`-Dsonar.qualitygate.wait=true`)."

**Deep Dive:** `UNSTABLE` vs `FAILURE`; `post { failure }` for notification; exceptions via a reviewed `.trivyignore` with expiry and justification; "break-glass" override requiring approval and audit; avoid gating on noisy findings (alert fatigue → devs bypass).
```groovy
sh '''trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed \
      --format json -o trivy.json $IMAGE'''
```
**Follow-ups**
- *Pipeline keeps failing because of the gate — what now?* → Scenario 9.12.
- *Should a medium vuln block?* Usually not; track with SLA.

---

### Question 2.8: Build artifacts and image creation
**Interview Answer:** "The artifacts are the Docker images — backend and frontend — each tagged with the Jenkins build number and `latest`, plus the Trivy HTML report. The build-number tag is immutable per build; `latest` moves. I should deploy the build-number tag so `kubectl apply` actually sees a change and rollbacks are possible."

**Deep Dive:** `latest` pitfalls: no change in YAML text → `apply` says "unchanged"; can't tell what's running; restarted pods may pull a different image; rollback ambiguous. Best: tag with **git SHA** and pin by **digest** (`image@sha256:…`), `imagePullPolicy: IfNotPresent`. `kubectl set image` / template the tag into the YAML; `kubectl rollout undo` to revert.

---

### Question 2.9: Environment variables and secrets in the app
`process.env` + `dotenv` loads `backend/.env`. ⚠️ **Fact:** your `.env` contains only `PORT`, so `JWT_SECRET` and `MONGO_URI` use hardcoded defaults. Fixes: remove fallback for secrets (`if(!process.env.JWT_SECRET) process.exit(1)`), `.gitignore` `.env`, inject via Kubernetes Secret/Vault, rotate.

---

# PART 3 — Security tools you used (full deep dive)

Each tool follows the 9-point structure: *what · internals · why · integration · output · detection · interpretation · pipeline impact · alternatives*.

## 3.1 HashiCorp Vault

**Question:** What is Vault and how did you use it?

**Interview Answer:** "Vault is a secrets management server. Instead of keeping the SonarCloud token in the Jenkinsfile or an environment variable, I stored it in Vault's KV v2 engine at `secret/sonarcloud` under the key `token`. During the pipeline, the Jenkins Vault plugin's `withVault` step authenticates to Vault, reads that secret and injects it as `SONAR_TOKEN` just for that block. I ran Vault in dev mode, which is in-memory, so my secrets vanished on restart — I understood that production needs server mode with persistent storage and unsealing, and a least-privilege AppRole instead of the root token I used."

| # | Point | Answer |
|---|---|---|
| 1 What | Central secrets store with encryption, auth, policies, audit |
| 2 Internals | Data encrypted with a **data-encryption key**, itself encrypted by a **master key**. Server mode starts **sealed**; the master key is split with **Shamir's Secret Sharing** (e.g. 5 shares, threshold 3). Unseal → master key reconstructed in memory → barrier decrypts storage. Storage backend (Raft/Consul/file) only ever sees ciphertext |
| 3 Why | Keep secrets out of code/Git/env, central rotation, audit, policy |
| 4 Integration | Compose service `hashicorp/vault`, `vault server -dev -dev-root-token-id=…`; secret via `vault kv put secret/sonarcloud token=…`; Jenkins: `withVault([configuration:[vaultUrl:'http://devsecops-vault:8200', vaultCredentialId:'vault-root-token'], vaultSecrets:[[engineVersion:2, path:'secret/sonarcloud', secretValues:[[envVar:'SONAR_TOKEN', vaultKey:'token']]]]])` |
| 5 Output | `SONAR_TOKEN` env var in the block |
| 6–7 Detection/interpretation | n/a (not a scanner). Failure signature: empty `SONAR_TOKEN` ⇒ wrong path/key (your challenge) or secret wiped (dev mode) |
| 8 Pipeline impact | Missing secret → stage UNSTABLE via `catchError` (silently!) |
| 9 Alternatives | Jenkins credentials store (simpler, no dynamic secrets), K8s Secrets (base64 only), AWS Secrets Manager/SSM, Azure Key Vault, SOPS/Sealed Secrets |

**Technical deep dive**
- **Dev vs server mode:** dev = in-memory storage, auto-unsealed, HTTP, known root token. Server = persistent storage, sealed, TLS, root token generated once at `vault operator init`.
- **Auth methods:** token, **AppRole** (role_id + secret_id → short-lived token) for machines, Kubernetes auth (pod service account JWT), OIDC/LDAP for people.
- **Policies (HCL):**
  ```hcl
  path "secret/data/sonarcloud" { capabilities = ["read"] }
  ```
  KV v2 API paths include `/data/` (`secret/data/…`), metadata under `/metadata/`; v2 keeps versions.
- **Dynamic secrets:** Vault creates short-lived DB/cloud creds on demand and revokes by lease TTL.
- **Audit devices:** every request/response (hashed) logged.
- **Vault vs K8s Secrets:** K8s Secrets are base64 in etcd (encrypted only if EncryptionConfiguration is enabled), RBAC-scoped to the cluster; Vault adds encryption barrier, policies, audit, dynamic secrets, tool-agnostic.

**Project connection / challenges**
1. Key mismatch (`token` vs `SONAR_TOKEN`) → empty value. 2. Dev mode → secrets lost on restart.
⚠️ Root token in compose + notes; Jenkins authenticates with root. ⚠️ `echo "… Token: ${SONAR_TOKEN}"` prints the secret.

**Follow-ups**
- *Walk me through what happens during `withVault`.* Plugin reads the Jenkins credential (`vault-root-token`), calls `GET /v1/secret/data/sonarcloud` with `X-Vault-Token`, parses `data.data.token`, exports it to env for the block, masks it, discards after.
- *What if Vault is down?* Secret fetch fails; with `catchError` the stage silently goes UNSTABLE and the build continues — a security stage should **fail closed**.
- *How would you rotate the SonarCloud token?* Generate new in SonarCloud, `vault kv put` (new version), Jenkins picks it up next run, revoke old. With KV v2 you can roll back versions.
- *What does "sealed" protect against?* Stolen disk/backup: without the unseal shares, data is ciphertext.
- *Why not put the token in a Jenkins credential?* Acceptable for one tool; Vault scales across apps with policy/audit/dynamic secrets.

**Quick Revision:** KV v2 path/key must match exactly · dev = RAM + auto-unseal + root token · prod = sealed + persistent + AppRole + policy · fail closed · never echo secrets.

---

## 3.2 SonarCloud (SAST)

**Question:** What is SonarCloud? How does it find vulnerabilities?

**Interview Answer:** "SonarCloud is a hosted static-analysis service. It parses source code into an abstract syntax tree and applies rules to detect bugs, vulnerabilities, security hotspots and code smells, then applies a quality gate to decide pass/fail. In my repo, `sonar-project.properties` defines the project key and sources (`backend`, `frontend/src`), and Vault supplies the token. In the current Jenkinsfile the stage only fetches the token — the actual `sonar-scanner` invocation and quality-gate wait aren't there yet — so that's the first thing I'd complete."

| # | Point | Answer |
|---|---|---|
| 1 What | SAST + code quality, hosted (SonarQube = self-hosted) |
| 2 Internals | Language analyzers build an AST/control-flow graph; rules pattern-match; **taint analysis** tracks untrusted input (source) to dangerous calls (sink) across functions. Scanner uploads results to the server; server computes metrics, issues, gate status |
| 3 Why | Free for open source, no infra to run, PR decoration, quality gates |
| 4 Integration | `sonar.projectKey=chahak02_devsecops-platform`, `sonar.organization`, `sonar.sources=backend,frontend/src`, `sonar.host.url=https://sonarcloud.io`; token via `SONAR_TOKEN` |
| 5 Output | Dashboard: Bugs, Vulnerabilities, Security Hotspots, Code Smells, Coverage, Duplications, Quality Gate |
| 6 Detection | Rule match / taint flow, e.g. `req.params` → `execSync(...)` = command injection (S2076) |
| 7 Interpretation | Vulnerability = confirmed risky code; **Hotspot** = needs human review; Smell = maintainability. Triage: Fix / Won't fix / False positive |
| 8 Pipeline impact | Target: `-Dsonar.qualitygate.wait=true` → scanner exits non-zero if gate fails → build fails |
| 9 Alternatives | SonarQube (self-host), Semgrep, CodeQL, Snyk Code, Checkmarx; trade-off: depth/false positives/cost/language support |

**Correct scanner stage (what to write):**
```groovy
withVault([...]) {
  sh '''
    sonar-scanner -Dsonar.projectKey=$KEY -Dsonar.organization=chahak02 \
      -Dsonar.sources=backend,frontend/src -Dsonar.host.url=https://sonarcloud.io \
      -Dsonar.token=$SONAR_TOKEN -Dsonar.qualitygate.wait=true
  '''
}
```
(no `catchError`, so a failed gate fails the build)

**Your challenge:** every project showed identical numbers → one hardcoded project key. Per-tenant analysis needs a key per project (created via SonarCloud API `POST /api/projects/create`) and scanning that tenant's repo.

**A finding SonarCloud would flag in *your* code:** `execSync(\`kubectl get pods … | grep "^${projectName}-"\`)` — user-controlled `projectName` → OS command injection (OWASP A03). Fix: validate `/^proj-[a-z0-9]{5}$/`, use `execFile`/the Kubernetes client with argument arrays, add auth + ownership check.

**Follow-ups**
- *SAST false positives?* Common (no runtime context). Triage, mark with reason, tune the quality profile; don't disable the rule globally.
- *SAST vs SCA?* SAST analyses **your** code; SCA analyses **dependencies**.
- *What's a quality gate vs a quality profile?* Gate = pass/fail conditions on metrics (new-code focus); profile = the rule set activated.
- *Clean-as-you-code?* Gate applies to **new code** so legacy debt doesn't block.

**Quick Revision:** AST + taint analysis · bugs/vulns/hotspots/smells · quality gate enforces · yours is placeholder (token only) · one key = identical results.

---

## 3.3 Trivy (container / SCA / IaC / secret scanner)

**Question:** What is Trivy and how does it find vulnerabilities?

**Interview Answer:** "Trivy is an open-source scanner from Aqua. For an image, it unpacks the layers, identifies the OS (Alpine from `node:20-alpine`) and installed packages from the package database, and also reads language manifests like `package-lock.json`. It matches each name+version against its vulnerability database — built from sources like NVD, Alpine secdb, GitHub advisories — and reports CVEs with severity and fixed versions. In my pipeline it scans the backend image and writes an HTML report into a shared folder that my backend serves and the dashboard links to. Currently it runs after the push and doesn't fail the build, so I'd move it before the push with `--exit-code 1 --severity CRITICAL,HIGH`."

| # | Point | Answer |
|---|---|---|
| 1 What | Scanner for images, filesystems, repos, IaC misconfig, secrets, SBOM |
| 2 Internals | Pulls image (here via `docker.sock`), reads layers, extracts package lists (apk/dpkg/rpm DBs) + lockfiles → joins with local **vulnerability DB** (downloaded on first run / cached) → version-range comparison → findings. Detects **known** CVEs only (no zero-days) |
| 3 Why | Single binary, fast, no server, many targets, templates (HTML), free |
| 4 Integration | Installed in Jenkins image (`apt-get install trivy`); `wget …/contrib/html.tpl`; `trivy image --format template --template "@contrib/html.tpl" -o /reports/trivy-${PROJECT_ID}.html chahak02/devsecops-backend:${BUILD_NUMBER}` |
| 5 Output | HTML table: Target, Package, CVE ID, Severity, Installed version, Fixed version, Title |
| 6 Detection | Package+version in affected range of a CVE advisory |
| 7 Interpretation | Severity (CVSS-based + vendor), **fixed version available?**, reachable? exposed? Distinguish OS vs app-package findings |
| 8 Pipeline impact | Today: none (no exit code). Target: `--exit-code 1` on severity threshold |
| 9 Alternatives | Grype/Syft, Snyk, Clair, Docker Scout, Anchore; trade-offs: DB coverage, licence, speed, SBOM support |

**Report delivery chain:** Trivy (in Jenkins container) → `/reports` (bind mount to host `infrastructure/reports`) → Express `app.use('/reports', express.static(...))` → webhook sends `trivyReportUrl` → stored in MongoDB → dashboard button opens it.
⚠️ Static path is hardcoded to `/Users/veeralagarwal/…`; report is overwritten per `PROJECT_ID`; it only covers the backend image.

**Useful commands**
```bash
trivy image --severity CRITICAL,HIGH --ignore-unfixed IMG        # scan
trivy image --exit-code 1 --severity CRITICAL IMG                # gate
trivy image -f json -o out.json IMG                              # machine-readable
trivy image --format cyclonedx -o sbom.json IMG                  # SBOM
trivy fs --scanners vuln,secret,misconfig .                      # repo: SCA + secrets + IaC
trivy config k8s/                                                # K8s/Docker misconfig
```
**What Trivy would find in your repo:** misconfig: no `securityContext`, `runAsNonRoot` not set, `:latest` tag, hostPath mounts; secret: tokens in committed files.

**Follow-ups**
- *Why scan before push?* Registry shouldn't ever hold known-critical images; shortens exposure; fail fast.
- *Image scan finds nothing — is it safe?* No: unknown/zero-day vulns, misconfig, logic bugs and runtime threats aren't covered (false negatives).
- *`--ignore-unfixed`?* Skip vulns with no patch so you only gate on actionable items.
- *How to reduce findings?* Smaller base (alpine/distroless), multi-stage builds, `npm ci --omit=dev`, update base regularly, rebuild nightly.
- *Image already deployed and a new CVE appears?* Scans are point-in-time. Rescan registry images on a schedule / use admission or continuous scanning (Trivy Operator).
- *Trivy DB freshness?* DB updates frequently; cache it and pin in CI, or use `--skip-db-update` in air-gapped with an internal mirror.

**Quick Revision:** layers → packages → DB match · known CVEs only · you scan backend only, after push, non-blocking · gate with `--exit-code 1` · distroless/alpine reduce surface.

---

## 3.4 Application-level security of your own backend (a likely deep-dive)

| Area | Your code | Interview note |
|---|---|---|
| Password storage | `bcrypt.hash(pw,10)` ✅ | Salted, adaptive; cost 10 ≈ 2¹⁰ rounds; consider 12 |
| Auth | JWT HS256 signed with `JWT_SECRET`, 8h | ⚠️ default secret in use; no revocation/refresh; `localStorage` (XSS-readable) |
| Error messages | Same `Invalid credentials` for bad email/password ✅ | Prevents user enumeration |
| Registration | `error.message` returned | Leaks internals (duplicate key details) |
| Headers | `helmet()` ✅ | Sets CSP-ish/HSTS/X-Frame etc. |
| CORS | `cors()` wide open ⚠️ | Restrict to dashboard origin |
| Rate limiting | none ⚠️ | `express-rate-limit` on login (brute force) |
| Input validation | none | Validate with zod/joi; `name`/`repo` are free text |
| Authorisation | owner filter on projects ✅ | Not applied to monitoring/logs/webhook ⚠️ |
| Injection | `execSync` with interpolated `projectName` ⚠️ | Use argument arrays / K8s client |
| Webhook | unauthenticated ⚠️ | HMAC |
| Role | `role` in token, never enforced | Implement RBAC middleware |

**Q: Explain JWT internals.** Three base64url parts: header (`alg`, `typ`), payload (claims: `id, username, role, iat, exp`), signature = `HMAC_SHA256(base64(header)+"."+base64(payload), secret)`. Payload is **readable by anyone**; signature prevents **tampering**. Verification = recompute signature + check `exp`. Algorithm confusion (`alg:none`) is prevented by pinning algorithms in `jwt.verify(token, secret, {algorithms:['HS256']})`.
**Follow-ups:** *Why 401 vs 403 here?* Missing token → 401; invalid/expired → 403 (arguably 401). *JWT vs sessions?* Stateless scaling across your 2 replicas vs easy revocation. *How to revoke?* Short access tokens + refresh tokens, denylist, token versioning.

**Q: How would you detect brute force on your login?** Log failed logins with IP/email (`console`/structured JSON) → Promtail → Loki → LogQL `sum(rate({app="backend"} |= "Invalid credentials" [5m])) by (ip)`; or expose a Prometheus counter `login_failures_total{reason}` with `prom-client` and alert `rate(login_failures_total[5m]) > 5`. Add rate limit + lockout/captcha. (⚠️ today: no failed-login logging beyond morgan's `401` access line.)

---

## 3.5 Secret management summary (what an interviewer may probe)
| Secret | Where it is today | Better |
|---|---|---|
| SonarCloud token | Vault (dev) ✅; also visible in your notes PDF ⚠️ | Rotate; AppRole read-only |
| Vault root token | compose file + Jenkins credential + notes ⚠️ | Init/unseal properly; revoke root |
| Jenkins API user/token | Hardcoded in `server.js` ⚠️ | Env var/Vault; rotate |
| Jenkins job token | Hardcoded `devsecops_secret_token` | Env var |
| JWT secret | Hardcoded fallback in use ⚠️ | Required env var from Secret |
| Docker Hub creds | Jenkins credential store ✅ | Scoped access token, not password |
| Grafana admin | `admin/admin` (Compose) / anon Admin (Helm) ⚠️ | Real password, Viewer role |
| Mongo | no auth | Enable auth, network-restrict |
**Action before any interview/sharing the repo: rotate every item above.** Also `git log -p | grep` for historical leaks; use `gitleaks`.

---

# PART 4 — Docker and container security

## 4.1 Your Dockerfiles
```dockerfile
# backend/Dockerfile                       # frontend/Dockerfile
FROM node:20-alpine                        FROM node:20-alpine as build
WORKDIR /app                               WORKDIR /app
COPY package*.json ./                      COPY package*.json ./
RUN npm install                            RUN npm install
COPY . .                                   COPY . .
EXPOSE 5000                                RUN npm run build
CMD ["node","server.js"]                   FROM nginx:alpine                     # multi-stage
                                           COPY --from=build /app/dist /usr/share/nginx/html
                                           EXPOSE 80 ; CMD ["nginx","-g","daemon off;"]
```
**Strengths:** alpine bases (small), layer-cache-friendly order, multi-stage frontend (build tools not shipped).
**Gaps ⚠️:** backend runs as **root** (no `USER node`); `npm install` instead of `npm ci --omit=dev`; no `.dockerignore` seen (could copy `node_modules`, `.env`, `backend.log` into the image — **`.env` baked into the image** is a real risk); no `HEALTHCHECK`; base not pinned by digest; `COPY . .` copies everything.

**Hardened backend Dockerfile**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --chown=node:node . .
USER node
EXPOSE 5000
HEALTHCHECK CMD wget -qO- http://localhost:5000/api/health || exit 1
CMD ["node","server.js"]
```

## 4.2 Core Q&A

### Question: Explain Docker architecture.
**Interview Answer:** "Docker is client–server. The `docker` CLI talks over a REST API on a Unix socket (`/var/run/docker.sock`) to the **dockerd** daemon, which manages images, networks and volumes and delegates container lifecycle to **containerd**, which uses **runc** to create the container using Linux namespaces and cgroups. Images come from a registry like Docker Hub."
**Deep dive:** `docker build` → BuildKit executes Dockerfile → layers (content-addressed, read-only) → image manifest + config. `docker run` adds a thin writable layer (overlayfs). Image identity = digest (`sha256:…`); tags are mutable pointers.
**Project:** Jenkins container's `docker` CLI uses the **host's** daemon through the mounted socket, so images are built by the host Docker, not inside Jenkins.
**Follow-ups:** *Docker vs containerd vs runc?* dockerd = management; containerd = runtime manager; runc = OCI low-level runtime. *Why does Kubernetes not need Docker?* It talks CRI to containerd/CRI-O (Minikube with `--driver=docker` still runs the node as a Docker container).

### Question: How do containers achieve isolation? (namespaces and cgroups)
**Interview Answer:** "A container is just a Linux process with restricted views and resources. **Namespaces** restrict what it can *see*: PID (own process tree), NET (own interfaces/ports), MNT (own filesystem), UTS (hostname), IPC, USER (UID mapping), cgroup. **cgroups** restrict what it can *use*: CPU, memory, I/O, PIDs. Containers share the host kernel, so isolation is weaker than a VM."
**Deep dive:** Kubernetes `limits.memory: 256Mi` → cgroup `memory.max`; exceeding → kernel OOM-killer → `OOMKilled`. `limits.cpu: 500m` → CFS quota (throttling, not killing). Other hardening layers: **capabilities** (drop `ALL`), **seccomp** (syscall filter), **AppArmor/SELinux**, read-only rootfs, non-root, user namespaces.
**Project:** your `requests/limits` map to cgroups; this is why CPU throttles but memory kills.
**Follow-ups:** *Container vs VM?* Shared kernel → kernel exploits escape; VM has hypervisor boundary. *Container escape examples?* privileged container, mounted `docker.sock`, `hostPath: /`, `CAP_SYS_ADMIN`, runc CVE-2019-5736, Dirty COW/Dirty Pipe kernel bugs.

### Question: Why is mounting `docker.sock` dangerous?
**Answer:** Whoever can talk to the Docker API can start a container with `-v /:/host --privileged` and read/write the whole host as root. So `docker.sock` access ≡ root on the host. **Your Jenkins** has it plus `privileged: true` and `user: root`: a malicious Jenkinsfile (or a compromised plugin) controls the laptop.
**Mitigations:** rootless builders (Kaniko, BuildKit rootless, Buildah), ephemeral build VMs/pods, Docker socket proxy with a restricted API, rootless Docker/Podman, separate build host.

### Question: Image vs container; layers; caching
Image = immutable template (layers + metadata); container = running instance + writable layer. Each Dockerfile instruction (`RUN/COPY/ADD`) adds a layer; cache invalidates from the first changed instruction down — so copy `package*.json`, install, **then** copy source (your order ✅). Layers are shared across images, saving space.
**Follow-up:** *Does deleting a file in a later layer remove it from the image?* No — it stays in the earlier layer (visible via `docker history`/`save`). That's why secrets copied then deleted still leak. Use multi-stage or BuildKit secrets (`RUN --mount=type=secret`).

### Question: RUN vs CMD vs ENTRYPOINT; EXPOSE
`RUN` = build-time; `CMD` = default run command (overridable); `ENTRYPOINT` = fixed executable (CMD supplies args). `EXPOSE` documents a port, does **not** publish it; `-p host:container` publishes. Prefer **exec form** `["node","server.js"]` so signals (SIGTERM) reach the process (PID 1) for graceful shutdown.

### Question: Container networking
Default `bridge` network: each container gets a private IP on a virtual bridge, NAT to host. User-defined bridge (Compose) adds **DNS by container/service name** — why Jenkins reaches `devsecops-vault:8200`. `host` network shares host stack. `-p 8081:8080` = DNAT from host:8081 to container:8080. `host.docker.internal` = host gateway alias (Docker Desktop).
**Project connection table**
| From | To | Address |
|---|---|---|
| Jenkins container | Vault container | `devsecops-vault:8200` |
| Jenkins container | Backend on host | `host.docker.internal:5000` |
| Backend on host | MongoDB container | `localhost:27017` (published) |
| Promtail | Loki | `loki:3100` |
**Follow-up:** *Inside a container what does `localhost` mean?* The container itself.

### Question: Volumes vs bind mounts vs tmpfs
| | Named volume | Bind mount | tmpfs |
|---|---|---|---|
| Managed by | Docker | You (host path) | RAM |
| Persistence | Yes | Yes | No |
| Yours | `mongo_data`, `jenkins_home` | `./reports:/reports`, `docker.sock`, kubeconfig, `.minikube` | – |
**Security:** bind mounts expose host paths (writable by default) → mount `:ro` where possible; absolute `/Users/veeralagarwal/…` paths make the setup non-portable. The Trivy report bridge (Jenkins container → host → Express) is a bind-mount use case.

### Question: Image tagging and immutability
Tags are mutable; digests are immutable. Use `repo:${GIT_SHA}` + deploy by digest; avoid `latest` in manifests (YAML unchanged → no rollout; unknown version; pods may drift). Enable registry **tag immutability** (ECR) and signing.

### Question: How would you secure containers in production? (checklist)
1. Minimal/distroless base, pinned by digest, rebuilt regularly. 2. Non-root `USER`, `readOnlyRootFilesystem`. 3. Drop all capabilities, `allowPrivilegeEscalation:false`, seccomp `RuntimeDefault`. 4. No privileged, no `docker.sock`, no hostPath. 5. Scan in CI + registry + runtime. 6. Secrets via Vault/K8s Secrets, not image/env files. 7. Limits/requests. 8. Signed images + admission policy. 9. Runtime detection (Falco). 10. Network policies. 11. Patch host kernel.

### Question: Distroless vs alpine vs debian-slim
Distroless: no shell/package manager → tiny attack surface, hard for attackers to live off the land; harder to debug (use ephemeral debug containers). Alpine: small, musl libc (occasional compat issues), has shell/apk. Slim: glibc compat, larger.

### Useful commands
```bash
docker ps / docker logs -f <c> / docker exec -it <c> sh
docker inspect <c> | jq '.[0].HostConfig.Privileged, .[0].Config.User'
docker history <img> ; docker image ls --digests
docker run --read-only --cap-drop ALL --security-opt no-new-privileges -u 1000 <img>
docker stats ; docker system df ; docker network inspect <net>
docker compose up -d / down / logs -f jenkins
```

---

# PART 5 — Kubernetes and deployment security (your exact configs)

## 5.1 What your manifests actually say
| Manifest | Key facts |
|---|---|
| `k8s/backend.yaml` | Deployment `devsecops-backend`, **2 replicas**, `chahak02/devsecops-backend:latest`, port 5000, req 100m/128Mi, **limit 500m/256Mi**; Service **NodePort 30082** |
| `k8s/frontend.yaml` | Deployment + Service **NodePort**, similar |
| `k8s/hpa.yaml` | `autoscaling/v2` HPA for both: min 2, max 5, **CPU 70% utilisation** |
| `backend/templates/deployment-template.yaml` | Per project: Deployment (1 replica, `nginxdemos/hello`, port 80, req 100m/128Mi, **limit 500m/512Mi**, labels `app`,`tenant`) + Service **NodePort** 80→80 |
| `k8s/promtail.yaml` | ConfigMap + DaemonSet (`grafana/promtail:2.8.3`), hostPath `/var/lib/docker/containers`, `/var/log/pods`, `/run/promtail`; `serviceAccountName: default`; **placeholder Loki URL** |
| Not present | probes, securityContext, NetworkPolicy, RBAC, PodSecurity labels, ResourceQuota, LimitRange, ConfigMaps/Secrets for app config, PDB, Ingress |

## 5.2 Architecture
**Control plane:** kube-apiserver (front door; every kubectl/Jenkins/backend call), etcd (state store), kube-scheduler (placement using requests), kube-controller-manager (control loops: ReplicaSet, Deployment, Node…). **Node:** kubelet (runs pods, probes, reports), container runtime (containerd), kube-proxy (Service networking via iptables/IPVS). Minikube = both on one node (`192.168.49.2` in your docker driver).
**`kubectl apply` flow:** kubectl → API server (authN → authZ(RBAC) → admission → validate) → etcd → controllers create ReplicaSet/Pods → scheduler binds Pod to node → kubelet pulls image & starts via runtime → kube-proxy programs Service rules → kubelet reports status.

## 5.3 Core objects Q&A

**Pod / ReplicaSet / Deployment.** Pod = smallest unit (≥1 containers sharing network namespace/IP/volumes). ReplicaSet keeps N pods alive. Deployment manages ReplicaSets for rolling updates/rollbacks. Pod name `devsecops-backend-74b445b5d7-28hvj` = deployment + RS hash + random.
**Why random suffixes matter to you:** backend finds the pod with `kubectl get pods | grep "^proj-5cbc4-"`; better `-l app=proj-5cbc4` (labels) or `kubectl get pods -o jsonpath`.
**Follow-ups:** *Rolling update mechanics?* New RS scaled up while old scaled down within `maxSurge/maxUnavailable`; readiness probes gate progress — **you have none**, so a broken pod counts as ready once the process starts. `kubectl rollout status/undo/history`.

**Services.** ClusterIP (internal, default), NodePort (30000–32767 on every node), LoadBalancer (cloud LB; `<pending>` on Minikube), Headless, ExternalName; Ingress for L7 routing. Service selects pods by **label selector** → Endpoints. `port` (Service) vs `targetPort` (container) vs `nodePort`.
**Your story:** ClusterIP → browser can't reach; port-forward fragile; LoadBalancer pending; NodePort + `minikube ip` ✅.
**Follow-up:** *Is NodePort production-suitable?* No: exposes every node, odd ports, no TLS/host routing → Ingress + TLS + LB.

**ConfigMaps and Secrets.** ConfigMap = non-secret config (your Promtail config uses one). Secret = base64 (not encryption) objects in etcd; enable **encryption at rest**, restrict RBAC `get/list secrets`, prefer external secret managers (Vault Agent/CSI, External Secrets). **Yours:** app config via `.env` baked in image/hostile; no K8s Secret used.

**Resource requests/limits and QoS.** Request = scheduling guarantee; limit = cap. QoS: Guaranteed (req=limit), Burstable (yours), BestEffort. CPU over limit → throttled; memory over → `OOMKilled` → restart → possibly `CrashLoopBackOff`. HPA uses **utilisation = usage/request** — without a request no scaling. Needs **metrics-server** (`kubectl get hpa` shows `<unknown>` otherwise).
**Formula:** `desired = ceil(current × currentMetric / target)` → 2 pods at 140% with target 70% → 4.

**Probes.** liveness (restart), readiness (remove from Service endpoints), startup (slow starts). **None in your repo.** Add:
```yaml
readinessProbe: { httpGet: { path: /api/health, port: 5000 }, periodSeconds: 5 }
livenessProbe:  { httpGet: { path: /api/health, port: 5000 }, initialDelaySeconds: 15, periodSeconds: 10 }
```
**Caveat:** `/api/health` checks Mongo; using it for **liveness** could restart-loop everything during a DB outage — use separate lightweight liveness vs readiness endpoints.

**Namespaces and multi-tenancy.** Namespace = scope for names, RBAC, quotas, policies. **Alone it does not isolate network, resources or access.** Real isolation = namespace per tenant + **NetworkPolicy** (default deny) + **ResourceQuota/LimitRange** + **RBAC** + Pod Security Admission + (optionally) separate node pools/vClusters.
**Your honest statement:** "Per-project Deployment/Service, tenant labels, per-pod limits, all in `devsecops-prod`; namespace-per-tenant with policies is the next step."
**Namespace mix-up challenge:** applied YAML without `-n` → created in `default`. Always `-n` or set the context namespace; put `namespace:` in manifests.

## 5.4 Kubernetes security Q&A

**RBAC.** Role/ClusterRole (verbs on resources) + RoleBinding/ClusterRoleBinding (subject: user, group, ServiceAccount). Least privilege; avoid `cluster-admin`; `kubectl auth can-i --list --as system:serviceaccount:ns:sa`.
**Service accounts.** Pod identity; token mounted at `/var/run/secrets/kubernetes.io/serviceaccount` (bound, time-limited tokens in modern versions). Set `automountServiceAccountToken:false` where unused. **Yours:** all pods use `default` SA; Promtail with `kubernetes_sd_configs` needs a ClusterRole (`get/list/watch pods`) — missing, so it would be forbidden.
**Network policies.** Pod-level L3/L4 firewall, enforced by CNI (Calico/Cilium; Minikube default CNI may not enforce — start with `--cni=calico`).
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: devsecops-prod }
spec: { podSelector: {}, policyTypes: [Ingress, Egress] }
```
then allow-list (same-namespace ingress, DNS egress, Prometheus scrape from `monitoring`).
**Pod Security Standards:** Privileged / Baseline / Restricted, enforced by **Pod Security Admission** labels: `pod-security.kubernetes.io/enforce: restricted`. Restricted requires non-root, no privilege escalation, drop ALL caps, seccomp.
**securityContext (what yours lacks):**
```yaml
securityContext: { runAsNonRoot: true, runAsUser: 10001, allowPrivilegeEscalation: false,
                   readOnlyRootFilesystem: true, capabilities: { drop: ["ALL"] },
                   seccompProfile: { type: RuntimeDefault } }
```
**Admission controllers:** plugins that mutate/validate requests after authN/authZ: PodSecurity, ResourceQuota, LimitRanger, and dynamic webhooks (**OPA Gatekeeper, Kyverno**) to enforce policies like "only signed images from our registry, no `:latest`, must set limits".
**API server security:** TLS, authN (certs/OIDC/SA tokens), authZ (RBAC), audit logging, no anonymous, restrict public endpoint. **Yours:** `insecure-skip-tls-verify: true` in the Jenkins kubeconfig (from your notes) ⚠️ — disables server cert verification (MITM risk); correct fix is to mount the right CA cert.
**Runtime security:** Falco/Tetragon (syscall/eBPF rules: shell in container, write to `/etc`, unexpected outbound), seccomp/AppArmor, read-only FS.
**Common attack vectors:** exposed dashboard/API, over-privileged SA, privileged pods/hostPath/hostNetwork, secrets in env/Git, vulnerable images, no NetworkPolicy (lateral movement), SSRF to metadata/API, supply chain (malicious image), kubelet read-only port, etcd exposure, container escape.
**Your attack surface, concretely:** Jenkins has cluster credentials (kubeconfig) and docker.sock; backend shells out to `kubectl` with cluster credentials and builds commands from user input → **command injection ⇒ cluster access**. That chain (unauthenticated `/api/logs/:name` → shell → kubectl) is your most serious finding.

## 5.5 Kubernetes command cheat-sheet
```bash
kubectl get pods,svc,deploy,hpa,ep -n devsecops-prod -o wide
kubectl describe pod <p> -n devsecops-prod                # events: scheduling, pulls, probes, OOM
kubectl logs <p> -n devsecops-prod [--previous] [-f] [--tail=50]
kubectl get pods -l app=proj-5cbc4 -n devsecops-prod
kubectl get svc <svc> -n devsecops-prod -o jsonpath='{.spec.ports[0].nodePort}'
kubectl get endpoints <svc>                               # empty ⇒ selector/label mismatch
kubectl rollout status|history|undo deploy/<d>
kubectl top pods -n devsecops-prod                        # needs metrics-server
kubectl auth can-i --list ; kubectl get events --sort-by=.lastTimestamp
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
minikube ip ; minikube service <svc> ; minikube addons enable metrics-server
```
**Failure modes:** `Pending` (no capacity / unbound PVC / taints) · `ImagePullBackOff` (name/tag/credentials) · `CrashLoopBackOff` (app exits; use `logs --previous`) · `OOMKilled` (memory limit) · `Running` but unreachable (Service type, endpoints, namespace, wrong IP/port) · `Evicted` (node pressure).

---

# PART 6 — Monitoring, logging and observability (your core area)

## 6.1 What actually exists (from the repo)
```
 ┌─ Minikube ────────────────────────────────────────────────────────────┐
 │ ns monitoring (Helm: kube-prometheus-stack)                           │
 │   Prometheus ──scrape──▶ cAdvisor(kubelet), node-exporter,            │
 │                          kube-state-metrics, apiserver, …             │
 │   Grafana (anon Admin, subpath /grafana, embedding on)                │
 │   Alertmanager (chart default, no custom receivers in repo)           │
 │ ns devsecops-prod: platform pods + proj-xxxxx pods                    │
 └──────────────▲──────────────────────────▲─────────────────────────────┘
   port-forward 9090                port-forward 3000:80
 Host: Backend ── PromQL /api/v1/query ──▶ Prometheus      Browser iframe ──▶ Grafana
 Host: Backend ── kubectl logs --tail=50 ──▶ pod           (Logs page, NOT Loki)
 Docker Compose (monitoring/): Promtail ──push──▶ Loki ◀── Grafana Explore {job="varlogs"}
```
Two separate stacks exist: **(A)** Helm kube-prometheus-stack in Kubernetes (metrics — the real one) and **(B)** a Compose stack (Prometheus self-scrape only, Loki, Promtail, Grafana). Know which does what. `k8s/promtail.yaml` is a third, unfinished piece (placeholder Loki URL, no RBAC).

## 6.2 Fundamentals

### Question: Monitoring vs logging vs observability; metrics vs logs vs traces
**Interview Answer:** "Monitoring is watching known signals and alerting when they cross thresholds. Observability is being able to ask new questions about a system from its outputs. Those outputs are metrics, logs and traces. Metrics are numeric time series — cheap, good for trends and alerts. Logs are discrete event records — rich context for 'what happened'. Traces follow one request across services — good for 'where is the latency'. Metrics tell me *something* is wrong, logs tell me *what*, traces tell me *where*."
| | Metrics | Logs | Traces |
|---|---|---|---|
| Shape | Number + labels + timestamp | Text/JSON event | Spans with parent/child + timings |
| Cost | Low (aggregated) | High (volume) | Medium (sampled) |
| Best for | Alerting, capacity, SLOs | Debugging, audit, security | Latency, dependencies |
| Yours | Prometheus ✅ | Loki + kubectl 🟡 | 📝 (OpenTelemetry/Jaeger) |
**Follow-ups:** *Cardinality?* Number of unique label combinations; high-cardinality labels (user ID, request ID) explode Prometheus/Loki memory — put such data in logs/traces, not labels.

### Question: Four golden signals; RED; USE
- **Four golden signals (Google SRE):** **Latency, Traffic, Errors, Saturation.**
- **RED** (request-driven services): **R**ate, **E**rrors, **D**uration.
- **USE** (resources): **U**tilisation, **S**aturation, **E**rrors.
**Applied to your system:** RED for the Express API (`http_requests_total`, `http_request_duration_seconds` — needs `prom-client`, 📝 not present); USE for nodes/pods (CPU/memory util, throttling/OOM as saturation, restarts as errors — available via cAdvisor/kube-state-metrics ✅).

### Question: SLI, SLO, SLA; alert fatigue; anomaly detection
- **SLI** = measured indicator (e.g. % requests < 300 ms and non-5xx). **SLO** = target for it (99.5% over 30 days). **SLA** = contractual promise with consequences (credit/penalty). SLA ≤ SLO in strictness. **Error budget** = 1 − SLO.
- **Alert fatigue:** too many/low-value alerts → people ignore them. Fix: alert on **symptoms** (user impact) not causes, add `for:` durations, severity routing, grouping/inhibition, runbooks, review noisy alerts, SLO burn-rate alerts.
- **Anomaly detection:** static thresholds → `avg_over_time` baselines, `predict_linear`, standard-deviation bands (`stddev_over_time`), week-over-week comparison (`offset 1w`), ML tools (out of scope here).
```promql
# memory trending to hit limit in 1h
predict_linear(container_memory_working_set_bytes{namespace="devsecops-prod"}[30m], 3600)
  > on(pod) group_left kube_pod_container_resource_limits{resource="memory"}
```

## 6.3 Prometheus (full format)

**Question:** What is Prometheus and how does it work?

**Interview Answer:** "Prometheus is an open-source metrics system built around a multidimensional time-series database and a pull model. It periodically scrapes HTTP `/metrics` endpoints from targets, stores samples as `metric_name{labels} value @timestamp`, and lets me query with PromQL and fire alerts through Alertmanager. In my project I installed kube-prometheus-stack with Helm, so Prometheus scrapes cAdvisor, node-exporter and kube-state-metrics inside Minikube. My backend finds a project's pod and queries Prometheus's HTTP API with PromQL to show per-project CPU and memory, and Grafana shows the dashboards."

**Technical Deep Dive**
- **Components:** Prometheus server (retrieval/scrape, TSDB, HTTP API, rule evaluator), exporters, Pushgateway (short-lived jobs), Alertmanager (dedupe, group, route, silence, inhibit), service discovery.
- **Data model:** time series identified by name + label set. Types: **Counter** (monotonic, resets on restart; use `rate()`), **Gauge** (up/down), **Histogram** (bucketed → `histogram_quantile`), **Summary**.
- **Scrape:** every `scrape_interval` (15s in your standalone config; chart default ~30s) GET `/metrics` text (OpenMetrics). `up{job,instance}` = 1/0 per target.
- **TSDB:** 2-hour in-memory head block + **WAL** for crash recovery → flushed to immutable on-disk **blocks**, compacted; retention (default 15d) by time/size; local storage, not clustered (use Thanos/Mimir/Cortex/VictoriaMetrics for HA/long term).
- **Service discovery:** `kubernetes_sd_configs` (roles: node, pod, service, endpoints), relabeling to select/rename targets. kube-prometheus-stack uses the **Prometheus Operator**: `ServiceMonitor`/`PodMonitor`/`PrometheusRule` CRDs. Your `values-monitoring.yaml` sets `serviceMonitorSelectorNilUsesHelmValues: false` etc., so Prometheus picks up **all** ServiceMonitors/rules regardless of Helm release labels.
- **Pull vs push:** pull gives central control of rate, free health check (`up`), simple targets, easy debugging (curl `/metrics`); push suits short-lived jobs/NAT (Pushgateway). Cons of pull: needs network reachability to targets; short jobs may be missed.
- **Exporter vs instrumented app:** exporter translates a system without native metrics (node-exporter for OS, kube-state-metrics for K8s objects, mongodb-exporter); instrumented app exposes its own metrics via a client library (`prom-client`). Your Express backend is **not instrumented** 📝.
- **cAdvisor** (inside kubelet, `/metrics/cadvisor`) → `container_*` metrics used by your backend.

**Project Connection — the actual code**
```js
const memoryQuery = `container_memory_usage_bytes{namespace="devsecops-prod",pod="${podName}"}`;
const cpuQuery    = `rate(container_cpu_usage_seconds_total{namespace="devsecops-prod",pod="${podName}"}[1m])`;
axios.get('http://localhost:9090/api/v1/query', { params: { query: memoryQuery } });
// value = result[0].value[1]  (string) → Number()
```
Flow: React → `GET /api/monitoring/proj-5cbc4` → backend `kubectl get pods | grep` → real pod → 2 PromQL calls → JSON `{cpu, memory, podName}`.
**Known issues ⚠️:** only `result[0]` (multi-replica → should `sum by (pod)`); `container_memory_usage_bytes` includes page cache — `container_memory_working_set_bytes` is what the OOM-killer/kubelet uses; no auth/ownership check; shell interpolation (injection); depends on a live port-forward (else 500); instant query only (no history).

**Possible Follow-ups (with answers)**
1. *Why pull-based?* Prometheus controls load and timing, detects down targets via `up==0`, targets stay simple, easy to debug by curl.
2. *What happens when a target becomes unavailable?* Scrape fails → `up=0`, samples missing (staleness marker); queries show gaps; alert `up == 0 for 2m`.
3. *Exporter vs instrumented app?* See above.
4. *What happens when storage fills?* Retention deletes oldest blocks; if disk is full Prometheus can crash/corrupt WAL; set `--storage.tsdb.retention.size`, PVC sizing, alert on `prometheus_tsdb_*` and disk, use remote write for long-term.
5. *Why Grafana instead of Prometheus UI?* Prometheus UI is for ad-hoc queries/graphs; Grafana gives dashboards, variables, multiple data sources (metrics + logs), sharing, annotations, RBAC, alerting.
6. *Why `rate()` on a counter?* Counter is cumulative; per-second increase is meaningful. `irate` = last two points (spiky); `increase` = total over window. Range must be ≥ 2× scrape interval (`[1m]` with 30s scrape is borderline; use `[2m]`+).
7. *CPU `rate()` result meaning?* Cores used (0.1 = 100m) — comparable to your requests/limits.
8. *Prometheus limitations?* Single-node storage, no long-term/HA by itself, pull needs reachability, high-cardinality sensitivity, not for logs/traces, not exact (sampling), weak multi-tenancy.
9. *HA?* Run two identical replicas (each scrapes) + Alertmanager cluster; Thanos/Mimir for global view/dedup.
10. *Does your standalone `monitoring/prometheus.yml` monitor your pods?* No — it only scrapes `prometheus:9090` (itself). Real pod metrics come from the Helm Prometheus.

**Quick Revision:** pull/scrape · time series name+labels · counter/gauge/histogram · TSDB head+WAL+blocks · ServiceMonitor discovery · `rate()` on counters.

### PromQL toolbox (memorise)
```promql
up == 0                                                          # down targets
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="devsecops-prod",container!=""}[5m]))
sum by (pod) (container_memory_working_set_bytes{namespace="devsecops-prod",container!=""})
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total[5m])) / sum by (pod) (rate(container_cpu_cfs_periods_total[5m]))   # CPU throttling ratio
increase(kube_pod_container_status_restarts_total{namespace="devsecops-prod"}[15m]) > 0         # restarts
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1                       # OOM
kube_deployment_status_replicas_available / kube_deployment_spec_replicas < 1                   # missing replicas
kube_pod_status_phase{phase=~"Pending|Failed|Unknown"} > 0
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)                               # node mem %
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))                                          # node CPU %
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes                        # disk free %
sum(rate(container_network_receive_bytes_total[5m])) by (pod)                                   # net in
sum(rate(container_network_transmit_bytes_total[5m])) by (pod)                                  # net out
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))                             # top CPU pods
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))          # p95 (needs app metrics)
```

### Alerting rules (what you should be able to write)
```yaml
groups:
- name: devsecops
  rules:
  - alert: TargetDown
    expr: up == 0
    for: 2m
    labels: { severity: critical }
    annotations: { summary: "{{ $labels.job }}/{{ $labels.instance }} is down" }
  - alert: PodRestartingFrequently
    expr: increase(kube_pod_container_status_restarts_total{namespace="devsecops-prod"}[15m]) > 3
    for: 5m
    labels: { severity: warning }
  - alert: PodNearMemoryLimit
    expr: container_memory_working_set_bytes{namespace="devsecops-prod",container!=""}
          / on(pod,container) kube_pod_container_resource_limits{resource="memory"} > 0.9
    for: 10m
```
**Alert pipeline:** Prometheus evaluates rules every `evaluation_interval` → alert `pending` (during `for:`) → `firing` → sent to **Alertmanager** → group/dedupe/inhibit/silence → routes to receivers (Slack, email, PagerDuty). **Yours:** chart ships default rules and Alertmanager, but **no receivers/custom rules are configured in the repo** → nobody is notified. In kube-prometheus-stack, custom rules are `PrometheusRule` CRs.
**What happens when a monitored service goes down:** scrape fails → `up=0` → `TargetDown` pending → firing → Alertmanager → notification; if the *Prometheus itself* is down use a **dead-man's switch** (always-firing `Watchdog` alert to an external service — kube-prometheus-stack includes `Watchdog`).

## 6.4 Grafana (full format)

**Question:** What is Grafana and how does it talk to Prometheus?

**Interview Answer:** "Grafana is a visualisation and alerting layer. It doesn't store metrics; each panel holds a query that Grafana sends to a data source — in my case Prometheus for metrics and Loki for logs — and renders the result. I deployed it with kube-prometheus-stack, which ships Kubernetes dashboards, and I embedded the pod dashboard in my React page via an iframe using `var-namespace` and `var-pod` URL variables so each project sees only its own pod."

**Deep Dive:** data source (HTTP URL + type) → dashboard (JSON model: panels, variables, time range) → panel (query, transform, visualisation: time series, stat, gauge, table, logs). Backend proxy mode: browser → Grafana → data source (the data source isn't exposed to the browser). Variables (`$namespace`, `$pod`) templated into queries. **Provisioning:** data sources/dashboards as YAML/JSON in `/etc/grafana/provisioning` — reproducible; **Grafana alerting** (unified): rules on any data source, contact points, notification policies. Dashboard design: top = SLO/health stat, then RED/USE, then drill-down; consistent units/thresholds; link to logs; avoid vanity graphs.
**Your config (`values-monitoring.yaml`):**
```yaml
grafana.ini:
  security: { allow_embedding: true }
  auth.anonymous: { enabled: true, org_role: Admin }   # ⚠️
  auth: { disable_login_form: true }
  server: { root_url: "%(protocol)s://%(domain)s:%(http_port)s/grafana/", serve_from_sub_path: true }
```
`allow_embedding` removes `X-Frame-Options: deny` so the iframe works. ⚠️ Anonymous **Admin** + no login = anyone reaching the port can edit/delete dashboards, data sources, and (with Explore) query all logs/metrics. **Fix:** `org_role: Viewer`, keep login, or an auth-proxy/OIDC that maps users to their project; restrict `frame-ancestors` to your dashboard origin; `cookie_samesite` settings.
**Challenge — "Datasource not found":** imported dashboards referenced data source UIDs from another Grafana → panels fail. Fix: select your data source / set `${DS_PROMETHEUS}` input at import, or provision data sources with **fixed UIDs**.
**Follow-ups:** *How does Grafana authenticate to Prometheus?* Not at all in your setup (internal service URL). *Per-tenant dashboards?* Grafana orgs/folders + RBAC; or auth-proxy + a dashboard variable locked to the tenant's namespace — URL parameters alone are **not** a security boundary (a user can edit `var-pod`). *Grafana unavailable?* Metrics/logs still collected (Grafana is stateless viewer); fix pod/port-forward; keep dashboards provisioned in Git.

## 6.5 Loki and Promtail (full format)

**Question:** What is Loki? Loki vs Elasticsearch?

**Interview Answer:** "Loki is a horizontally-scalable log aggregation system from Grafana Labs, modelled on Prometheus. Unlike Elasticsearch it doesn't index the log text — it indexes only the labels (like `job`, `namespace`, `pod`) and stores compressed chunks of log lines. Queries pick streams by label then grep through their chunks with LogQL. That makes it much cheaper in memory and disk, which suited my Minikube-on-a-laptop setup. Promtail is the agent that tails log files, attaches labels and pushes them to Loki; Grafana queries Loki as a data source."

| | Loki | Elasticsearch (ELK) |
|---|---|---|
| Indexing | Labels only | Full-text inverted index |
| Resource use | Low | High (JVM, heap, disk) |
| Query | LogQL (label selector + filters) | Lucene/KQL, rich aggregation |
| Search speed on arbitrary text | Slower (scans chunks) | Fast |
| Ops complexity | Low | High |
| Fit | Kubernetes, label-driven, cost-sensitive | Heavy analytics, SIEM, full-text forensics |

**Deep Dive**
- **Stream** = unique label set; each stream = ordered log entries → chunks. Components (distributed mode): distributor → ingester → store (index + chunks in object storage/filesystem), querier/query-frontend, ruler, compactor. Single-binary mode (yours): all-in-one.
- **Your `loki-config.yml`:** `auth_enabled:false` (single tenant, no `X-Scope-OrgID`), `common.path_prefix:/tmp/loki`, filesystem chunks, `ring.kvstore: inmemory`, `replication_factor:1`, schema `tsdb`/`v13`, 24h index period, ruler → `alertmanager_url: http://localhost:9093`. ⚠️ Data under `/tmp` in a container **without a volume** is lost when the container is recreated; no retention configured; no auth.
- **Promtail (Compose) config:**
  ```yaml
  clients: [{ url: http://loki:3100/loki/api/v1/push }]
  scrape_configs:
  - job_name: system
    static_configs: [{ targets: [localhost], labels: { job: varlogs, __path__: /var/lib/docker/containers/*/*.log } }]
  ```
  Mounts `/var/log`, `/var/lib/docker/containers:ro`, `docker.sock`. Tracks read offsets in `positions.yaml` (`/tmp/positions.yaml` — lost on restart ⇒ possible re-ingest/duplicates).
- **Promtail (K8s DaemonSet, `k8s/promtail.yaml`):** `kubernetes_sd_configs: role: pod` + relabel `app/namespace/pod` + `docker: {}` stage. ⚠️ Placeholder Loki URL and basic-auth values, default SA without RBAC → not functional as committed. Note `docker: {}` parses Docker JSON logs; modern K8s uses containerd → use `cri: {}` stage.
- **Promtail vs alternatives:** Promtail is **deprecated in favour of Grafana Alloy**; Fluent Bit/Fluentd/Vector are common alternatives. Say: "Promtail was what I used; today I'd consider Alloy or Fluent Bit."
- **LogQL:**
  ```logql
  {job="varlogs"}                                           # all lines of a stream
  {namespace="devsecops-prod", pod=~"proj-5cbc4.*"}         # one tenant
  {app="devsecops-backend"} |= "error"                      # contains
  {app="devsecops-backend"} |~ "(?i)invalid credentials"    # regex
  {app="devsecops-backend"} | json | status >= 500          # parse JSON, filter field
  sum by (pod) (rate({namespace="devsecops-prod"} |= "error" [5m]))   # log-derived metric
  count_over_time({app="devsecops-backend"} |= "Invalid credentials" [5m])
  ```
- **Your 429 challenge:** Promtail also read host/system logs → exceeded Loki's per-tenant ingestion limits (`ingestion_rate_mb`, `ingestion_burst_size_mb`, `per_stream_rate_limit`) → HTTP **429**. 4xx = client sent too much. Fix: narrow `__path__`/add `drop` stages/relabel `keep`; or raise limits; Promtail retries with backoff, so sustained 429s mean dropped logs.
- **Dashboard Logs page ≠ Loki:** backend runs `kubectl logs <pod> -n devsecops-prod --tail=50 --timestamps=true`. Pros: zero setup, tenant-scoped; cons: last-50-lines only, current container only, no search/history, shell exec.

**Possible Follow-ups**
1. *Why Loki?* Lightweight, label model identical to Prometheus (same labels → correlate metric spike with logs), native Grafana integration, cheap storage.
2. *What if Loki stops receiving logs?* → Scenario 9.8.
3. *Labels vs log content?* Keep labels low-cardinality (app, namespace, pod, level). Don't label by user ID/IP — put in the line and filter with `|=`/`| json`.
4. *Where does log data physically live?* Chunks + index in filesystem (yours) or S3/GCS in production; Loki needs a persistent volume.
5. *How do you stop sensitive data reaching logs?* → §7.4.

**Quick Revision:** label-only index · Promtail tails + labels + pushes · LogQL = selector + filters · 429 = ingestion limit · yours: Compose pipeline works; K8s DaemonSet unfinished; dashboard logs use kubectl.

## 6.6 End-to-end monitoring data flow (answer to "how does data travel?")
1. **Metrics:** container runs → kernel cgroups count CPU/memory → **kubelet's cAdvisor** exposes `/metrics/cadvisor` → **Prometheus** scrapes (discovered via ServiceMonitor/kubelet) → stored in TSDB → queried by (a) your backend (HTTP API) and (b) Grafana panels.
2. **Cluster state:** **kube-state-metrics** watches the API server and exposes object state (`kube_pod_*`); **node-exporter** DaemonSet exposes host metrics.
3. **Logs:** app writes to stdout/stderr → container runtime writes JSON/CRI files under `/var/lib/docker/containers/<id>/<id>-json.log` (`/var/log/pods/…`) → **Promtail** tails & labels → HTTP push to **Loki** → Grafana Explore/log panels via LogQL.
4. **Alerts:** Prometheus rules → Alertmanager → receivers (not configured in repo).
5. **Per-project view:** backend maps project → pod name (kubectl) → PromQL; iframe sets `var-pod`.

**How would you monitor CPU/memory/disk/network (spoken):** "Node level with node-exporter — CPU idle rate, `MemAvailable`, filesystem avail, network bytes; container level with cAdvisor — `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, throttling and network counters; Kubernetes level with kube-state-metrics for restarts, replicas and pod phases. Then USE-style dashboards and alerts on saturation — throttling, memory near limit, disk < 15%."

## 6.7 Monitoring cross-questions — hardest ones
- *"Your backend queries `localhost:9090`. What if you run 2 backend replicas in Kubernetes?"* Port-forward is a laptop artifact; inside the cluster use the Service DNS `http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090` (+ NetworkPolicy), or better, let Grafana/Prometheus handle queries and avoid proxying raw PromQL.
- *"Can a user abuse your monitoring API?"* Yes: no auth, no ownership check, injection via `projectName`. Anyone can read any project's metrics/logs or run commands.
- *"How do you know Prometheus itself is healthy?"* `up{job="prometheus"}`, `prometheus_tsdb_head_series`, `prometheus_target_scrape_pool_sync_total`, `Watchdog` alert to an external heartbeat service.
- *"CPU 100% on the dashboard but app is fine?"* Compare to **request** vs **limit**; check **throttling ratio**; `rate()` window; 1.0 = 1 core, not 100%.
- *"Why `container!=""`?"* cAdvisor emits pod-level aggregate series with empty `container`; excluding avoids double counting.
- *"Metric says memory 240Mi of 256Mi but no OOM?"* `usage_bytes` includes reclaimable cache; check `working_set_bytes`.

---

# PART 7 — Security monitoring and incident response

## 7.1 Concepts (compact)
| Topic | Answer |
|---|---|
| **SIEM** | Collects, normalises, correlates and alerts on security logs from many sources (Splunk, Elastic Security, Wazuh, Sentinel). Prometheus/Loki/Grafana is an **observability** stack, not a SIEM: no correlation engine, case management, threat intel, long immutable retention. They can feed one. |
| **Audit logs** | Who did what when: K8s API audit log, Jenkins audit trail, Vault audit device, cloud trail, app auth logs. Must be tamper-resistant, time-synced, retained. |
| **Auth-failure monitoring** | Count failed logins per IP/user/time; alert on bursts; correlate with later success (credential stuffing). |
| **Brute-force detection** | rate of failures per source; lockout/captcha/rate limit; alert + auto-block (WAF/fail2ban). |
| **MTTD / MTTR** | Mean time to **detect** / **respond (or recover)**; reduce with good alerts, runbooks, automation. (MTTA = acknowledge; MTBF = between failures.) |
| **Severity (SEV1–4)** | By impact × scope: SEV1 = data breach/prod down; SEV2 = major degradation; SEV3 = minor; SEV4 = cosmetic. Prioritise by business impact + exploitability. |
| **RCA / blameless postmortem** | Timeline, root cause (5 Whys), contributing factors, what went well, action items with owners. |
| **IR phases (NIST 800-61)** | Preparation → Detection & Analysis → Containment, Eradication, Recovery → Post-incident. |
| **Escalation** | On-call L1 → L2 → incident commander; page by severity; comms lead; documented runbooks. |

## 7.2 What your project has vs production
| Capability | Your project | Production-grade |
|---|---|---|
| Metrics collection | ✅ Prometheus | + app RED metrics, SLO burn alerts |
| Central logs | 🟡 Loki (Compose); K8s DaemonSet unfinished | All pods/nodes + audit logs, retention, HA, object storage |
| Auth failure visibility | ⚠️ only morgan `401` line | Structured auth events, counters, alerts |
| Alerting | 📝 none configured | Alertmanager → PagerDuty/Slack, runbooks |
| Audit logs | 📝 none (K8s audit, Vault audit, Jenkins audit) | Enabled, shipped to immutable store |
| Runtime detection | 📝 | Falco, eBPF |
| SIEM/correlation | 📝 | Wazuh/Elastic/Splunk |
| IR process | 📝 | Runbooks, on-call, postmortems |

## 7.3 Detection recipes (say these with confidence)
- **Brute-force login (needs structured logs):**
  ```logql
  sum by (ip) (count_over_time({app="devsecops-backend"} |= "Invalid credentials" | json | __error__="" [5m])) > 10
  ```
  Or Prometheus `rate(login_failures_total[5m]) > 0.5`. Response: rate-limit (`express-rate-limit`), lockout, captcha, block IP, check for later `200` from same IP.
- **403 spike (expired/forged tokens):** `sum(count_over_time({app="devsecops-backend"} |~ "\\s403\\s"[5m]))` (morgan format) — in your logs `GET /api/stats 403` appears.
- **Suspicious shell in a container (Falco):** rule "Terminal shell in container" → alert.
- **Unexpected outbound traffic:** `rate(container_network_transmit_bytes_total[5m])` anomaly vs baseline + NetworkPolicy egress deny + flow logs (Cilium Hubble).
- **Crypto-mining / compromised container:** sustained CPU at the **limit** + high throttling + unknown process + outbound to unusual IP.
- **Unauthorized K8s access:** API audit log `verb=create,resource=pods/exec`, `Forbidden` events; alert on `exec`, secret reads, new ClusterRoleBindings.
- **Pipeline tampering:** Jenkinsfile changes (Git audit), new credentials, new plugins, builds from unexpected branches.

## 7.4 Preventing sensitive data in logs
1. **Don't log it:** never log passwords/tokens/PII (`console.log(req.body)` on login is a leak; your Jenkins `echo` of the token is one).
2. **Redact at source:** logging library with redact paths (`pino`'s `redact`), mask in morgan custom tokens (don't log `Authorization`/query tokens — note your Jenkins call puts `token=` in the **URL query**, which appears in access logs).
3. **Redact in pipeline:** Promtail `replace` stage / Fluent Bit filters regex-mask emails, tokens, card numbers.
4. **Jenkins:** `withCredentials` masking, avoid `set -x`, no secrets in `echo`/env dumps.
5. **Control access** to logs (RBAC), retention limits, encryption.
6. **Scan:** gitleaks/trufflehog in CI; periodic log-secret scans.
**Promtail redaction stage example:**
```yaml
pipeline_stages:
- replace: { expression: '(?i)(authorization|token|password)=([^&\s]+)', replace: '$1=***' }
```

## 7.5 Correlation during an incident (spoken)
"I anchor on time. A Grafana panel shows CPU/latency/restarts jump at 10:42. I use the same labels — namespace and pod — to jump to Loki and read the logs in that window; that's why consistent labels across metrics and logs matter. I check `kubectl describe pod` events and the last deployment (`kubectl rollout history`, Jenkins build number) to see if a release coincided. I then form a hypothesis — bad release, traffic spike, dependency down, attack — and test it with a targeted query, mitigate (rollback/scale/block), and write the RCA."

---

# PART 8 — Architecture and end-to-end workflow (using YOUR actual sequence)

> The generic "GitHub webhook → Jenkins" story does **not** match your code. Here is your real sequence, then how it differs from the textbook one.

## 8.1 Your real flow
```
[1] Browser POST /api/projects (JWT)                      ── trust boundary: Internet/LAN → backend
[2] authenticateToken → req.user
[3] MongoDB: project {status:"Building", port, owner}
[4] Backend: name=proj-<last5 of _id>; fill template; kubectl apply -n devsecops-prod
[5] K8s: Deployment+RS+Pod (nginxdemos/hello), NodePort Service
[6] Backend: kubectl get svc -o json → nodePort; minikube ip → liveUrl → MongoDB
[7] Backend → Jenkins buildWithParameters(PROJECT_ID) ; replies 201 immediately   ── boundary: backend → Jenkins (basic auth + job token)
[8] Jenkins: Checkout → Build Frontend → Security Scan(Vault) → Containerize(push) → Trivy → Deploy
[9] Jenkins → Backend webhook (no auth)                    ── boundary: container → host
[10] Backend updates MongoDB (status, report URLs, hardcoded scan numbers)
[11] Dashboard shows status/URL/report; /api/monitoring → PromQL; /api/logs → kubectl logs
```
**Failure points & what you'd see**
| Point | Failure | Symptom |
|---|---|---|
| 2 | expired JWT | 403, dashboard calls fail |
| 4 | wrong kubeconfig/Minikube down | `IaC Provisioning failed`, no pod |
| 6 | read before nodePort assigned | undefined URL (your challenge) |
| 7 | Jenkins down / bad token | project stuck "Building" (error only logged) |
| 8 | Vault empty / kubeconfig stale / Docker Hub auth | UNSTABLE / FAILURE |
| 9 | backend unreachable (`host.docker.internal`) | status never updates |
| 11 | port-forward died | monitoring 500 |

**Security boundaries:** internet/LAN ↔ backend (JWT, helmet; weak: open CORS, no rate limit) · backend ↔ cluster (kubeconfig; weak: shell exec) · Jenkins ↔ host (docker.sock = root) · Jenkins ↔ Vault (root token) · tenant ↔ tenant (labels only; no NetworkPolicy/RBAC) · Jenkins → backend (none) · Grafana (anon Admin).

## 8.2 Textbook sequence vs yours
| Step (generic) | Yours |
|---|---|
| 1 Developer commits | ✅ (code in Git) |
| 2 GitHub webhook → Jenkins | ❌ **backend** triggers Jenkins via API |
| 3–4 Build/test | Frontend build only; **no test stage** |
| 5 Security checks | Vault fetch (Sonar placeholder) |
| 6 Docker image | ✅ |
| 7 Scan & validate | Trivy after push, non-blocking |
| 8 Push | ✅ (before scan) |
| 9 Deploy | ✅ `kubectl apply` |
| 10–11 Metrics/logs/Grafana | ✅ (Compose Loki; dashboard uses kubectl logs) |
| 12 Alerts/IR | ❌ not configured |

## 8.3 Q&A: "Walk me through the architecture / why these choices?"
**Interview Answer:** "There are four planes. A **control plane** — React and Express with MongoDB — is the user interface and orchestrator. A **delivery plane** — GitHub, Jenkins, Docker Hub — builds and ships. A **runtime plane** — Kubernetes on Minikube — runs the workloads. And an **observability/security plane** — Vault, SonarCloud, Trivy, Prometheus, Grafana, Loki. The backend is the hub: it provisions Kubernetes, triggers Jenkins, receives its webhook and proxies metrics and logs. I made some local-environment compromises — NodePort instead of Ingress, Vault in dev mode, one namespace — and I know the production changes: Ingress with TLS, namespace-per-tenant with NetworkPolicies and quotas, Vault server mode with AppRole, Trivy and Sonar as hard gates, and Alertmanager."
**Follow-ups:** *Why a Node backend shelling out to kubectl?* Fast to prototype; production → official `@kubernetes/client-node`, Operators/CRDs, or GitOps (Argo CD) where the backend just commits manifests. *Why MongoDB?* Document-shaped metadata (nested `sonarResults`), flexible schema; relational would also work. *Single point of failure?* Backend host process, Jenkins container, single-node Minikube, Vault in RAM. *Scalability?* Port-forward and kubectl exec don't scale; use in-cluster services, async job queue, controller pattern.

## 8.4 Target production architecture
```
Dev → GitHub (branch protection, CODEOWNERS) ──webhook(HMAC)──▶ CI (ephemeral agents)
   SAST (Sonar gate) + SCA + secret scan ─▶ build (Kaniko) ─▶ Trivy gate ─▶ sign (cosign) + SBOM
   ─▶ registry (immutable tags) ─▶ GitOps repo ─▶ Argo CD ─▶ K8s (namespace/tenant, NetworkPolicy, PSA restricted,
      Kyverno verify-signature, ResourceQuota) ; Ingress+TLS ; Vault via CSI/agent
   Observability: Prometheus(+Thanos) / Loki(S3) / Tempo(OTel) / Grafana(SSO) / Alertmanager → PagerDuty ; Falco ; audit logs → SIEM
```

---

# PART 9 — Scenario-based questions (problem → investigate → commands → root cause → mitigate → prevent → spoken answer)

## 9.1 A Docker image contains a critical CVE
- **Problem:** Trivy reports CRITICAL in `backend` image.
- **Investigate:** which package/layer (OS vs npm)? fixed version? reachable/exposed? exploit known (KEV/EPSS)? deployed where?
- **Commands:** `trivy image --severity CRITICAL --ignore-unfixed IMG` · `docker history IMG` · `npm ls <pkg>` · `kubectl get pods -A -o jsonpath='{..image}'` (where is it running).
- **Root cause:** old base image / transitive dependency.
- **Mitigate:** rebuild on patched base (`node:20-alpine` newer), `npm update <pkg>`/override, redeploy by new tag; if no fix: mitigate/accept with expiry.
- **Prevent:** Trivy gate before push; scheduled rebuilds; Renovate/Dependabot; registry rescans.
- **Spoken:** "I'd treat it as a blocker if it's fixable and reachable. I'd identify the package and layer, patch by updating the base image or dependency, rebuild, rescan, and redeploy a new immutable tag. To prevent it recurring I'd gate the pipeline with `--exit-code 1` before the push — which my current pipeline doesn't do yet."

## 9.2 Jenkins accidentally exposes a secret in logs
(You actually have one: `echo "… Token: ${SONAR_TOKEN}"`.)
- **Investigate:** which build(s), who can view logs, was it masked, was it archived/emailed/forwarded to Loki?
- **Mitigate (in order):** **rotate/revoke the secret immediately** (assume compromised), delete/redact build logs, check Vault/SonarCloud audit for use, remove the `echo`.
- **Commands:** Jenkins UI → Delete build log; `grep -r` in `$JENKINS_HOME/jobs/*/builds/*/log`; Sonar: revoke token; Vault: `vault kv put` new value.
- **Prevent:** secret scanning of logs, `withCredentials`/`withVault` masking, single-quoted `sh`, no `set -x`, code review of Jenkinsfile.
- **Spoken:** "Deleting the log isn't enough. First rotate the secret, then clean the log, then audit usage and fix the step."

## 9.3 A developer pushes vulnerable dependencies
- **Detect:** `npm audit`/Trivy fs/Dependabot/Sonar SCA in PR; **Fix:** upgrade/override, lockfile; **Gate:** block PR on high/critical; **Prevent:** Renovate, `npm ci`, allow-list registry, SBOM.
- **Commands:** `npm audit --omit=dev --audit-level=high` · `trivy fs --scanners vuln .`

## 9.4 A container runs as root
- **Detect:** `docker inspect -f '{{.Config.User}}' c` / `kubectl exec p -- id` / Trivy config / kube-linter / Pod Security Admission.
- **Impact:** root in container + any misconfig (hostPath, caps, privileged, docker.sock) ⇒ host compromise; writable FS.
- **Fix:** `USER node`; `securityContext: runAsNonRoot, runAsUser, readOnlyRootFilesystem, drop ALL`; enforce PSA `restricted`/Kyverno.
- **Yours:** backend/frontend images run as root (nginx master), Jenkins is root+privileged.

## 9.5 An attacker compromises a Jenkins agent (or Jenkins)
- **Impact in YOUR setup:** can read mounted kubeconfig (cluster access), Docker Hub creds, Vault root token, use docker.sock to take over the host, push malicious images.
- **Investigate:** build history/who triggered, Jenkins audit log, unusual outbound traffic, new credentials/jobs/plugins, Docker Hub push history/tags, K8s audit (`kubectl get events`, new deployments).
- **Contain:** isolate/stop the Jenkins container, revoke Docker Hub token, rotate Vault/kubeconfig creds, revoke Sonar token, review/redeploy images from known-good digests.
- **Prevent:** ephemeral agents, least-privilege creds per job, no docker.sock, separate controller, egress restrictions, signed images + admission verification, Vault AppRole.

## 9.6 Deployment succeeds but monitoring shows abnormal behaviour
- **Investigate:** time of anomaly vs build number (`kubectl rollout history`); restarts (`increase(kube_pod_container_status_restarts_total[15m])`), memory/CPU vs limits, throttling, error logs (`logs --previous`, Loki), dependency health (Mongo), HPA events.
- **Mitigate:** `kubectl rollout undo deploy/devsecops-backend`; scale; fix config.
- **Prevent:** readiness probes + `rollout status` gate, canary, SLO alerts, smoke test stage.
- **Why your pipeline reports "Deployed" wrongly:** it doesn't verify rollout.

## 9.7 Prometheus reports missing targets
- **Investigate:** Status → Targets page; `up == 0`; error text (connection refused, 403, timeout, TLS); ServiceMonitor selector/labels vs Service labels; port **name** match; namespace selector; RBAC for discovery; NetworkPolicy; target pod healthy.
- **Commands:** `kubectl get servicemonitor -A` · `kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090` → /targets · `kubectl describe servicemonitor` · `curl pod:port/metrics`.
- **Common root causes:** label mismatch (Helm release label) — your values set `serviceMonitorSelectorNilUsesHelmValues:false` to avoid it; wrong port name; scrape path; auth required; pod crashed.
- **Spoken:** "I check `/targets` for the error, then verify the label/port selectors between ServiceMonitor and Service, then reachability with curl from inside the cluster."

## 9.8 Loki receives logs but Grafana cannot display them / Loki stops receiving
- **Grafana can't display:** data source URL/health ("Save & test"), UID mismatch ("Datasource not found"), time range, wrong label selector (`{job="varlogs"}` vs actual labels via label browser), query timeout/limits, query-frontend errors, Grafana→Loki network, tenant header (`X-Scope-OrgID`) if auth enabled.
- **Loki not receiving:** Promtail status (`:9080/targets`, `/metrics` `promtail_sent_entries_total`, `promtail_dropped_entries_total`), 429 rate limits, wrong URL, positions file, path/glob doesn't match (containerd vs docker), permissions on log dir, Loki disk full/OOM, clock skew (entries too old rejected).
- **Commands:** `docker logs promtail` · `curl loki:3100/ready` · `curl 'loki:3100/loki/api/v1/labels'` · `docker logs loki | grep -i 429`.
- **Detect silently-lost logs:** alert on `rate(promtail_sent_entries_total[5m]) == 0` and on `{app="x"}` absent: `absent_over_time({app="devsecops-backend"}[10m])`.

## 9.9 Sudden traffic spike
- **Investigate:** HPA (`kubectl get hpa -w`, `describe hpa`), CPU vs request, replicas, pending pods (node capacity), latency/errors, source IPs (legit vs DoS).
- **Reality check for you:** HPA for the platform only (2–5, 70% CPU); node is a single Minikube; per-project pods are 1 replica with no HPA; no rate limit.
- **Mitigate:** scale out, raise limits, rate-limit/WAF, cache; **Prevent:** load test, capacity planning, cluster autoscaler, queueing, circuit breakers.

## 9.10 The monitoring system itself becomes unavailable
- **Principles:** monitor the monitors; **dead-man's switch** (`Watchdog` alert → external heartbeat like healthchecks.io); run Prometheus HA + Alertmanager cluster; remote-write to external store; keep a minimal out-of-band check (external uptime probe on the app); alert on Prometheus-specific metrics (`prometheus_tsdb_*`, `up{job="prometheus"}`).
- **Yours:** Prometheus/Grafana depend on a laptop port-forward; if it drops, your backend returns 500 and the iframe goes blank; no external watchdog.

## 9.11 A security scan generates false positives
- **Triage** with evidence (reachability, config), document reason, suppress **narrowly** (`.trivyignore` with CVE + expiry; Sonar "False positive" with comment; `// NOSONAR` only with justification), tune rules, review periodically, track FP rate.
- **Never:** disable the gate globally. Answer for "FP vs FN": FP wastes time; FN is dangerous.

## 9.12 Pipeline keeps failing due to security gates
- **Investigate:** which finding, is it new or legacy, is it fixable (`--ignore-unfixed`), is the gate too strict (`LOW`/`MEDIUM`)?
- **Resolve:** fix root causes; gate on **new code**/critical-high-with-fix; baseline legacy debt with a tracked remediation plan; time-boxed exception with approval; automate dependency updates.
- **Don't:** convert the gate to `catchError` (which is what your Sonar stage does today) — that's removing the control.
- **Spoken:** "A gate that always fails is either mis-scoped or a sign of real debt. I'd scope it to what's actionable and block on that, and manage the rest through tracked SLAs rather than turning the gate off."

## 9.13 Repeated unauthorized login attempts
- **Detect:** failed-login counters/logs by IP and account; many 401 from one IP; later success.
- **Respond:** rate limit/lockout/captcha, block IP at ingress/WAF, force reset on compromised account, notify user, check for credential stuffing across accounts.
- **Build:** `express-rate-limit` + structured log `{event:"login_failed", ip, email}` + LogQL/Prometheus alert. (⚠️ today only morgan access line.)

## 9.14 Investigate a sudden CPU spike
1. Confirm scope (`up`, which pod/node): `topk(5, sum by(pod)(rate(container_cpu_usage_seconds_total[5m])))`.
2. Compare with request/limit and **throttling** ratio.
3. Correlate with deploys, traffic (RED), logs in the same window.
4. Inside the pod: `kubectl exec -it p -- top` / `kubectl debug`; look for unknown processes (miner) vs legit load / infinite loop.
5. Mitigate (rollback, scale, kill pod, isolate with NetworkPolicy), RCA.

## 9.15 A compromised container — how to detect and respond
- **Signals:** unexpected shell (`kubectl exec` audit/Falco), outbound connections to unknown IPs, CPU pegged at limit, new files in `/tmp`, changed binaries, image digest ≠ expected, privilege use.
- **Respond:** **don't delete first** — isolate (label + deny-all NetworkPolicy), snapshot/preserve evidence (logs, `kubectl cp`, node forensics), then rotate secrets it could read (mounted Secrets, SA token), kill & redeploy from verified digest, hunt for lateral movement, RCA.

## 9.16 Investigating a failed deployment
`kubectl get pods` (status) → `describe pod` (events: `FailedScheduling`, `ErrImagePull`, probe failures) → `logs --previous` → Jenkins console (which stage; Vault empty? kubeconfig stale? docker login?) → `kubectl rollout status/history` → `kubectl get events --sort-by=.lastTimestamp` → check namespace (`-n`), image tag exists on Docker Hub, requests vs node capacity.

---

# PART 10 — Final revision cheat sheet (30–45 min)

## 10.1 30-second pitch
"I built a multi-tenant DevSecOps platform. A user logs into a React dashboard and creates a project; my Node/Express backend provisions a Kubernetes Deployment and NodePort Service for it, triggers a Jenkins pipeline that pulls secrets from Vault, builds Docker images, scans them with Trivy and deploys to Kubernetes, and the dashboard shows status, live URL, security report, CPU/memory from Prometheus and logs. My focus was security — Vault, SonarCloud, Trivy — and monitoring with Prometheus, Grafana and Loki."

## 10.2 1-minute version
Add: JWT+bcrypt auth, per-project labels/limits, webhook back to the dashboard, Prometheus queried by PromQL per pod, Grafana embedded, Loki+Promtail logs; one challenge (ClusterIP→NodePort); one next step (namespace-per-tenant + real gates).

## 10.3 My role (fill in) — security: Vault integration, SonarCloud/Trivy stages, reports on dashboard; monitoring: Helm stack, PromQL API, Grafana embedding, Loki/Promtail.

## 10.4 Architecture in one breath
React → Express (hub) → MongoDB / kubectl→Minikube / Jenkins / Prometheus. Jenkins → Vault, Docker (sock), Docker Hub, kubectl, → backend webhook. Prometheus ← cAdvisor/node-exporter/kube-state-metrics. Promtail → Loki ← Grafana (+Prometheus).

## 10.5 Pipeline flow
`Checkout → Build Frontend → Security Scan(Vault) → Containerize(push) → Trivy(report) → Deploy → post: webhook`

## 10.6 Tools and purpose
| Security | Purpose | Monitoring | Purpose |
|---|---|---|---|
| Vault | secrets | Prometheus | metrics (pull, TSDB, PromQL) |
| SonarCloud | SAST (planned) | Grafana | dashboards (viz only) |
| Trivy | image CVEs | Loki | log store (label index) |
| JWT/bcrypt | auth | Promtail | log shipper (push) |
| limits/requests | blast radius | node-exporter / kube-state-metrics / cAdvisor | node / object / container metrics |
| helmet | HTTP headers | Alertmanager | alert routing (unconfigured) |

## 10.7 Jenkins essentials
controller/agent · executors · JENKINS_HOME · declarative vs scripted · `post` · `withCredentials`/`withVault` · `catchError` → UNSTABLE · SUCCESS/UNSTABLE/FAILURE · buildWithParameters · webhook vs polling · CSRF crumb · RBAC (Role Strategy) · ephemeral agents.

## 10.8 Docker commands
`docker build -t img:tag .` · `docker run -d -p 8081:8080 img` · `docker ps -a` · `docker logs -f` · `docker exec -it c sh` · `docker inspect` · `docker history` · `docker volume ls` · `docker compose up -d|down|logs` · `docker push` · `docker run --read-only --cap-drop ALL -u 1000 img`.

## 10.9 Kubernetes commands
`get pods,svc,deploy,hpa,ep -n NS` · `describe` · `logs [--previous]` · `apply -f` · `rollout status|undo` · `get events --sort-by` · `top` · `port-forward` · `auth can-i` · `minikube ip|service|addons`.

## 10.10 PromQL / LogQL top picks
PromQL: `up==0` · `rate(container_cpu_usage_seconds_total{…}[5m])` · `container_memory_working_set_bytes` · `increase(kube_pod_container_status_restarts_total[15m])` · `kube_deployment_status_replicas_available` · `predict_linear` · `topk`.
LogQL: `{app="x"} |= "error"` · `{app="x"} | json | status>=500` · `rate({…} |= "error"[5m])` · `absent_over_time({app="x"}[10m])`.

## 10.11 Concept comparisons
- **CVE vs CVSS:** ID vs severity score (0–10).
- **SAST / DAST / SCA:** code / running app / dependencies.
- **Metrics / logs / traces:** numbers / events / request paths.
- **Prometheus / Grafana / Loki:** collect+store metrics / visualise / store logs.
- **Counter / gauge:** only up (use `rate`) / up-down.
- **Liveness / readiness:** restart / stop traffic.
- **Request / limit:** scheduling guarantee / cap (CPU throttle, memory OOM).
- **ClusterIP / NodePort / LoadBalancer / Ingress:** internal / node port / cloud LB / L7 routing.
- **Image / container; volume / bind mount; RUN / CMD.**
- **Hashing / encryption; authentication / authorisation; 401 / 403.**

## 10.12 Common vulnerabilities (with your example)
Command injection (`execSync` + `projectName`) · broken access control (unauth monitoring/logs/webhook) · weak secrets (default JWT secret) · misconfig (Grafana anon Admin, Jenkins privileged/CSRF off, root containers) · vulnerable components (`:latest`, no gate) · logging failures (no auth-failure alerts, token echoed) · supply chain (unsigned images, docker.sock).

## 10.13 Troubleshooting quick map
| Symptom | First checks |
|---|---|
| Live URL won't open | Service type, endpoints, namespace, `minikube ip`, nodePort |
| Pod `Pending` | `describe` → resources/taints |
| `ImagePullBackOff` | tag exists? pushed? creds? |
| `CrashLoopBackOff` | `logs --previous` |
| `OOMKilled` | memory limit, `working_set` |
| Jenkins deploy fails | kubeconfig/IP/cert stale |
| Empty `SONAR_TOKEN` | Vault path/key, dev-mode wipe |
| Grafana "Datasource not found" | UID mismatch |
| Loki 429 | Promtail volume vs ingestion limits |
| Monitoring 500 | port-forward, Prometheus up, pod name, JWT |
| HPA `<unknown>` | metrics-server, CPU requests |

## 10.14 Top 30 most probable project questions
1. Explain your project. 2. Your role? 3. Architecture and why each tool? 4. What happens on "Create Project"? 5. Walk through the Jenkinsfile. 6. How do you manage secrets? 7. Why Vault vs Jenkins credentials? 8. Dev vs prod Vault? 9. What does SonarCloud do; how integrated? 10. What does Trivy do; how integrated? 11. What happens if Trivy finds a critical CVE? 12. How is multi-tenancy achieved? 13. Why NodePort? 14. Deployment vs Pod vs ReplicaSet. 15. Requests vs limits. 16. How does HPA work? 17. How does monitoring work end-to-end? 18. Why Prometheus pull? 19. What do Grafana/Loki/Promtail do? 20. Loki vs ELK. 21. How does the backend get per-project metrics? 22. How is auth implemented (JWT/bcrypt)? 23. How does Jenkins report back? 24. Biggest challenge? 25. Minikube kubeconfig issue. 26. What would you improve? 27. Security weaknesses of your project? 28. How would you add alerting? 29. How would you scale this? 30. Docker volumes and the Trivy report bridge.

## 10.15 Top 20 hard cross-questions
1. Your Sonar stage only echoes — so is SAST implemented? (No; token plumbing only; here's the fix.) 2. Why does Trivy run after push? 3. Why `catchError`→UNSTABLE for a security stage? 4. Prove tenants are isolated. 5. Command injection in `/api/logs`? 6. Webhook forgery? 7. docker.sock risk? 8. Why root Vault token? 9. `latest` tag consequences? 10. Race in port allocation? 11. nodePort read timing? 12. Prometheus storage full? 13. Prometheus HA? 14. Loki without a volume? 15. Cardinality risks? 16. Metrics vs working_set memory? 17. How to monitor Prometheus itself? 18. Grafana anonymous Admin + iframe risk (clickjacking/XSS)? 19. JWT revocation? 20. How do you keep secrets out of logs?

## 10.16 Top 15 scenario questions — see Part 9 (9.1–9.16).

## 10.17 Biggest challenges (STAR one-liners)
1. ClusterIP→port-forward→NodePort. 2. Stale kubeconfig in Jenkins after Minikube restart. 3. Vault key mismatch. 4. Vault dev-mode memory. 5. Identical Sonar results (one project key). 6. Random pod names vs PromQL. 7. Grafana datasource UID. 8. Loki 429. 9. nodePort read too early. 10. YAML applied to wrong namespace. 11. Expired JWT misdiagnosed as infra problem.
**Theme:** each tool was healthy; integration (networking, naming, config sync, timing, auth) failed.

## 10.18 Limitations and improvements
Real scans + quality gate · Trivy before push with exit code · user-repo build · namespace-per-tenant + NetworkPolicy/Quota/RBAC · Ingress+TLS · probes + immutable tags · Vault server mode + AppRole · webhook HMAC · authenticate/validate monitoring+logs routes, drop shell exec · Alertmanager + SLO alerts · Loki persistence + K8s Promtail/Alloy with RBAC · Grafana Viewer/SSO · rotate leaked secrets.

---

# PART 11 — Mock interview (45–60 min)

> Read the interviewer's question, answer aloud, then compare. Difficulty rises each round.

## Round 1 — Introduction & architecture (8 min) · *evaluating: clarity, ownership, big picture*
**Q1. Introduce your project and your role.** → Use 30-sec pitch + role.
**Q2. Draw the architecture.** → Four planes (control/delivery/runtime/observability), backend as hub.
**Q3. What happens when I click Create Project?** → Priya story, 11 steps.
*Follow-up:* **Why does the backend not wait for Jenkins?** Async: replies 201 immediately, Jenkins reports via webhook; long builds shouldn't block HTTP.
*Follow-up:* **Which parts are real vs simulated?** Honest table (nginx demo workload; hardcoded scan numbers; Sonar placeholder).
*Evaluating:* honesty about scope. **Red flag:** claiming per-user namespaces or real Sonar results.

## Round 2 — DevSecOps & CI/CD fundamentals (8 min) · *evaluating: concepts + application*
**Q4. What is DevSecOps and shift-left?** **Q5. CI vs CD vs CD?** **Q6. Declarative vs scripted?** **Q7. Controller vs agent?**
*Follow-up:* **Is your pipeline shifted left? What would you move?** Trivy before push; Sonar PR checks; secret scan pre-commit.
*Follow-up:* **A scan finds a critical CVE — what should happen?** Fail build, patch, rescan, redeploy.
*Follow-up:* **Where's your test stage?** None — only frontend build. Add unit/integration tests; coverage in Sonar gate.

## Round 3 — Security implementation (12 min) · *evaluating: depth, internals, self-awareness*
**Q8. How is Vault used? What happens in `withVault`?** (path/key/KV v2/env var/masking.)
**Q9. Dev vs production Vault; root token problem; AppRole + policy.**
**Q10. What does SonarCloud do internally? What does your pipeline do with it?** (AST/taint; token fetched; scanner missing; fix snippet.)
**Q11. How does Trivy detect vulnerabilities? Why before push? Show the gate command.**
*Hard follow-ups:* **Your Jenkins echoes the secret — what's wrong?** **docker.sock?** **Is `/api/logs/:projectName` safe?** (Command injection → fix.) **How would you secure the webhook?** (HMAC.) **How do you stop a bad image reaching prod?** (Gate + signing + admission policy.)
*Evaluating:* can you find your own flaws? Strong candidates volunteer them with fixes.

## Round 4 — Monitoring & observability (12 min) · *evaluating: internals + troubleshooting*
**Q12. Metrics vs logs vs traces.** **Q13. How does Prometheus collect data? Pull vs push.**
**Q14. Show the PromQL your backend uses and explain `rate`.** (counter vs gauge; cores meaning.)
**Q15. Loki vs Elasticsearch; how does Promtail work?**
**Q16. How does data flow from a pod to a Grafana panel?** (cAdvisor → Prometheus → Grafana; stdout → Promtail → Loki → Grafana.)
*Hard follow-ups:* **Prometheus stops scraping — what do you do?** **Loki gets logs but Grafana shows none?** **How would you alert on brute force?** **Is Grafana anonymous Admin okay?** **How would you monitor Prometheus itself?** **SLI/SLO for your API?** **What did "429 Too Many Requests" mean?**
*Evaluating:* depth beyond tool names; can you debug each hop.

## Round 5 — Docker / Kubernetes / infra (10 min) · *evaluating: fundamentals + tradeoffs*
**Q17. Image vs container; layers; why `package.json` copied first?** **Q18. Volumes vs bind mounts — how does the Trivy report reach the dashboard?** **Q19. Deployment vs ReplicaSet vs Pod; what does `kubectl apply` do internally?** **Q20. NodePort vs LoadBalancer vs Ingress.** **Q21. Requests/limits; what happens on OOM; how does HPA decide?**
*Hard follow-ups:* **Are your tenants isolated?** (No: labels+limits only; NetworkPolicy/RBAC/Quota.) **No probes — consequence?** **Why is `:latest` bad?** **`insecure-skip-tls-verify`?** **Pod Security Standards for your pods?** (Would fail `restricted`: root, no securityContext.)

## Round 6 — Advanced troubleshooting & scenarios (8 min) · *evaluating: structured thinking under pressure*
**S1.** Dashboard shows "Deployed" but the live URL doesn't open. → pod status, Service type/endpoints, namespace, minikube IP/nodePort, port-forward.
**S2.** Jenkins secret leaked in a log. → rotate first, then clean, audit, fix step.
**S3.** CPU spike at 3 a.m. → scope, throttling, correlate deploy/traffic/logs, rollback/scale, RCA.
**S4.** A Jenkins agent is compromised. → contain, rotate, audit, rebuild from known-good, ephemeral agents/least privilege.
**S5.** Monitoring itself is down. → watchdog, HA, external probe.
**Closing Q:** "What would you do differently in production?" → pick 4: Ingress+TLS, namespace-per-tenant with NetworkPolicy/RBAC/Quota, hard security gates with signing, Alertmanager + SLO alerts.
*Evaluating:* method (hypothesis → query → mitigation → prevention), calm communication, ownership.

### Scoring yourself
| Score | Meaning |
|---|---|
| Each answer includes definition + **your project example** + a **trade-off or gap** | strong |
| You volunteer weaknesses with the fix before being asked | exceptional |
| You claim something not in the repo | ❌ fix immediately |

---

# Appendix A — Pre-interview checklist
- [ ] Rotate: SonarCloud token, Jenkins API token, Vault root token, JWT secret; remove from repo/PDF.
- [ ] Fill in the "My contribution" table (§0.5).
- [ ] Be able to say, without notes: Priya story · six stages · Vault path/key · Trivy command · PromQL pair · Loki flow · ClusterIP→NodePort story · top 3 weaknesses + fixes.
- [ ] Decide: can you honestly run the demo? (Minikube, port-forwards for 9090/3000, backend, frontend, compose up.)
- [ ] Quick wins you could implement and mention: Trivy `--exit-code 1` before push; real `sonar-scanner` + `qualitygate.wait`; webhook HMAC; auth on monitoring/logs; remove token `echo`; `.gitignore` `.env`; `USER node`; probes.

# Appendix B — Items I could not verify from the repo
- Whether SonarCloud analysis was ever run via a different Jenkins job configuration (the repo's Jenkinsfile doesn't run it).
- Jenkins security settings (realm/authorization) inside the running instance.
- Whether `monitoring/` Compose stack was used alongside the Helm stack in your demo.
- Who authored which component (see §0.5).
- Exact Grafana dashboard JSON used (not in repo).
