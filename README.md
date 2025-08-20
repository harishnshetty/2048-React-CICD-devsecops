# 2048-React-CICD-devsecops — Setup Guide

This README collects useful commands and links to install common DevOps, CI/CD and security tooling on Ubuntu systems. It has been cleaned up, organized and corrected for clarity. Always review commands before running them and check official product docs for the latest versions.

Table of contents
- Prerequisites
- System update & common packages
- Java
- Jenkins
- Docker
- Docker Scout
- Trivy
- Prometheus
- Node Exporter
- Grafana
- kubectl
- Helm
- Gitleaks
- eksctl
- Terraform

---

## Prerequisites
This guide assumes an Ubuntu/Debian-like environment and sudo privileges.

## System update & common packages
```bash
sudo apt update
sudo apt upgrade -y

# Common tools
sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig
```
Reload bash completion if needed:
```bash
source /etc/bash_completion
```

## Java
Install OpenJDK (choose 17 or 21 depending on your needs / application requirements):
```bash
# Option 1: OpenJDK 17
sudo apt install -y openjdk-17-jdk

# Option 2: OpenJDK 21
sudo apt install -y openjdk-21-jdk
```
Verify:
```bash
java --version
```

## Jenkins
Official docs: https://www.jenkins.io/doc/book/installing/linux/

Example for Debian/Ubuntu (uses the jenkins package signing key and repository):
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo gpg --dearmor -o /etc/apt/keyrings/jenkins-keyring.asc

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```
Initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then open: http://<your-server-ip>:8080

Note: Jenkins requires a compatible Java runtime. Check the Jenkins documentation for supported Java versions.

## Docker
Official docs: https://docs.docker.com/engine/install/ubuntu/

Quick install (official Docker repository):
```bash
sudo apt update
sudo apt install -y ca-certificates curl

# Prepare keyrings dir
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group (log out / in or newgrp to apply)
sudo usermod -aG docker $USER
newgrp docker
docker ps
```
If Jenkins needs Docker access, also add the jenkins user:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
Check Docker status:
```bash
sudo systemctl status docker
```

## Docker Scout
Docs: https://github.com/docker/scout-cli

Install via install script:
```bash
docker login   # provide Docker Hub credentials

mkdir -p ~/.docker/cli-plugins

curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh

# or via pipe
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --

docker scout version
```

## Trivy (vulnerability scanner)
Docs: https://trivy.dev/getting-started/installation/

Install via apt repo:
```bash
sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt-get update
sudo apt-get install -y trivy

trivy --version
```

## Prometheus
Official downloads: https://prometheus.io/download/

Generic install steps (use the current release from the Prometheus download page):
```bash
# Create a prometheus user
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prometheus

# Download the release and extract (replace URL with the latest)
wget -O prometheus.tar.gz "https://github.com/prometheus/prometheus/releases/download/<VERSION>/prometheus-<VERSION>.linux-amd64.tar.gz"
tar -xvf prometheus.tar.gz
cd prometheus-*/

sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

sudo chown -R prometheus:prometheus /etc/prometheus /data
```
Systemd service (`/etc/systemd/system/prometheus.service`):
```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
```
Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```
Access: http://<your-server-ip>:9090

## Node Exporter
Docs: https://prometheus.io/docs/guides/node-exporter/

Example:
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter

wget -O node_exporter.tar.gz "https://github.com/prometheus/node_exporter/releases/download/<VERSION>/node_exporter-<VERSION>.linux-amd64.tar.gz"
tar -xvf node_exporter.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
rm -rf node_exporter*

# systemd service: /etc/systemd/system/node_exporter.service
```
Service file:
```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```
Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```
In Prometheus config add the target (port 9100) for the node exporter.

## Grafana
Docs: https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

Quick install:
```bash
sudo apt-get install -y apt-transport-https software-properties-common wget

sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```
Access: http://<your-server-ip>:3000

## kubectl
Docs: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

Example (using official apt repo for kubectl):
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
```

## Helm
Docs: https://helm.sh/docs/intro/install/

Install via apt:
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install -y apt-transport-https
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install -y helm
```

## Gitleaks
Repo: https://github.com/gitleaks/gitleaks

Run with Docker:
```bash
docker run --rm -v "$(pwd)":/path zricethezav/gitleaks:latest \
  detect --source=/path --report-path=/path/gitleaks-report.json --exit-code 1
```
Install binary (example):
```bash
sudo curl -sSL "https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_$(uname -s)_$(uname -m).tar.gz" -o /tmp/gitleaks.tar.gz
sudo tar -xzf /tmp/gitleaks.tar.gz -C /tmp
sudo mv /tmp/gitleaks /usr/local/bin/gitleaks
sudo chmod +x /usr/local/bin/gitleaks
gitleaks version
```
Example pipeline stage:
```groovy
sh 'gitleaks detect --source . --report-path=gitleaks-report.json --exit-code 1'
```

## eksctl
Docs: https://eksctl.io/installation/

Example:
```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf eksctl_${PLATFORM}.tar.gz -C /tmp && rm eksctl_${PLATFORM}.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

## Terraform
Docs: https://developer.hashicorp.com/terraform/install

Example (HashiCorp apt repo):
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt install -y terraform
```

---

Notes and recommendations
- Replace `<VERSION>` and `<your-server-ip>` placeholders with the specific version or IP you intend to use.
- Where the guide uses "latest" downloads, prefer pinned versions for production environments.
- Check each project's official documentation pages linked above for the most up-to-date installation instructions, supported versions, and security guidance.

```