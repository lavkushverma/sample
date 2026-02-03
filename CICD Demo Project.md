Below is a **complete, beginner‑friendly CI/CD demo project** that combines:

- **GitHub** → source code version control  
- **Docker** → containerization of your app  
- **Jenkins** → CI/CD engine to automate build & deploy  
- **Ansible** → configuration management and remote deployment  

You can implement this end‑to‑end example on your laptop or cloud instances (e.g., AWS EC2 or local VM).

---

# ⚙️ CI/CD Demo Project — GitHub + Docker + Jenkins + Ansible

---

## 🧭 Project Goal

**Objective:**  
Automatically build, test, package, and deploy a simple web application (a Python or Node.js app) whenever code is pushed to GitHub.

**Pipeline workflow overview:**
1. Developer pushes code → GitHub triggers Jenkins webhook.  
2. Jenkins pulls the code, builds a **Docker image**, and pushes it to Docker Hub.  
3. Jenkins uses **Ansible** to deploy the container on a remote server (Staging/Prod).

---

## 🗄️ 1. Project Architecture

```
┌──────────────┐      webhook       ┌────────────┐
│   GitHub     │  ───────────────▶ │   Jenkins  │
│  (Source)    │                   │ (CI/CD)    │
└──────────────┘                   └────────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │   Ansible Host  │
                             │ (runs playbooks)│
                             └─────────────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │ Docker Host     │
                             │ (App deployed)  │
                             └─────────────────┘
```

---

## 🧩 Repository Structure

Your GitHub project might look like:

```
ci-cd-demo/
├── app.py                 ← Sample web app
├── requirements.txt       ← (for Python or package.json for Node)
├── Dockerfile             ← Container image recipe
├── Jenkinsfile            ← Pipeline configuration
└── ansible/
     ├── inventory          ← Target servers list
     └── playbook.yml       ← Deployment playbook
```

---

## 💻 2. Step‑by‑Step Implementation

---

### 🧰 Step 1 — Prepare Basic Web App

**Example: Python Flask app (app.py):**

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from CI/CD Pipeline - Powered by Jenkins, Docker & Ansible!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**requirements.txt**
```
Flask==3.0.0
```

You can adapt the same concept for Node.js or any language; principles remain identical.

---

### 🧱 Step 2 — Create Dockerfile

Your app needs to containerize cleanly.

```dockerfile
# Dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

Build and verify locally before automating:
```bash
docker build -t demo-flask-app .
docker run -d -p 5000:5000 demo-flask-app
```
Visit: `http://localhost:5000`

---

### ⚙️ Step 3 — Setup Jenkins Server

**Install Jenkins** (on Ubuntu EC2 or VM):
```bash
sudo apt update -y
sudo apt install openjdk-11-jdk -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update -y
sudo apt install jenkins -y
```

Start Jenkins:
```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Access Jenkins at:
```
http://<JENKINS_SERVER_IP>:8080
```

Unlock and setup plugins:
- GitHub Integration
- Docker Pipeline
- Ansible Plugin
- Pipeline Plugin
- Blue Ocean (optional, for fancy view)

---

### 🔑 Step 4 — Configure Jenkins Credentials

Inside Jenkins → **Manage Credentials**:

| ID | Type | Usage |
|----|------|--------|
| `dockerhub` | Username + Password | Used to push Docker image |
| `github` | GitHub token (optional) | Access private repos |
| `ansible_ssh` | SSH private key | Used for deployment via Ansible |

---

### 🐳 Step 5 — Install Docker on Jenkins Host

```bash
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo systemctl enable docker
sudo systemctl start docker
```

Logout/login so Jenkins user gets Docker permissions.

---

### 📦 Step 6 — Install Ansible on Jenkins Host

```bash
sudo apt install ansible -y
ansible --version
```

---

### 🌐 Step 7 — Configure Ansible Inventory and Playbook

**ansible/inventory**
```
[web]
192.168.10.20 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**ansible/playbook.yml**
```yaml
---
- hosts: web
  become: yes
  tasks:
    - name: Stop old container
      shell: docker rm -f demo-flask || true

    - name: Pull new image from Docker Hub
      shell: docker pull your_dockerhub_username/demo-flask:latest

    - name: Run new container
      shell: docker run -d --name demo-flask -p 5000:5000 your_dockerhub_username/demo-flask:latest
```

---

### ☁️ Step 8 — Create Jenkinsfile (Declarative Pipeline)

**Jenkinsfile**
```groovy
pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        IMAGE = "your_dockerhub_username/demo-flask"
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/your_username/ci-cd-demo.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $IMAGE:$BUILD_NUMBER .'
                }
            }
        }
        stage('Push to Docker Hub') {
            steps {
                script {
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    sh 'docker push $IMAGE:$BUILD_NUMBER'
                    sh 'docker tag $IMAGE:$BUILD_NUMBER $IMAGE:latest'
                    sh 'docker push $IMAGE:latest'
                }
            }
        }
        stage('Deploy with Ansible') {
            steps {
                script {
                    sh 'ansible-playbook -i ansible/inventory ansible/playbook.yml'
                }
            }
        }
    }
    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Deployment failed!'
        }
    }
}
```

---

### 🔔 Step 9 — Integrate GitHub Webhook

So Jenkins automatically builds on commits:

1. In GitHub repo → **Settings → Webhooks → Add webhook**  
   - Payload URL: `http://<jenkins-server>:8080/github-webhook/`
   - Content type: `application/json`
   - Trigger: "Just the push event"

2. In Jenkins:  
   - Create a new pipeline job → “Build Triggers” → enable **GitHub hook trigger for GITScm polling**.

---

### 🌀 Step 10 — Full Pipeline Flow

1. Developer commits & pushes code to GitHub.  
2. GitHub notifies Jenkins through webhook.  
3. Jenkins:
   - Pulls new code  
   - Builds Docker image  
   - Pushes to Docker Hub  
   - Runs Ansible playbook to deploy on remote host  
4. The new container replaces the old one on port 5000.  

Visit the remote host IP:  
👉 `http://<target-server-ip>:5000`

You’ll see your freshly deployed application running!

---

## 🎨 Bonus: Visual Pipeline Example
```
[Git Push]
     ↓
  Jenkins CI
     ↓
(1) Checkout Code
(2) Build Docker Image
(3) Push Image to DockerHub
(4) Ansible Deploy
     ↓
 [Docker Host Running Latest App]
```

---

## 🧠 11. Optional Add‑Ons for Real Projects
| Enhancement | Description |
|--------------|-------------|
| **Testing Stage** | Add pytest or unit tests before building the image |
| **Multi‑stage Dockerfile** | Optimize final image size |
| **Blue/Green Deployments** | Run new container then swap ports |
| **Monitoring** | Use Prometheus + Grafana to track container health |
| **Reverse Proxy** | Deploy behind Nginx or Traefik for SSL & routing |
| **AWS integration** | Use EC2, ECR, and CodeDeploy in a similar model |

---

## 📚 Official Documentation References

| Tool | Official Docs |
|------|----------------|
| GitHub Actions / Webhooks | [https://docs.github.com/webhooks](https://docs.github.com/webhooks) |
| Jenkins Pipelines | [https://www.jenkins.io/doc/book/pipeline/](https://www.jenkins.io/doc/book/pipeline/) |
| Docker | [https://docs.docker.com/](https://docs.docker.com/) |
| Ansible | [https://docs.ansible.com/](https://docs.ansible.com/) |
| Docker–Jenkins Integration | [https://plugins.jenkins.io/docker-workflow/](https://plugins.jenkins.io/docker-workflow/) |

---

## 🎯 Summary

By the end of this demo:
- You’ve built a **complete CI/CD chain** integrated with GitHub.  
- Each commit triggers a **Dockerized build** and **Ansible-driven deployment**.  
- You understand **each link of the chain**: SCM → Jenkins → Docker → Ansible → Deployment.

> ☁️ *Freshers who master this setup move from “I know what CI/CD means” to “I’ve actually deployed with CI/CD.” That’s real DevOps confidence.*
