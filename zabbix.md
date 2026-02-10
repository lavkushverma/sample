Perfect — let’s make Zabbix setup fully GitHub-ready, clean, reusable, and interview/production friendly 💪
Below is a real-world repo structure + files + manual steps you can directly push to GitHub.


---

🚀 Zabbix Setup – GitHub Ready (Production-Style)

📦 What this repo gives you

Zabbix Server + Web + DB (Docker based)

Agent installation (Linux)

Secure defaults

Easy to deploy on EC2 / VM

Interview-ready + real usage



---

📁 Repository Structure

zabbix-setup/
├── docker-compose/
│   ├── docker-compose.yml
│   └── .env
│
├── agent/
│   ├── install-agent.sh
│   └── zabbix_agent2.conf
│
├── scripts/
│   ├── firewall.sh
│   └── backup-db.sh
│
├── docs/
│   ├── architecture.md
│   ├── setup-manual.md
│   └── troubleshooting.md
│
├── .gitignore
├── README.md
└── LICENSE


---

1️⃣ Zabbix Server (Docker Compose)

📄 docker-compose/docker-compose.yml

version: "3.9"

services:
  mysql:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql

  zabbix-server:
    image: zabbix/zabbix-server-mysql:latest
    container_name: zabbix-server
    restart: always
    depends_on:
      - mysql
    environment:
      DB_SERVER_HOST: mysql
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "10051:10051"

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:latest
    container_name: zabbix-web
    restart: always
    depends_on:
      - zabbix-server
    environment:
      DB_SERVER_HOST: mysql
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      PHP_TZ: Asia/Kolkata
    ports:
      - "8080:8080"

volumes:
  mysql-data:


---

📄 docker-compose/.env

MYSQL_ROOT_PASSWORD=StrongRootPass123
MYSQL_PASSWORD=StrongZabbixPass123


---

▶️ Start Zabbix

cd docker-compose
docker compose up -d

Access UI:
👉 http://<EC2-PUBLIC-IP>:8080
Default login:

Username: Admin
Password: zabbix


---

2️⃣ Zabbix Agent (Linux – EC2 / VM)

📄 agent/install-agent.sh

#!/bin/bash
sudo apt update -y
sudo apt install -y zabbix-agent2

sudo systemctl enable zabbix-agent2
sudo systemctl start zabbix-agent2


---

📄 agent/zabbix_agent2.conf

Server=<ZABBIX_SERVER_PRIVATE_IP>
ServerActive=<ZABBIX_SERVER_PRIVATE_IP>
Hostname=linux-node-01

Apply config:

sudo systemctl restart zabbix-agent2


---

3️⃣ Firewall / Security Group Script

📄 scripts/firewall.sh

#!/bin/bash
sudo ufw allow 10050
sudo ufw allow 10051
sudo ufw allow 8080
sudo ufw reload

AWS SG required:

8080 → Web UI

10051 → Zabbix Server

10050 → Agent



---

4️⃣ Database Backup Script

📄 scripts/backup-db.sh

#!/bin/bash
DATE=$(date +%F)
docker exec zabbix-mysql \
mysqldump -u zabbix -pStrongZabbixPass123 zabbix \
> zabbix-backup-$DATE.sql

(For prod → store in S3 using cron)


---

5️⃣ README.md (GitHub Front Page)

# Zabbix Monitoring – Production Ready Setup

## Features
- Dockerized Zabbix Server
- MySQL backend
- Agent-based monitoring
- Secure & scalable
- Easy EC2 deployment

## Quick Start
```bash
git clone https://github.com/yourname/zabbix-setup.git
cd zabbix-setup/docker-compose
docker compose up -d

Access UI: http://<server-ip>:8080
Login: Admin / zabbix

---
 