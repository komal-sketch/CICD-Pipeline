# GitLab DevOps (Practical Guide)
---

## 📚 Content
- [1. What is GitLab?](#1-what-is-gitlab)
- [2. GitLab vs GitHub vs Bitbucket — Differences & Use Cases](#2-gitlab-vs-github-vs-bitbucket--differences--use-cases)
- [3. Groups & Projects Explained](#3-groups--projects-explained)
- [4. Using GitLab with AI (Code Suggestions, Security, Automation)](#4-using-gitlab-with-ai-code-suggestions-security-automation)
- [5. GitLab Duo — Why & How + Use Cases](#5-gitlab-duo--why--how--use-cases)
- [6. Authentication: “There are no passwords” — SSH, PATs, SSO/SAML](#6-authentication-there-are-no-passwords--ssh-pats-ssosaml)
- [7. Auto DevOps (Zero-to-Prod Pipelines)](#7-auto-devops-zero-to-prod-pipelines)
- [8. Creating & Managing Projects/Groups (RBAC & Governance)](#8-creating--managing-projectsgroups-rbac--governance)
- [9. Creating Pipelines in GitLab (Why Jenkins isn’t required)](#9-creating-pipelines-in-gitlab-why-jenkins-isnt-required)
- [10. File Naming Conventions](#10-file-naming-conventions)
- [11. How GitLab CI Works (From Push → Runner → Status)](#11-how-gitlab-ci-works-from-push--runner--status)
- [12. Runners (Shared vs Specific, Shell vs Docker vs Kubernetes)](#12-runners-shared-vs-specific-shell-vs-docker-vs-kubernetes)
- [13. Web IDE — When To Use It](#13-web-ide--when-to-use-it)
- [14. CI/CD Variables (Scoped, Masked, Protected)](#14-cicd-variables-scoped-masked-protected)
- [15. Artifacts — What/Why/How + Retention](#15-artifacts--whatwhyhow--retention)
- [16. Self-Hosted Runners (Production-Grade Setup)](#16-selfhosted-runners-production-grade-setup)
- [17. SaaS Runners (When to Use & Limits)](#17-saas-runners-when-to-use--limits)
- [18. “Pipeline stuck” Errors — Root Causes & Fixes](#18-pipeline-stuck-errors--root-causes--fixes)
- [19. DevSecOps with OWASP — Practical Setup](#19-devsecops-with-owasp--practical-setup)
- [20. Real-World GitLab Errors & Scenarios (with fixes)](#20-realworld-gitlab-errors--scenarios-with-fixes)
- [21. Reference Pipeline Snippets (Your Examples: reviewed & corrected)](#21-reference-pipeline-snippets-your-examples-reviewed--corrected)
- [22. Bonus Topics (Monorepos, Review Apps, Caching, DIND, K8s)](#22-bonus-topics-monorepos-review-apps-caching-dind-k8s)
- [23. Questions

---

## 1. What is GitLab?
GitLab is an *end-to-end* DevSecOps platform: SCM (Git hosting), planning (Issues/Boards/Roadmaps), CI/CD, security scanning, package registries, environments, observability hooks, and compliance — all in one place. The main benefit is **tight integration**: one UI, one permission model, one audit trail.

**Real-world:** A fintech team uses GitLab to plan sprints, review MRs, run SAST/DAST, build Docker images, deploy to Kubernetes, and sign artifacts — without stitching 6 tools.

---

## 2. GitLab vs GitHub vs Bitbucket — Differences & Use Cases
**High-level:**
- **GitLab:** All-in-one DevSecOps, strong self-hosted story, robust CI/CD native, enterprise compliance, Duo AI.
- **GitHub:** Massive OSS gravity, Actions are great for CI, strong ecosystem & marketplace.
- **Bitbucket:** Deep Atlassian integration (Jira/Confluence), pipelines improving, popular in orgs standardized on Atlassian.

**Use Cases:**
- **GitLab:** Regulated industries (health/finance), self-hosting, built-in security & compliance, mono-platform governance.
- **GitHub:** OSS/community, marketplace actions, developer familiarity.
- **Bitbucket:** Jira-first shops; simple pipelines; Confluence-centric documentation.

---

## 3. Groups & Projects Explained
- **Groups**: Top-level containers for orgs/departments. Inherit permissions, share runners/variables, enforce policies.
- **Subgroups**: Structure for domains/teams (e.g., `company/platform/k8s`).
- **Projects**: Repos with issues, boards, CI/CD, registries, releases.

**Real-world:** `company/payments` group with subgroups `services`, `infra`, `mobile`; projects hold microservices and IaC.

---

## 4. Using GitLab with AI (Code Suggestions, Security, Automation)
- **Code suggestions** in MR diffs and Web IDE.
- **AI explanations** for failing jobs, MR summaries, test failure root causes.
- **AI security**: triage vulnerabilities, suggest fixes, generate policies.
- **AI in ops**: summarize logs, recommend remediation steps.

**Impact:** Faster code reviews, reduced cognitive load, quicker MTTR.

---

## 5. GitLab Duo — Why & How + Use Cases
**Why:** AI copiloting across plan→code→secure→deploy.  
**How:** Enable at group/project level; assign seats; users opt-in.  
**Use cases:** Autocomplete CI YAML; explain job failures; draft secure code; propose MR descriptions; generate unit tests.

---

## 6. Authentication: “There are no passwords” — SSH, PATs, SSO/SAML
- **Human login:** SSO/SAML/OIDC (Okta/AzureAD/Auth0). Avoid local passwords.
- **CLI/repo:** **SSH keys** (preferred) or **PATs** (scoped, expiry, masked).
- **Bots/automation:** Project/Group Access Tokens with least privilege.
- **Best practice:** Enforce MFA via IdP; protect main branches with code owners + approvals.

---

## 7. Auto DevOps (Zero-to-Prod Pipelines)
Auto-detects languages, builds, tests, scans, containerizes, and deploys to K8s using opinionated templates. Great for greenfield and prototypes, or as a baseline before customizing.

**Real-world:** Startup ships a Rails API in a day to a managed K8s cluster with SAST/DAST enabled by default.

---

## 8. Creating & Managing Projects/Groups (RBAC & Governance)
- **Groups**: set **Owners/Maintainers/Developers/Reporters/Guests**.
- **Compliance**: approval rules, code owners, MR templates, security policies.
- **Auditing**: audit events, protected branches/tags, signed commits.

**Tip:** Put environment secrets at **group level** (protected variables) and inherit to projects.

---

## 9. Creating Pipelines in GitLab (Why Jenkins isn’t required)
GitLab CI is *built-in*: no extra server, plugins, or ACL layer. You define pipelines in `.gitlab-ci.yml`, runners pick jobs automatically, permissions follow the same project RBAC, artifacts/registry/secrets are native.

**When Jenkins still makes sense:** Legacy shared libraries, complex multi-platform build farms, or org mandates. Otherwise GitLab CI reduces operational overhead.

---

## 10. File Naming Conventions
- `.gitlab-ci.yml` at repo root.
- Lowercase, hyphenated filenames for scripts: `build-image.sh`, `deploy-k8s.sh`.
- IaC: `terraform/`, `helm/`, `k8s/`.
- Keep job scripts small; move complex logic to versioned scripts in `ci/` or `scripts/`.

---

## 11. How GitLab CI Works (From Push → Runner → Status)
1. **Push/MR** triggers pipeline → GitLab schedules jobs by **stage**.
2. **Runner** (Docker/Shell/Kubernetes) picks jobs matching **tags**.
3. Job executes with **variables** + **cache** + **artifacts**.
4. Results update MR: status checks, code quality, vulnerability reports.

---

## 12. Runners (Shared vs Specific, Shell vs Docker vs Kubernetes)
- **Shared:** Managed by admin; available to many projects; good for bursty workloads.
- **Specific:** Dedicated to project/group; control images, network, secrets.
- **Executors:**
  - **Docker**: reproducible, easy caching, DIND for builds.
  - **Shell**: fast but shares host; careful with isolation.
  - **Kubernetes**: ephemeral pods per job; best isolation/scale.

**Tagging:** Use `tags:` on jobs and configure runner tags to match.

---

## 13. Web IDE — When To Use It
Edit files, create branches, commit, and open MRs directly in the browser.  
**Use cases:** Hotfix docs, tweak CI YAML, review small changes, pair with Duo for quick test/mr descriptions.

---

## 14. CI/CD Variables (Scoped, Masked, Protected)
- **Masked/Protected** for secrets (only on protected branches/tags).
- **Group-level** for shared creds/URLs.
- **Environment-scoped**: `STAGING_DB_URL` vs `PROD_DB_URL`.
- **File variables** for kubeconfigs, service accounts, or JSON keys.

**Example (plain text shown by indentation):**
    
    variables:
      DOCKER_DRIVER: overlay2
      DOCKER_TLS_CERTDIR: ""
      APP_ENV: production

---

## 15. Artifacts — What/Why/How + Retention
Artifacts persist job outputs to later stages or for download (e.g., test logs, reports, build bundles).

**Best practices:**
- Retain only what you need (`expire_in`).
- Use `paths:` for specific directories; avoid bloating with node_modules.
- Use **reports** (junit/codequality/sast) to decorate MRs.

**Example (plain text):**
    
    test_job:
      stage: test
      script:
        - pytest --junitxml=reports/junit.xml
      artifacts:
        when: always
        expire_in: 7 days
        paths:
          - reports/
        reports:
          junit: reports/junit.xml

---

## 16. Self-Hosted Runners (Production-Grade Setup)
- Hardened VM or K8s node pool.
- Use Docker executor with pinned image digests.
- Private registry mirror for speed.
- Cache: S3/GCS backend to speed builds.
- Autoscaling with spot pools (cost-optimized).
- Network egress controls (egress-only NAT).

---

## 17. SaaS Runners (When to Use & Limits)
- Great for quick starts and public projects.
- Limited customization of underlying images/network.
- Avoid for regulated workloads or when you need VPC-only access.

---

## 18. “Pipeline stuck” Errors — Root Causes & Fixes
Common reasons:
- No runner available / no matching **tags**.
- Runners paused/offline.
- Project disallowed shared runners.
- Manual job awaiting play.
- Protected variables on unprotected branch.

**Fix:** Align `tags:`, enable runners, set permissions, convert manual → auto if intended, ensure variables are accessible.

---

## 19. DevSecOps with OWASP — Practical Setup
- **SAST**: enable built-in analyzers; block MR if high severity.
- **DAST**: run against Review Apps or staging.
- **Dependency scanning**: SBOM + license compliance.
- **Secret detection**: prevent committing keys.
- **OWASP Top 10**: Add checks in CI (ZAP for DAST, Semgrep rules).

**Example (plain text):**
    
    include:
      - template: Security/SAST.gitlab-ci.yml
      - template: Security/DAST.gitlab-ci.yml

---

## 20. Real-World GitLab Errors & Scenarios

### A) “Pipeline stuck. This job is stuck because the project doesn’t have any runners online…”
**Why:** No runner or tag mismatch.  
**Fix:** Start runner; ensure job `tags:` match runner tags; enable shared runners for project.

### B) “No stages / jobs found”
**Why:** `.gitlab-ci.yml` invalid or all jobs excluded by `rules/only/except`.  
**Fix:** Validate YAML; add at least one job with a `stage`; verify `rules:` evaluate to true.

### C) Docker build errors in runners
- **Cannot connect to the Docker daemon at unix:///var/run/docker.sock**  
  Use Docker-in-Docker service or privileged runner with `services: [docker:dind]` and `DOCKER_TLS_CERTDIR=""`.

### D) Registry push denied
- **denied: requested access to the resource is denied**  
  Wrong registry path or missing `docker login`. Use CI variables for `CI_REGISTRY_USER` and `CI_REGISTRY_PASSWORD` (or use GitLab’s built-in `CI_REGISTRY` and `CI_JOB_TOKEN`).

### E) Artifacts too large / expired
- Reduce `paths:`; compress; set `expire_in` appropriately; keep junit small.

### F) Protected variables not available on feature branches
- Either mark variables unprotected (if safe) or use protected branches/tags for deployments.

### G) Kubernetes deploy fails: Unauthorized
- ServiceAccount permissions missing. Bind roles (RBAC), mount kubeconfig via masked file variable.

### H) DAST failing in PRs
- Target URL unreachable (private network). Use Review Apps with public ingress or run DAST inside cluster.

---

## 21. Reference Pipeline Snippets (Your Examples: reviewed & corrected)

### 21.1 Your First Snippet (as provided — shown as plain text)
    
    stages:
        - build
        - test
        - push
        - deploy
    
    build_job:
        stage: build
        script:
            - echo "This build is done using command <docker build -t .>"
    
    test_job:
        stage: test
        script: 
            - echo "This is testing of docker build"
    
    push_job:
        stage: push
        script:
            - echo "this is pushing to docker hub"
    
    deploy_job:
        stage: deploy
        script:
            - echo "This is deploying to EC2 instance"

**Review & Suggestions:**
- Add an image (e.g., `docker:20`) and `services: [docker:dind]` if actually building images.
- Push stage should authenticate and push to a registry.
- Use artifacts between build → push if needed.
- Add `rules:` for branches/tags and environment for deploy.

**A more realistic corrected version (plain text):**
    
    image: docker:20
    services:
      - name: docker:20-dind
    variables:
      DOCKER_TLS_CERTDIR: ""
      IMAGE: $CI_REGISTRY_IMAGE/app:$CI_COMMIT_SHORT_SHA
    
    stages:
      - build
      - test
      - push
      - deploy
    
    build_job:
      stage: build
      script:
        - docker build -t "$IMAGE" .
      artifacts:
        expire_in: 1 week
        reports:
          dotenv: build.env
      rules:
        - if: $CI_PIPELINE_SOURCE == "push"
    
    test_job:
      stage: test
      needs: ["build_job"]
      script:
        - echo "Run tests here (unit/integration)"
    
    push_job:
      stage: push
      needs: ["test_job"]
      script:
        - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
        - docker push "$IMAGE"
      rules:
        - if: $CI_COMMIT_BRANCH == "main"
    
    deploy_job:
      stage: deploy
      needs: ["push_job"]
      environment:
        name: production
        url: https://your-app.example.com
      script:
        - ./scripts/deploy.sh "$IMAGE"
      rules:
        - if: $CI_COMMIT_TAG

---

### 21.2 snipet
    
    stages:
        - build
        - test
        - push
        - deploy
    variables:
        DEPLOY_ENV: "production"
        GITLAB_USER_KEY: "key"
    build_job:
        stage: build
        script:
            - echo "This $CI_PROJECT_NAME build is done using command <docker build -t .>"
    
    test_job:
        stage: test
        script: 
            - echo "This is testing of docker build done by $CI_COMMIT_AUTHOR"
    
    push_job:
        stage: push
        script:
            - echo "this is pushing to docker hub for $DEPLOY_ENV"
    
    deploy_job:
        stage: deploy
        script:
            - echo "This is deploying to EC2 instance"
    
    dev_test_job:
        stage:
        script:
            - echo "tested for dev using $GITLAB_USER_KEY"

**Issues Identified:**
- `dev_test_job.stage` is empty ⇒ pipeline lints will fail.
- No images/services for Docker work.
- Using `$CI_COMMIT_AUTHOR` is not a standard predefined variable in all contexts (prefer `$GITLAB_USER_NAME` or `$GITLAB_USER_LOGIN` in some versions).
- Secrets like `GITLAB_USER_KEY` should be masked variables, not plaintext in YAML.

**Corrected version (plain text):**
    
    image: alpine:3.20
    
    stages:
      - build
      - test
      - push
      - deploy
    
    variables:
      DEPLOY_ENV: "production"
    
    build_job:
      stage: build
      script:
        - echo "Building $CI_PROJECT_NAME"
    
    test_job:
      stage: test
      script:
        - echo "Testing by $GITLAB_USER_LOGIN"
    
    push_job:
      stage: push
      script:
        - echo "Pushing artifacts for $DEPLOY_ENV"
    
    deploy_job:
      stage: deploy
      environment:
        name: production
      script:
        - echo "Deploying to EC2"
    
    dev_test_job:
      stage: test
      rules:
        - if: $CI_COMMIT_BRANCH =~ /^dev/
      script:
        - echo "Tested for dev branch"

---

### 21.3 Artifacts Example
    
    test_job:
      stage: test
      script:
        - echo "This is testing of docker build done by $GITLAB_USER_LOGIN"
        - mkdir -p logs
        - echo "these are my test results" > logs/app.log
      artifacts:
        expire_in: 1 week
        paths:
          - logs/
    
    push_job:
      stage: push
      needs: ["test_job"]
      script:
        - echo "this is pushing to docker hub for $DEPLOY_ENV"
    
    deploy_job:
      stage: deploy
      needs: ["push_job"]
      script:
        - echo "This is deploying to EC2 instance for $DOCKERHUB_USER"
        - echo "this log is from $CI_JOB_STAGE" >> logs/app.log
      artifacts:
        expire_in: 1 week
        paths:
          - logs/

**Note:** A downstream job cannot append to a previous job’s artifact file unless it **downloads/unpacks** artifacts. Prefer producing new logs per stage or use `dependencies:`/`needs:` to fetch artifacts into later jobs.

---

## 22. Bonus Topics (Monorepos, Review Apps, Caching, DIND, K8s)

**Monorepos:** Use `rules:changes` to trigger per-folder pipelines; component-level caching; codeowners for routing approvals.

**Review Apps:** Auto-deploy each MR to ephemeral URLs; enables DAST on preview environments and stakeholder UAT before merge.

**Caching:** Speed builds (npm/pip/maven/gradle). Use S3/GCS cache with keys per branch + lockfile hash.

**DIND:** For building container images, use `docker:dind` service with `DOCKER_TLS_CERTDIR=""` and privileged runner.

**Kubernetes:** Use `environments`, `kubectl` or Helm jobs, and GitOps (ArgoCD/Flux) by pushing manifests to infra repos.

---

## 23. Questions 

**1) What triggers a GitLab pipeline?**  
Pushes, MRs, schedules, webhooks, API calls, and manual jobs via `when: manual`.

**2) Difference between `only/except` and `rules`?**  
`rules` is modern, more expressive (conditions, variables, pipeline source). Prefer `rules`.

**3) Why are my jobs “stuck”?**  
No runner/paused runner/tag mismatch/manual job/protected variable on unprotected branch. Fix runners/tags/rules/variable scope.

**4) Docker image builds fail on runner — why?**  
Not using DIND or privileged mode. Add `services: [docker:dind]` and set `DOCKER_TLS_CERTDIR=""`.

**5) How do artifacts differ from cache?**  
Artifacts persist outputs between stages/pipelines for download; cache speeds dependency reuse and can be restored across jobs.

**6) How to manage secrets safely?**  
Use CI/CD variables (masked/protected, short-lived), group-level for inheritance, environment scoping, HashiCorp Vault integration.

**7) When use self-hosted vs SaaS runners?**  
Self-hosted for VPC access/compliance/custom images; SaaS for quick starts/public repos.

**8) How to deploy per MR?**  
Review Apps with dynamic environment names using `$CI_COMMIT_REF_SLUG` and destroy on close/merge.

**9) GitLab vs Jenkins?**  
GitLab CI is native to SCM with built-in approvals/artifacts/security; Jenkins is plugin-rich but requires separate admin and auth wiring.

**10) Explain `needs:` vs `dependencies:`**  
`needs:` defines DAG execution (parallel stages) and artifact download; `dependencies:` controls which artifacts to fetch (older syntax). Prefer `needs:`.

**11) What is Auto DevOps? Pros/Cons?**  
Pros: fast path to CI/CD + security; Cons: opinionated; may require overrides for complex stacks.

**12) Typical OWASP checks in CI?**  
SAST, DAST (ZAP), Dependency scanning, Secret detection, License compliance, Container scanning.

**13) How to enforce code owners & approvals?**  
Set `CODEOWNERS`, branch protection, MR approval rules; enforce on `main/release` branches.

**14) How to speed a slow pipeline?**  
Parallelize tests, cache deps, split jobs, use smaller base images, avoid “latest”, use remote cache backends.

**15) How to run a job only on tags?**  
Use `rules: - if: $CI_COMMIT_TAG`.

**16) What are protected branches/tags?**  
Restricted force-push/merge/tag creation; required for production releases; variables can be protected to only expose on protected refs.

**17) How to deploy to Kubernetes safely?**  
Use environment-scoped variables, RBAC service accounts, helm charts, and GitOps for drift detection.

**18) Common cause of “Access denied” when pushing images?**  
Not logging into registry, wrong image path, missing `CI_JOB_TOKEN`/PAT permissions.

**19) How to run matrix jobs?**  
Use `parallel: matrix` to fan out combinations (e.g., language version × OS).

**20) How does GitLab Duo help CI?**  
Explains failures, proposes YAML fixes, generates job skeletons, summarizes MRs, and suggests secure code changes.

---



