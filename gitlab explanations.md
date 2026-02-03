---

🔹 Option 1: Use GitLab Cloud (Easiest)

✅ Step 1: Create GitLab Account

1. Go to gitlab.com


2. Sign up with email / Google


3. Login → Dashboard




---

✅ Step 2: Create a Project

1. Click New Project


2. Select Blank project


3. Enter:

Project name

Visibility: Private / Public



4. Click Create project




---

✅ Step 3: Push Code to GitLab

git clone https://gitlab.com/username/project.git
cd project
git add .
git commit -m "first commit"
git push origin main


---

🔹 Option 2: Install GitLab on Server (EC2 / VM)

✅ Step 1: Server Requirement

OS: Ubuntu 20.04 / 22.04

RAM: 4 GB minimum

Open ports: 22, 80, 443



---

✅ Step 2: Install GitLab

sudo apt update
sudo apt install -y curl openssh-server ca-certificates
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
sudo apt install gitlab-ee


---

✅ Step 3: Configure URL

sudo nano /etc/gitlab/gitlab.rb

Set:

external_url "http://your_server_ip"

Apply config:

sudo gitlab-ctl reconfigure

Login:

http://your_server_ip


---

🔹 Configure GitLab CI/CD (Very Important)

✅ Step 4: Create .gitlab-ci.yml

Example:

stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Build started"

test-job:
  stage: test
  script:
    - echo "Testing..."

deploy-job:
  stage: deploy
  script:
    - echo "Deploying..."

Push file → pipeline runs automatically 🚀


---

🔹 Setup GitLab Runner

✅ Step 5: Install Runner

sudo apt install gitlab-runner

✅ Step 6: Register Runner

sudo gitlab-runner register

Enter:

GitLab URL

Registration token (Project → Settings → CI/CD → Runners)

Executor: shell or docker



---

🔹 Common Configurations

🔐 Add Environment Variables

Project → Settings → CI/CD → Variables

AWS_ACCESS_KEY
AWS_SECRET_KEY

👥 Add Users

Project → Members → Add user → Role

🔑 SSH Key

Profile → SSH Keys → Add key


---

🔹 Typical GitLab Flow

Code Push → Pipeline Trigger → Build/Test → Deploy


---
