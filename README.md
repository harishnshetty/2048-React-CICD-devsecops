# 2048-React-CICD-devsecops — Setup Guide

This README collects useful commands and links to install common DevOps, CI/CD and security tooling on Ubuntu systems. It has been cleaned up, organized and corrected for clarity. Always review commands before running them and check official product docs for the latest versions.

Table of contents
- Prerequisites
- System update & common packages
- Java
- Jenkins
- Docker
- Trivy
- Prometheus
- Node Exporter
- Grafana
- kubectl
- Helm
- eksctl


---
plugins need to install on jenkins

Prometheus metrics plugin
Docker API Plugin
Docker Commons Plugin
Docker Pipeline
Docker plugin
docker-build-step
Eclipse Temurin installer Plugin
Email Extension Plugin
OWASP Dependency-Check Plugin
Pipeline: Stage View Plugin
SonarQube Scanner for Jenkins


Ports need to enable in the SG
ssh                     22
sonarqube               9000
promethues              9090
node_exporter           9100
grafana                 3000
http                    80
https                   443

-------------credational to store

               ID 
mail           mail-cred         username and app password
sonarqube      sonar-token       secret text (take from the sonarqube application) 
docker         docker-cred       secret text (take from your docker hub profile)

http://<jenkins-ip>:8080/sonarqube-webhook/  

---tools configuration
jdk
node
SonarQube Scanner installations
Maven installations
Dependency-Check installations
Docker installations



--- system configuation
SonarQube servers   
Name:   sonar-server
url:    http://10.59.10.239:9000
cred-add

Extended E-mail Notification
SMTP server                     smtp.gmail.com
SMTP Port                       465
                                Use SSL
Default user e-mail suffix      @gmail.com


E-mail Notification
SMTP server                     smtp.gmail.com
Default user e-mail suffix      @gmail.com
advanced
Use SMTP Authentication
User Name                       example@gmail.com
password-credations
                                Use TLS
SMTP Port                       587
Reply-To Address                example@gmail.com    





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



# apt-get install git
For Ubuntu, this PPA provides the latest stable upstream Git version

# add-apt-repository ppa:git-core/ppa
# apt update; apt install git



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
nano /etc/prometheus/prometheus.yml

 - job_name: "node_exporter"
    static_configs:
      - targets: ["10.59.10.239:9100"]

  - job_name: "jenkins"
    metrics_path: /prometheus
    static_configs:
      - targets: ["10.59.10.239:8080"]


promtool check config /etc/prometheus/prometheus.yml

curl -X POST http://localhost:9090/-/reload



 


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
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
Access: http://<your-server-ip>:3000







# EKS ALB Ingress Kubernetes Setup Guide

This guide covers the step-by-step installation and setup process for AWS CLI, `kubectl`, `eksctl`, and `helm`, along with instructions to create and configure an EKS cluster with AWS Load Balancer Controller.

---

## 1. AWS CLI Installation

Refer: [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## 2. kubectl Installation
git 
Refer: [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
    
  # If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

sudo apt-get update
sudo apt-get install -y kubectl
```

---

## 3. eksctl Installation

Refer: [eksctl Installation Guide](https://eksctl.io/installation/)

```bash
# For ARM systems, set ARCH to: arm64, armv6, or armv7
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

---

## 4. Helm Installation

Refer: [Helm Installation Guide](https://helm.sh/docs/intro/install/)

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

---

## 5. AWS CLI Configuration

```bash
aws configure
```
```bash
aws configure list
```
---

## 6. Create EKS Cluster and Nodegroup

```bash
eksctl create cluster \
  --name my-cluster \
  --region ap-south-1 \
  --version 1.33 \
  --without-nodegroup

eksctl create nodegroup \
  --cluster my-cluster \
  --name my-nodes \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 6 \
  --node-type t3.medium
```

---

## 7. Update kubeconfig

```bash
aws eks update-kubeconfig --name my-cluster --region ap-south-1
```

---

## 8. Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve
```

---

## 9. Create IAM Policy for AWS Load Balancer Controller

	link for new updated policy----->	https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

---

## 10. Create IAM Service Account

Replace `<ACCOUNT_ID>` with your AWS account ID.

```bash
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region ap-south-1 \
  --approve
```

---

## 11. Install AWS Load Balancer Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks



helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --version 1.13.3
```

**Optional:** List available versions of the chart



```bash
helm search repo eks/aws-load-balancer-controller --versions
```


```bash
helm list -A
```

**Verify installation:**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```



---

## 12. Create and Set Namespace for Your Application

```bash
kubectl apply -f namespace.yml
kubectl apply -f deployment.yml
kubectl apply -f ingress-alb-80-without-acm.yml
kubectl config set-context --current --namespace=store-ns
```

---


 
## 13. Delete EKS Cluster (Cleanup)   [stop here if you tired ]

```bash
eksctl delete cluster --name my-cluster --region ap-south-1
```



Monitor Kubernetes with Prometheus
Install Node Exporter using Helm

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

kubectl create namespace prometheus-node-exporter

helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
nano /etc/prometheus/prometheus.yml

  - job_name: 'k8s'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['node1Ip:9100']


promtool check config /etc/prometheus/prometheus.yml

curl -X POST http://localhost:9090/-/reload



Installing Argo CD              https://www.eksworkshop.com/docs/automation/gitops/argocd/access_argocd


helm repo add argo-cd https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo-cd/argo-cd --version "${ARGOCD_CHART_VERSION}" \
  --namespace "argocd" --create-namespace \
  --values ~/environment/eks-workshop/modules/automation/gitops/argocd/values.yaml \
  --wait


export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname')
echo "ArgoCD URL: https://$ARGOCD_SERVER"

The load balancer will take some time to provision. Use this command to wait until ArgoCD responds:

curl --head -X GET --retry 20 --retry-all-errors --retry-delay 15 \
  --connect-timeout 5 --max-time 10 -k \
  https://$ARGOCD_SERVER


For authentication, the default username is admin and the password is auto-generated. Retrieve the password with the following command:

export ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGOCD_PWD"


---

Notes and recommendations
- Replace `<VERSION>` and `<your-server-ip>` placeholders with the specific version or IP you intend to use.
- Where the guide uses "latest" downloads, prefer pinned versions for production environments.
- Check each project's official documentation pages linked above for the most up-to-date installation instructions, supported versions, and security guidance.



## 13. Delete EKS Cluster (Cleanup)

```bash
eksctl delete cluster --name my-cluster --region ap-south-1
```

---
























```