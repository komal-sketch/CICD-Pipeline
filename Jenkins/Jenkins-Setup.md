# Jenkins

This README gives you a **clear, non-complicated** overview of Jenkins and a **clean set of steps** to install it (LTS) and run your **first job**.

---

## 🧠 Intro (What is Jenkins?)

- **Jenkins** is a popular **CI/CD server** written in **Java**.
- You define **Jobs** (Freestyle or Pipeline) that build, test, and deploy code.
- Jenkins can run builds on the **controller** or on separate **agents** (nodes).
- A **Pipeline** (Jenkinsfile) is just code that describes stages/steps (recommended for teams).

**Core pieces (kept simple):**
- **Jenkins controller** – the web UI, scheduling, plugins, credentials.
- **Jobs** – work definitions (Freestyle = form-based; Pipeline = code-based).
- **Workspace** – per-job folder where build steps run (usually `/var/lib/jenkins/workspace/<job>`).
- **Plugins** – add integrations (Git, Docker, Kubernetes, etc.).
- **Credentials** – secure storage for tokens/keys to use in jobs.

---

## ✅ Requirements

- A Linux host (e.g., Ubuntu on an **EC2 instance** in AWS).
- **OpenJDK 21+** (Jenkins runs on Java).
- **Port 8080** open on the server/firewall (**EC2 Security Group**: allow inbound TCP 8080 from your IP).

---

## ⚙️ Setup

Official docs (Linux): https://www.jenkins.io/doc/book/installing/linux/

### 1) Install Java (OpenJDK 21)

    sudo apt update
    sudo apt install -y fontconfig openjdk-21-jre
    java -version
    # Expected:
    # openjdk version "21.0.3" 2024-04-16
    # OpenJDK Runtime Environment (build 21.0.3+11-Debian-2)
    # OpenJDK 64-Bit Server VM (build 21.0.3+11-Debian-2, mixed mode, sharing)

### 2) Install Jenkins (LTS)

    # Add Jenkins LTS repo key and list
    sudo install -d -m 0755 /etc/apt/keyrings
    sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

    echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
      sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

    sudo apt update
    sudo apt install -y jenkins

### 3) Manage the service

    systemctl status jenkins
    sudo systemctl enable jenkins
    sudo systemctl start jenkins

> If you’re on EC2, **allow inbound TCP 8080** on the Security Group.

### 4) Access the UI

- Open a browser to **`http://<EC2_PUBLIC_IP>:8080/`**  
  (replace with your server’s IP or hostname)

### 5) Unlock Jenkins (initial admin password)

    sudo cat /var/lib/jenkins/secrets/initialAdminPassword

- Copy the token and paste it into the **Unlock Jenkins** screen.
- Install **suggested plugins** (good default) and create your **admin user**.

---

## 🖥️ Jenkins UI → Dashboard → Jobs

### Create your first job (Freestyle)

1. **New Item** → name: **`first-job`** → **Freestyle project** → OK  
2. **Build** section → **Add build step** → **Execute shell**  
3. Add simple commands:

        echo "hello Jenkins"
        mkdir -p devops
        echo "Devops folder created"

4. **Save** → **Build Now**  
5. Click the build → **Console Output** to view results.

**Where it ran:**  
Workspace is typically at:

    /var/lib/jenkins/workspace/first-job

---

## 🧰 Handy commands

    # Service
    systemctl status jenkins
    sudo systemctl restart jenkins

    # Logs (Debian/Ubuntu)
    sudo journalctl -u jenkins -f

    # Jenkins home (jobs, plugins, secrets)
    sudo ls -la /var/lib/jenkins

---

## 🧠 What to learn next (quick pointers)

- **Pipelines (Jenkinsfile)** – pipeline-as-code with stages/steps.
- **SCM** – connect to GitHub/GitLab; trigger builds on PRs/commits.
- **Credentials** – store tokens/SSH keys and reference them in jobs.
- **Agents** – run builds on separate Linux/Windows/Docker/Kubernetes nodes.
- **Backups** – back up `/var/lib/jenkins` (plugins, jobs, creds, config).

---

## 🩺 Troubleshooting basics

- If UI isn’t reachable: verify port 8080 is open and `systemctl status jenkins` is **active (running)**.
- If Java errors appear: check `java -version` is 21+; ensure `openjdk-21-jre` is installed.
- If builds fail: check **Console Output**, workspace permissions, and disk space.

---

## ✅ Summary

- Jenkins runs on **Java** and is installed with the **Debian LTS repo**.  
- You accessed the UI on **port 8080**, unlocked with the **initialAdminPassword**, and created a **Freestyle job** that ran shell commands.  
- Next step: move to **Pipelines**, connect **SCM**, and add **agents** for scalable CI/CD.
