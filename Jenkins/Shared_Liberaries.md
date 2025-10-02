# Jenkins Shared Libraries & Groovy for Pipelines

This file focuses **only** on:
- **Jenkins Shared Libraries** (what, why, how, structure, usage, versioning, testing, security)
- **Groovy for Jenkins Pipelines**
- **Common Questions & Error Fixes**


---

## 1) What Are Jenkins Shared Libraries?

Reusable, versioned **pipeline code** kept in a Git repo and loaded by many Jenkins jobs. They let you:
- **De-duplicate** steps (build, push, deploy, scan, notify).
- **Enforce standards** (quality gates, naming, credentials, policies).
- **Ship updates centrally** (one PR updates all consumers).
- **Harden security** (no secrets in Jenkinsfile; audited code paths).

Real-world outcomes:
- Consistent CI/CD across dozens of services.
- Faster onboarding (teams call `dockerBuildPush()` instead of copy-pasting).
- Governance (mandatory stages; approved library versions).

---

## 2) How to Register a Shared Library in Jenkins

- Go to:
  
    Manage Jenkins → System → Global Pipeline Libraries → Add

- Key fields:
  
    Name: Shared
    Default version: main (or a **tag** like v1.2.0 for stability)
    Retrieval: Modern SCM / Git → set repo URL
    Load implicitly: (optional) only if you want auto-load for all jobs
    Allow default version override: Enable to support @Library('Shared@vX.Y.Z')

- For **private** repos: configure credentials.

---

## 3) Repository Structure (Typical & Recommended)

    jenkins-shared-lib/
      vars/                          # simple global steps (exposed as functions)
        hello.groovy
        dockerUtils.groovy
        notifySlack.groovy
        kubeDeploy.groovy
      src/                           # namespaced Groovy classes for richer logic
        org/company/ci/ImageTag.groovy
        org/company/ci/Semver.groovy
      resources/                     # static assets (templates, manifests)
        templates/notify.md
        compose/docker-compose.prod.yml
      test/                          # (optional) JenkinsPipelineUnit tests
        ... 
      README.md

- `vars/*.groovy`: Expose a `call()` method or methods; callable directly in Jenkinsfile.
- `src/**.groovy`: Use packages and classes for complex logic.
- `resources/**`: Load with `libraryResource 'path'`.

---

## 4) Writing Steps in `vars/`

**Example: `vars/hello.groovy`**

    // vars/hello.groovy
    def call(String name = 'world') {
      echo "Hello, ${name}! (from Shared Library)"
    }

Use in Jenkinsfile:

    @Library('Shared') _
    pipeline {
      agent any
      stages {
        stage('Greet') {
          steps { script { hello('Team') } }
        }
      }
    }

**Example: `vars/dockerUtils.groovy`**

    // vars/dockerUtils.groovy
    def build(String image, String tag = 'latest') {
      sh "docker build -t ${image}:${tag} ."
    }
    def login(String user, String pass) {
      sh "echo '${pass}' | docker login -u '${user}' --password-stdin"
    }
    def push(String image, String tag = 'latest') {
      sh "docker push ${image}:${tag}"
    }

Use with credentials binding:

    withCredentials([usernamePassword(credentialsId: 'dockerhubcred',
                                      usernameVariable: 'DOCKER_USER',
                                      passwordVariable: 'DOCKER_PASS')]) {
      dockerUtils.login(env.DOCKER_USER, env.DOCKER_PASS)
      dockerUtils.build("${env.DOCKER_USER}/notes-app", "latest")
      dockerUtils.push("${env.DOCKER_USER}/notes-app", "latest")
    }

**Example: `vars/kubeDeploy.groovy` (Compose/K8s wrapper)**

    // vars/kubeDeploy.groovy
    def composeUp(String file = 'docker-compose.yml') {
      sh "docker compose -f ${file} up -d"
    }
    def kubectlApply(String manifest) {
      sh "kubectl apply -f ${manifest}"
    }

---

## 5) Consuming Libraries in Jenkinsfile

- Top of your Jenkinsfile:

    @Library('Shared') _
    // or pin exact version for prod-hardening:
    // @Library('Shared@v1.2.0') _

- Call your steps:

    pipeline {
      agent { label 'linux-docker' }
      stages {
        stage('Build & Push') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'dockerhubcred',
                                              usernameVariable: 'DOCKER_USER',
                                              passwordVariable: 'DOCKER_PASS')]) {
              script {
                dockerUtils.login(env.DOCKER_USER, env.DOCKER_PASS)
                dockerUtils.build("${DOCKER_USER}/svc", "latest")
                dockerUtils.push("${DOCKER_USER}/svc", "latest")
              }
            }
          }
        }
      }
    }

- Load a file from `resources/`:

    def tpl = libraryResource 'templates/notify.md'
    echo tpl

---

## 6) Versioning & Release Strategy

- **Tag releases** of the shared library (e.g., `v1.3.4`).
- **Pin** critical pipelines:

    @Library('Shared@v1.3.4') _

- Maintain a **changelog**; communicate breaking changes.
- Use **branches** for development; PR to main; automate tests on PR (JenkinsPipelineUnit).

---

## 7) Testing Shared Libraries

- Use **JenkinsPipelineUnit** to unit test steps locally (fast feedback).
- Mock Jenkins steps like `sh`, `withCredentials`, `git`, `echo`.
- Validate:
  - command strings (safe quoting)
  - parameter combinations
  - error paths (non-zero returnStatus)
  - environment behavior

High ROI: Catch breaking changes **before** teams see them.

---

## 8) Security & Compliance

- **Never** hardcode secrets in library code. Always use:
  
    withCredentials([usernamePassword(...)]) { ... }

- Approve **script signatures** if using sandboxed Groovy constructs.
- Restrict who can change the library repo (code owners).
- Optionally disable “Load implicitly” to force opt-in usage.
- Consider **signed commits/tags** for library releases.

---

## 9) Groovy Essentials for Jenkins Pipelines

- **Strings**:
  
    "Double quotes interpolate: ${var}"
    'Single quotes do not interpolate'

- **Lists & Maps**:
  
    def xs = ['a','b','c']
    def cfg = [image: 'svc', tag: 'latest']
    xs.each { echo it }
    echo cfg.image

- **Closures**:
  
    xs.findAll { it.contains('b') }.each { echo it }

- **Methods in vars/**:
  
    // expose as a step
    def call(args...) { ... }

- **DSL vs Groovy**:
  - `pipeline`, `stage`, `steps`, `agent`, `when` are **Jenkins DSL**, not pure Groovy.
  - Use `script { ... }` to run arbitrary Groovy within Declarative Pipeline.

- **Safe navigation**:
  
    echo user?.name ?: 'unknown'

- **GString pitfalls**:
  - Some Java APIs dislike GStrings; cast to String:
    
      String s = "${var}".toString()

- **Multiline shell**:
  
    sh """
      set -e
      echo "building..."
      docker build -t ${IMAGE}:${TAG} .
    """

---

## 10) Pipeline Gotchas (CPS, Serialization, Sandbox)

- **CPS (Continuation Passing Style)**:
  - Jenkins serializes pipeline state. **Don’t keep non-serializable objects** across steps/suspensions.
  - Avoid storing live process handles, InputStreams, large closures in global vars.

- **NonSerializableException**:
  - Wrap transient objects inside the same `script` block; don’t return them to outer scope.
  - Convert to plain data (String, Map) before leaving the step.

- **Sandbox**:
  - If you see “Scripts not permitted to use method…”, an admin must approve or you must refactor to permitted APIs.

- **`node` vs `agent`**:
  - In Declarative, prefer `agent` at `pipeline` or `stage` level.
  - In Scripted, you explicitly use `node('label') { ... }`.

---

## 11) Common Errors & Fixes (Copy/Paste Friendly)

- **`@Library` not found / step undefined**  
  - Ensure library is registered under **Manage Jenkins → Global Pipeline Libraries** with **Name** matching `@Library('Name')`.
  - Correct the leading underscore after annotation:
    
      @Library('Shared') _        # required

  - If pinning a version, ensure the tag exists: `@Library('Shared@v1.2.0')`.

- **Missing method `hello` / step not found**  
  - The file must be `vars/hello.groovy` with a `call()` method or a method named `hello(...)` and invoked appropriately.
  - File name and function exposure must match usage.

- **Credentials errors**  
  - `credentialsId` wrong or not global. Create under:
    
      Manage Jenkins → Credentials → (Global) → Add Credentials

  - Ensure variable names in `withCredentials` match what your code reads.

- **Docker permission denied**  
  - Add agent user (e.g., `jenkins`) to `docker` group:
    
      sudo usermod -aG docker jenkins
      # re-login / restart agent

- **GString vs String type mismatch**  
  - Use `.toString()` when passing to Java libs:
    
      someJavaApi("${val}".toString())

- **Non-serializable errors**  
  - Don’t store command outputs as complex objects between steps; capture as String with `sh(returnStdout: true, script: '...').trim()` and pass around simple data.

- **Sandbox signature**  
  - Admin: **Manage Jenkins → In-process Script Approval** → approve needed signatures.

- **Library changes not picked up**  
  - If caching occurs, bump version or tag and re-pin `@Library('Shared@vX.Y.Z')`. Disable “Load implicitly” to force explicit calls.

---

## 12) FAQ (Short, Industry-Focused)

- **Why use Shared Libraries instead of pasting steps in every Jenkinsfile?**  
  - Centralized updates, governance, less drift, fewer copy/paste bugs, faster rollout of security fixes.

- **How big should a library be?**  
  - Start lean; group steps by domain (build, test, deploy, notify). Split into multiple libs if scope grows too large.

- **Declarative vs Scripted?**  
  - Prefer **Declarative** for team consistency. Use `script {}` for advanced Groovy. Scripted is flexible but harder to standardize.

- **How do I version and roll back?**  
  - Tag library releases; pin with `@Library('Shared@vX.Y.Z')`. Rollback by pinning a previous tag.

- **How do I test library changes?**  
  - JenkinsPipelineUnit for unit tests; a staging Jenkins or sandbox job for integration tests.

- **Can I load multiple libraries?**  
  - Yes, comma-separate: `@Library(['Shared','SecurityLib@v2']).`

---

## 13) Example: Minimal but Real Jenkinsfile Using Shared Library

    @Library('Shared@v1.0.0') _
    pipeline {
      agent { label 'linux-docker' }

      environment {
        IMAGE = 'notes-app'
        TAG   = 'latest'
      }

      stages {
        stage('Checkout') {
          steps {
            git url: 'https://github.com/your-org/notes-app.git', branch: 'main'
          }
        }

        stage('Build & Push') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'dockerhubcred',
                                              usernameVariable: 'DOCKER_USER',
                                              passwordVariable: 'DOCKER_PASS')]) {
              script {
                dockerUtils.login(env.DOCKER_USER, env.DOCKER_PASS)
                dockerUtils.build("${DOCKER_USER}/${IMAGE}", TAG)
                dockerUtils.push("${DOCKER_USER}/${IMAGE}", TAG)
              }
            }
          }
        }

        stage('Deploy (Compose)') {
          steps {
            script {
              kubeDeploy.composeUp('docker-compose.yml')
            }
          }
        }
      }
    }

---

## 14) Migration Tips & Governance

- Move duplicated steps from old Jenkinsfiles into `vars/` with clear names and docs.
- Add `@Library('Shared@tag')` to critical repos first; expand gradually.
- Enforce pre-merge checks on the library repo (tests, lint).
- Publish usage docs in the library README.
- Avoid breaking changes; when necessary, release a major tag and keep old tag alive.

---

## 15) Quick Checklists

- Library registered in Jenkins with correct **Name**?
- Jenkinsfile has `@Library('Name') _` with **underscore**?
- Steps defined under `vars/` use `def call(...)`?
- Credentials exist and `credentialsId` matches?
- Agent has required tools (Java, Docker, kubectl, compose)?
- Non-serializable state avoided between steps?
- Sandbox approvals completed?

