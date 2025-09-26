# Jenkins: Built-in Node vs Agent Node + Declarative Pipeline

Simple explanations with a runnable flow to compare **built-in** vs **agent** nodes and to run a **Declarative Pipeline** on a remote agent.

---

## 🧠 Concepts (kept simple)

**Built-in node (a.k.a. master/controller executor)**  
- The Jenkins controller itself can run builds (if executors are enabled).  
- Good for quick demos. In production, many teams set built-in executors to **0** and use agents instead (safer, scalable).

**Agent node**  
- A separate machine/VM/container that runs builds.  
- Connects to the controller (SSH or JNLP/Inbound).  
- Lets you scale horizontally and isolate toolchains (e.g., Docker, Android SDK, Node.js).

**Why Pipelines (vs Freestyle)**  
- Pipelines are **code** (Jenkinsfile), versioned with your repo.  
- Enable **stages**, **parallelism**, **retries**, **post** actions, **when** conditions, **environment** vars, and **agent** selection by label.  
- Easier to share patterns and review in PRs.

---

## 🎬 Create a Pipeline Job (Hello World)

1) **New Item** → name: **Demo-CICD**  
2) Select **Pipeline**  
3) Description: **this is demo-cicd pipeline**  
4) Pipeline definition: **Pipeline script** (inline for now) → paste script below

**Script: Hello World (multiple stages possible)**

    pipeline {
      agent any     // runs on any available executor (built-in or any agent)
      stages {
        stage('Hello') {
          steps {
            echo 'Hello World'
          }
        }
        stage('Build') {
          steps {
            sh '''
              echo "Simulating build..."
              mkdir -p output
              echo "artifact.txt" > output/artifact.txt
            '''
          }
        }
        stage('Test') {
          steps {
            sh 'echo "Running tests..."'
          }
        }
        stage('Deploy') {
          when { expression { return false } } // example gate, set to true when ready
          steps {
            sh 'echo "Deploying..."'
          }
        }
      }
      post {
        always {
          echo 'Pipeline finished (success or fail).'
          archiveArtifacts artifacts: 'output/**', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }

5) **Build Now**  
6) **Console Output** → you’ll see each stage, logs, and post actions.

**Benefit over Freestyle**  
- Pipelines are portable, reviewable, re-runnable, and support real delivery patterns (fan-out parallel tests, canaries, retries). Freestyle is form-based, harder to reuse and version.

---

## 🧑‍💻 “agent any” explained

- `agent any` = run on any available node.  
- In real teams you use **labels** to pin to specific environments (e.g., `linux && docker`, `android`, `gpu`).  
- Example:

    pipeline {
      agent { label 'node' } // only run on nodes with label "node"
      stages { /* ... */ }
    }

---

## 🏭 Declarative Pipeline in industry (plain answer)

- Declarative Pipeline (the `pipeline { ... }` syntax) is widely used because it’s **opinionated**, **readable**, and supports:  
  - Clear **stages** and **post** steps  
  - **Options** (timeouts, timestamps)  
  - **Tools** and **environment** blocks  
  - **Library** steps (shared libraries)  
- Teams codify delivery patterns once (lint, build, test, package, scan, deploy), then reuse across repos.

**Short example with common patterns**

    pipeline {
      agent { label 'docker' }
      options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
      }
      environment {
        NODE_OPTIONS = '--max-old-space-size=2048'
      }
      stages {
        stage('Checkout') {
          steps { checkout scm }
        }
        stage('Build') {
          steps {
            sh 'docker build -t myapp:${BUILD_NUMBER} .'
          }
        }
        stage('Unit Tests') {
          steps {
            sh 'npm ci && npm test -- --ci --reporter=junit'
            junit 'reports/**/*.xml'
          }
        }
        stage('Security Scan') {
          steps {
            sh 'trivy image --exit-code 0 myapp:${BUILD_NUMBER} || true'
          }
        }
        stage('Package & Publish') {
          when { branch 'main' }
          steps {
            sh 'docker push myrepo/myapp:${BUILD_NUMBER}'
          }
        }
      }
      post {
        success { echo '🎉 Success' }
        failure { echo '❌ Failed' }
        always  { cleanWs() }
      }
    }

---

## ⚠️ Common Pipeline Issues (quick)

- **Wrong label**: pipeline says `agent { label 'node' }` but no agents have that label.  
- **SSH auth fails**: bad key, wrong user, missing `authorized_keys`, or file permissions.  
- **Java mismatch**: controller or agent lacks required Java (Jenkins requires Java 17+; many use 21).  
- **PATH/tooling**: tools not installed on the agent; ensure proper environment or use containers.  
- **Workspace conflicts**: use `cleanWs()` or unique workspaces for multibranch builds.

---

## 🔗 Set Up an SSH Agent (Controller = “Jenkins-Master”, Agent = “Jenkins-Agent”)

**Goal**: Make Jenkins controller connect to a Linux agent over **SSH** and run builds there.

### 1) Launch an EC2 named **Jenkins-Agent**
- Ubuntu (e.g., 22.04), security group allows **SSH (22)** from the controller.

### 2) Install Java on the Agent

    sudo apt update
    sudo apt install -y openjdk-21-jre
    java -version

### 3) Generate an SSH key pair on the **Controller** (“Jenkins-Master”)

    cd ~/.ssh
    ls
    ssh-keygen -t ed25519 -C "jenkins-controller"
    ls
    # Now you should see:
    # id_ed25519 (private key) and id_ed25519.pub (public key)

**Important**  
- **Private key stays on the controller** (used by Jenkins credentials).  
- **Public key is copied to the agent** (`~/.ssh/authorized_keys`).  
  (This is the standard and secure direction; avoid placing private keys on the agent.)

### 4) Put the **public key** on the **Agent**

    # On the controller (to display the public key)
    cat ~/.ssh/id_ed25519.pub

    # On the agent:
    mkdir -p ~/.ssh && chmod 700 ~/.ssh
    nano ~/.ssh/authorized_keys
    # paste the public key from controller
    chmod 600 ~/.ssh/authorized_keys

Test SSH from controller to agent:

    ssh -i ~/.ssh/id_ed25519 ubuntu@<AGENT_PUBLIC_IP> 'echo connected'
    # should print: connected

### 5) Create the node in Jenkins UI

- **Dashboard → Manage Jenkins → Nodes → New Node**  
  - Name: **Agent-node**  
  - Type: **Permanent Agent**  
  - Description: **this is an ubuntu machine agent**  
  - **Remote root directory**: `/home/ubuntu`  
  - **Labels**: `node`  (we’ll use this in the pipeline)  
  - **Launch method**: *Launch agents via SSH*  
    - **Host**: `<AGENT_PUBLIC_IP>`  
    - **Credentials**: Add → Kind: **SSH Username with private key**  
      - Username: `ubuntu`  
      - Private key: paste contents of `~/.ssh/id_ed25519` (the **private** key from controller)  
  - Save.

### 6) Launch and verify

- From the node page → **Launch agent** (or it will auto-connect).  
- You should see it **Online**.  
- The agent will create a **`/home/ubuntu/workspace`** directory for builds.

---

## 🧪 Run the Pipeline on the Agent

Edit your **Demo-CICD** job → **Configure** → replace the top line to target our label:

    pipeline {
      agent { label 'node' } // must match the label set on Agent-node
      stages {
        stage('Hello') { steps { echo 'Hello from Agent' } }
      }
    }

**Save** → **Build Now** → check **Console Output**.  
On the agent host:

    ls -la /home/ubuntu/workspace
    # You should see the job folder(s) here

This proves the controller can schedule work to a remote server.

---

## 🧹 Useful admin commands

    # Controller or Agent service logs (systemd-based hosts may differ)
    sudo journalctl -u jenkins -f
    # Check Java
    java -version
    # SSH permissions sanity
    ls -ld ~/.ssh
    ls -l ~/.ssh/authorized_keys
    # Test connectivity
    ssh -i ~/.ssh/id_ed25519 ubuntu@<AGENT_PUBLIC_IP> 'uname -a'

---

## ✅ Summary

- **Built-in node** is fine for demos; **agents** are better for scale/isolation.  
- **Declarative Pipelines** (Jenkinsfile) are standard in industry: readable stages, reliable post steps, and agent labels.  
- **SSH agent setup**: private key stays on the controller; public key goes to the agent’s `authorized_keys`.  
- Target the agent via a **label** in your pipeline and verify builds land in the agent’s **workspace**.
