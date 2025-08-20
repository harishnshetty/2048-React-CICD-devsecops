sudo apt update
sudo apt install upgrade-system -y

sudo apt install bash-completion -y
source /etc/bash_completion
sudo apt install wget git zip unzip curl jq net-tools build-essential ca-certificates -y
sudo apt-get install wget apt-transport-https gnupg fontconfig -y

sudo apt install openjdk-17-jdk -y 
or
sudo apt install openjdk-21-jdk -y

java --version
---------------->
 Jenkins installation LTS (jenkins need openjdk-21-jre )
--------------->   			
				https://www.jenkins.io/doc/book/installing/linux/

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins


sudo cat /var/lib/jenkins/secrets/initialAdminPassword


http://<ip-address>:8080


---------------->
 Docker installation
--------------->   			https://docs.docker.com/engine/install/ubuntu/

# Add Docker's official GPG key:
sudo apt-get update 
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker $USER
newgrp docker
docker ps

sudo usermod -aG docker jenkins && newgrp docker
sudo systemctl restart jenkins


sudo systemctl status docker




---------------->
 Docker Scout installation
--------------->   			https://docs.docker.com/engine/install/ubuntu/
					https://github.com/docker/scout-cli

docker login       `Give Dockerhub credentials here`

mkdir -p ~/.docker/cli-plugins

curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh

or

curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --

docker scout version
---------------->
 trivy installation
--------------->   			https://trivy.dev/v0.55/getting-started/installation/

sudo apt-get install wget apt-transport-https gnupg -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

trivy --version



---------------->
Prometheus installation
--------------->   			https://prometheus.io/download/


sudo useradd --system --no-create-home --shell /bin/false prometheus
wget -O prometheus.tar.gz https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz


tar -xvf prometheus*.tar.gz
cd prometheus*/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

cd ~

sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
sudo nano /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

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
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus



http://<your-server-ip>:9090


---------------->
 Node Exporter installation
--------------->   			https://prometheus.io/download/

cd ~
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget -O node_exporter.tar.gz https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz

tar -xvf node_exporter*.tar.gz
sudo mv node_exporter*/node_exporter /usr/local/bin/
rm -rf node_exporter*


sudo nano /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload


http://<your-prometheus-ip>:9090/targets




---------------->
Grafana installation
--------------->   			https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/


sudo apt-get install -y apt-transport-https software-properties-common wget

sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list




sudo apt-get update

sudo apt-get -y install grafana

sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server


http://<your-server-ip>:3000



--------------> node_exporter of the promethues for k8s
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    
    kubectl create namespace prometheus-node-exporter
    
    helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
    

      - job_name: 'k8s'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['node1Ip:9100']
      
   
   
---------------->
kubectl installation
--------------->   			https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
   
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
    
  # If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

sudo apt-get update
sudo apt-get install -y kubectl




---------------->
helm installation
--------------->   			https://helm.sh/docs/intro/install/


curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm



---------------->
gitleaks installation
--------------->   			https://github.com/gitleaks/gitleaks/releases

sh '''
docker run --rm -v $(pwd):/path zricethezav/gitleaks:latest \
  detect --source=/path --report-path=/path/gitleaks-report.json --exit-code 1
'''



# 1. Download the latest Gitleaks release tarball
sudo curl -sSL https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_$(uname -s)_$(uname -m).tar.gz -o /tmp/gitleaks.tar.gz

# 2. Extract the tarball to /tmp
sudo tar -xvzf /tmp/gitleaks.tar.gz -C /tmp

# 3. Move the binary into /usr/local/bin (accessible to all users)
sudo mv /tmp/gitleaks /usr/local/bin/gitleaks

# 4. Make sure it's executable
sudo chmod +x /usr/local/bin/gitleaks

# 5. Verify installation
gitleaks version



pipeline stage
sh 'gitleaks detect --source . --report-path=gitleaks-report.json --exit-code 1'


---------------->
eksctl installation
--------------->   			https://eksctl.io/installation/#prerequisite


# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl






---------------->
terrafrom installation
--------------->   			https://developer.hashicorp.com/terraform/install#linux


wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y







