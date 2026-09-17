# Setup Lab Foundation

Given your validated Latitude 7490:

* **CPU:** Intel 8th Gen mobile platform (suitable)
* **RAM:** 16 GB DDR4-2400 (usable; 32 GB later would help)
* **SSD:** Micron 256 GB SATA SSD (healthy, but capacity will become the first constraint)

I recommend building the lab in **layers**. Do not install everything at once. Each layer should be validated before moving on.

The target architecture:

```text
Ubuntu 24.04 LTS
│
├── System Foundation
│   ├── Updates
│   ├── Build tools
│   ├── Git
│   ├── SSH
│   └── Python environment
│
├── Developer Tools
│   ├── VS Code
│   ├── Python venv
│   ├── Rust
│   └── CLI utilities
│
├── Container Platform
│   ├── Docker Engine
│   ├── Docker Compose
│   └── PostgreSQL container
│
├── Data Engineering
│   ├── dbt
│   ├── Airflow
│   ├── Dagster
│   └── PySpark
│
├── Platform Engineering
│   ├── Terraform
│   ├── Kubernetes tools
│   └── Jenkins
│
└── Cloud Integration
    ├── AWS CLI
    ├── Azure CLI
    └── Google Cloud SDK
```

---

# Phase 1 — Ubuntu Foundation

## 1. Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

Reboot:

```bash
sudo reboot
```

Verify:

```bash
cat /etc/os-release
```

Expected:

```text
Ubuntu 24.04 LTS
```

---

# Phase 2 — Install Base Linux Tools

Install common engineering utilities:

```bash
sudo apt install -y \
build-essential \
curl \
wget \
git \
vim \
nano \
htop \
tree \
jq \
unzip \
zip \
net-tools \
openssh-client \
software-properties-common
```

Verify:

```bash
git --version
python3 --version
curl --version
```

---

# Phase 3 — Configure Git

Set identity:

```bash
git config --global user.name "Jim Sproul"

git config --global user.email "your-email@example.com"
```

Verify:

```bash
git config --list
```

Create workspace:

```bash
mkdir -p ~/projects
cd ~/projects
```

---

# Phase 4 — Install Python Development Environment

Ubuntu already includes Python.

Install development packages:

```bash
sudo apt install -y \
python3-pip \
python3-venv \
python3-dev
```

Verify:

```bash
python3 --version
pip3 --version
```

Create lab virtual environment:

```bash
mkdir -p ~/projects/data-platform-lab

cd ~/projects/data-platform-lab

python3 -m venv .venv
```

Activate:

```bash
source .venv/bin/activate
```

Upgrade Python tools:

```bash
pip install --upgrade pip setuptools wheel
```

Verify:

```bash
python --version
```

---

# Phase 5 — Install VS Code

Install Microsoft repository:

```bash
sudo apt update

sudo apt install -y wget gpg

wget -qO- https://packages.microsoft.com/keys/microsoft.asc \
| gpg --dearmor \
| sudo tee /usr/share/keyrings/packages.microsoft.gpg > /dev/null
```

Add repository:

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" \
| sudo tee /etc/apt/sources.list.d/vscode.list
```

Install:

```bash
sudo apt update
sudo apt install code -y
```

Launch:

```bash
code
```

Recommended extensions:

* Python
* Pylance
* Docker
* YAML
* GitLens
* SQLTools
* PostgreSQL

---

# Phase 6 — Install Docker Engine

Remove old versions:

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

Install prerequisites:

```bash
sudo apt install -y \
ca-certificates \
curl \
gnupg
```

Add Docker key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor \
-o /etc/apt/keyrings/docker.gpg
```

Add repository:

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list
```

Install:

```bash
sudo apt update

sudo apt install -y \
docker-ce \
docker-ce-cli \
containerd.io \
docker-compose-plugin
```

Add yourself to Docker group:

```bash
sudo usermod -aG docker $USER
```

Logout/login.

Test:

```bash
docker run hello-world
```

---

# Phase 7 — Create Lab Directory Structure

```bash
cd ~/projects/data-platform-lab

mkdir -p \
ingestion/python \
ingestion/rust \
database/postgres \
database/sql \
dbt/models \
airflow/dags \
spark/jobs \
spark/tests \
dagster \
jenkins \
terraform \
kubernetes \
data/raw \
data/staging \
data/output \
tests
```

Verify:

```bash
tree
```

---

# Phase 8 — PostgreSQL Container

Create:

```bash
cd database/postgres

nano docker-compose.yml
```

Add:

```yaml
services:

  postgres:
    image: postgres:16
    container_name: lab-postgres
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: datalab
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Start:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Connect:

```bash
docker exec -it lab-postgres psql -U devuser -d datalab
```

Test:

```sql
select version();
```

Exit:

```sql
\q
```

---

# Phase 9 — Install dbt

Activate Python environment:

```bash
cd ~/projects/data-platform-lab

source .venv/bin/activate
```

Install:

```bash
pip install dbt-postgres
```

Verify:

```bash
dbt --version
```

---

# Phase 10 — Install Airflow

Create environment:

```bash
python -m venv ~/airflow-env

source ~/airflow-env/bin/activate
```

Set constraint:

```bash
export AIRFLOW_VERSION=2.10.2
export PYTHON_VERSION=3.12
```

Install:

```bash
pip install apache-airflow
```

Initialize:

```bash
airflow db init
```

---

# Phase 11 — Install Rust

```bash
curl https://sh.rustup.rs -sSf | sh
```

Reload:

```bash
source ~/.cargo/env
```

Verify:

```bash
rustc --version
cargo --version
```

---

# Phase 12 — Install Terraform

```bash
sudo apt install terraform -y
```

Verify:

```bash
terraform version
```

---

# Phase 13 — Install Kubernetes Tools

Install kubectl:

```bash
sudo snap install kubectl --classic
```

Install Minikube:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Verify:

```bash
kubectl version --client
```

---

# Phase 14 — Install Cloud CLIs

## AWS

```bash
sudo apt install awscli -y
```

Verify:

```bash
aws --version
```

## Azure

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Verify:

```bash
az version
```

## Google Cloud

Install later when needed.

---

# Phase 15 — Create First Pipeline Test

Your first milestone:

```text
CSV File
   |
   v
Python Ingestion
   |
   v
PostgreSQL Raw Schema
   |
   v
dbt Transformation
   |
   v
Analytics Tables
```

Directory:

```text
data-platform-lab/

ingestion/python/load_csv.py

database/postgres/docker-compose.yml

dbt/models/
```

---

## Recommended order for your machine

Because you have **16 GB RAM / 256 GB SSD**, I would stop after:

1. Ubuntu updates ✅
2. Git ✅
3. Python ✅
4. VS Code ✅
5. Docker ✅
6. PostgreSQL container ✅
7. dbt ✅

Then validate disk usage:

```bash
df -h
```

and memory:

```bash
free -h
```

Before adding:

* Airflow
* Spark
* Kubernetes
* Jenkins

Those are the components most likely to consume RAM and disk.

This creates a realistic **Data Engineer → Platform Engineer → Cloud Engineer progression** without overwhelming the Latitude 7490.
