# 🚀 DevOps Pipeline: Jenkins + Docker + Kubernetes + Prometheus & Grafana

## 📐 Architecture Overview

```
Terminal
  └── Git Push
        └── GitHub Repository
              └── Jenkins (CI/CD)
                    └── Docker Images (Built & Pushed to DockerHub)
                          └── Kubernetes Cluster (EKS)
                                └── Monitoring (Prometheus & Grafana)

CD Tool: ArgoCD via Helm
Ingress: NGINX Ingress Controller (Path-based routing)
```

---

## 📋 Table of Contents

1. [Phase 1 – Git Setup](#phase-1--git-setup)
2. [Phase 2 – EC2 Instance Launch](#phase-2--ec2-instance-launch)
3. [Phase 3 – Jenkins Installation](#phase-3--jenkins-installation)
4. [Phase 4 – Access Jenkins & Install Plugins](#phase-4--access-jenkins--install-plugins)
5. [Phase 5 – Docker Installation](#phase-5--docker-installation)
6. [Phase 6 – Install kubectl, eksctl & AWS CLI](#phase-6--install-kubectl-eksctl--aws-cli)
7. [Phase 7 – Create EKS Cluster](#phase-7--create-eks-cluster)
8. [Phase 8 – Configure Jenkins Credentials & Tools](#phase-8--configure-jenkins-credentials--tools)
9. [Phase 9 – Install NGINX Ingress Controller](#phase-9--install-nginx-ingress-controller)
10. [Phase 10 – Run Jenkins Pipeline & Verify Deployment](#phase-10--run-jenkins-pipeline--verify-deployment)
11. [Phase 11 – Monitoring with Prometheus](#phase-11--monitoring-with-prometheus)

---

## Phase 1 – Git Setup

> Push your local code to a remote GitHub repository.

```bash
# Initialize a local Git repository
git init

# Stage all files
git add ./

# Check the status of staged files
git status

# Commit the changes
git commit -m "msg"

# Add the remote GitHub repository
git remote add origin <your-repo-url>

# Set the remote URL with credentials (token-based auth)
git remote set-url origin https://<username>:<your-github-token>@github.com/<username>/<repo-name>.git

# Push code to the main branch
git push origin main
```

> ⚠️ Replace `<username>` and `<your-github-token>` with your actual GitHub username and Personal Access Token.

### ✅ Phase 1 Summary
Local code is initialized, committed, and pushed to GitHub. Token-based authentication is used for secure remote access.

---

## Phase 2 – EC2 Instance Launch

> Launch an EC2 instance to host Jenkins, Docker, and the Kubernetes CLI tools.

### Recommended Instance Type
- **Type:** `m7i-flex` (preferred for performance)

### Security Group – Ports to Open

| Port Range     | Purpose                          |
|----------------|----------------------------------|
| 22             | SSH access                       |
| 80             | HTTP                             |
| 443            | HTTPS                            |
| 465            | SMTPS (mail)                     |
| 1000–10000     | General application ports        |
| 8080           | Jenkins Web UI                   |
| 30000–32767    | Kubernetes NodePort range        |

### ✅ Phase 2 Summary
EC2 instance launched with the correct instance type and all required ports open in the Security Group.

---

## Phase 3 – Jenkins Installation

> Install Jenkins along with Java (Temurin 17 JDK) on the EC2 instance.

### Step 1: Create the Jenkins install script

```bash
vi jenkins.sh
```

Paste the following content into the file:

```bash
#!/bin/bash
sudo apt update -y

# Install Temurin 17 JDK
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### Step 2: Set permissions and run the script

```bash
sudo chmod +x jenkins.sh
./jenkins.sh
```

### Step 3: If the script above gives errors, run these commands manually

```bash
# Download the updated Jenkins GPG key
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

# Add the Jenkins repo
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt update
sudo apt install jenkins

# Enable and start Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### ✅ Phase 3 Summary
Jenkins is installed with Temurin 17 JDK as the runtime. The service is enabled to auto-start and is running on port 8080.

---

## Phase 4 – Access Jenkins & Install Plugins

> Unlock Jenkins, set credentials, and install all required plugins.

### Step 1: Access Jenkins in your browser

```
http://<your-ec2-public-ip>:8080
```

### Step 2: Get the initial admin password

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the output and paste it into the Jenkins UI to unlock.

### Step 3: Set your admin credentials

| Field     | Value         |
|-----------|---------------|
| Username  | Your choice   |
| Password  | Your choice   |
| Full Name | Your choice   |

### Step 4: Install the following plugins

Navigate to: **Manage Jenkins → Plugins → Available Plugins**

Install all of the following:

| Category       | Plugins                                                                 |
|----------------|-------------------------------------------------------------------------|
| Docker         | Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step  |
| AWS            | AWS Credentials                                                         |
| Pipeline       | Pipeline Stage View                                                     |
| Kubernetes     | Kubernetes, Kubernetes CLI, Kubernetes Client API, Kubernetes Credentials, Config File Provider |
| Monitoring     | Prometheus metrics                                                      |

> 🔄 Restart Jenkins after plugin installation.

### ✅ Phase 4 Summary
Jenkins is accessible via the browser, admin credentials are set, and all required plugins for Docker, Kubernetes, AWS, and Prometheus are installed.

---

## Phase 5 – Docker Installation

> Install Docker on the EC2 instance so Jenkins can build and push images.

### Step 1: Create the Docker install script

```bash
vi docker.sh
```

Paste the following content:

```bash
#!/bin/bash

# Update package manager repositories
sudo apt-get update

# Install dependencies
sudo apt-get install -y ca-certificates curl

# Create directory for Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Set correct permissions for the key
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Verify Docker version
docker --version
```

### Step 2: If errors occur, run these commands manually

```bash
# Add Docker's GPG key
sudo install -m 755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker packages
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
sudo docker run hello-world
```

### ✅ Phase 5 Summary
Docker is installed and verified on the EC2 instance. The `hello-world` test confirms Docker is running correctly.

---

## Phase 6 – Install kubectl, eksctl & AWS CLI

> Install the Kubernetes CLI, EKS CLI, and AWS CLI tools required to manage the cluster.

### Install kubectl

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

### Install AWS CLI

```bash
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Attach IAM Role to EC2 Instance

Navigate to: **EC2 Console → Instance → Actions → Security → Modify IAM Role**

Attach the following managed policies to the IAM role:

- `AmazonEC2FullAccess`
- `AmazonEKS_CNI_Policy`
- `AmazonEKSClusterPolicy`
- `AmazonEKSWorkerNodePolicy`
- `AWSCloudFormationFullAccess`
- `IAMFullAccess`

> ⚠️ Alternatively, you can attach `AdministratorAccess` for simplicity, but this is **not recommended** for production.

### Add Inline Policy for EKS Access

Attach the following inline policy to the IAM role for cluster-level EKS permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
    }
  ]
}
```

> Also configure your AWS Access Key and Secret Key in the EC2 instance using `aws configure` before creating the EKS cluster.

### ✅ Phase 6 Summary
`kubectl`, `eksctl`, and the AWS CLI are installed. The EC2 IAM role has the necessary permissions to create and manage the EKS cluster.

---

## Phase 7 – Create EKS Cluster

> Provision a managed Kubernetes cluster on AWS using eksctl.

```bash
eksctl create cluster \
  --name BlueRay1-cluster \
  --region ap-south-1 \
  --node-type t3.small \
  --zones ap-south-1a,ap-south-1b
```

> ⏳ Cluster creation takes approximately **15–20 minutes**. eksctl uses CloudFormation under the hood.

### ✅ Phase 7 Summary
An EKS cluster named `BlueRay1-cluster` is created in the `ap-south-1` region with `t3.small` worker nodes across two availability zones.

---

## Phase 8 – Configure Jenkins Credentials & Tools

> Set up Docker and GitHub credentials in Jenkins, configure Docker as a tool, and fix group permissions.

### Step 1: Configure Docker in Jenkins Tools

Navigate to: **Manage Jenkins → Tools → Docker Installations → Add Docker → Save**

### Step 2: Add DockerHub Credentials

Navigate to: **Manage Jenkins → System → Global Credentials → Add Credentials**

| Field    | Value            |
|----------|------------------|
| Kind     | Username/Password |
| Username | Your DockerHub username |
| Password | Your DockerHub password/token |
| ID       | `dockerhub_cred` |

> ⚠️ The credential ID `dockerhub_cred` must match exactly what is used in your Jenkinsfile pipeline.

### Step 3: Add GitHub Credentials

Navigate to: **Manage Jenkins → System → Global Credentials → Add Credentials**

| Field    | Value            |
|----------|------------------|
| Kind     | Username/Password |
| Username | Your GitHub username |
| Password | Your GitHub Personal Access Token |
| ID       | `github_cred` |

> ⚠️ The credential ID `github_cred` must match exactly what is used in your Jenkinsfile pipeline.

### Step 4: Fix Docker Group Permissions for Jenkins

```bash
# Add Jenkins user to the Docker group
sudo usermod -aG docker jenkins

# Restart Jenkins to apply group changes
systemctl restart jenkins
```

Refresh the Jenkins browser tab and log back in.

### ✅ Phase 8 Summary
Jenkins is fully configured with DockerHub and GitHub credentials. Docker tool integration is set up. Jenkins has the Docker group permission to build and push images without `sudo`.

---

## Phase 9 – Install NGINX Ingress Controller

> Deploy the NGINX Ingress Controller on the EKS cluster for path-based routing.

```bash
# Apply the NGINX Ingress Controller manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/aws/deploy.yaml

# Wait for pods to be ready (this may take a minute)
kubectl get pods -n ingress-nginx
```

### ✅ Phase 9 Summary
The NGINX Ingress Controller is deployed on the cluster and will provision an AWS Load Balancer. This enables path-based routing for your microservices.

---

## Phase 10 – Run Jenkins Pipeline & Verify Deployment

> Trigger the Jenkins pipeline and verify that pods and services are running.

### Step 1: Configure and run the Jenkins pipeline

1. Create a new Pipeline job in Jenkins
2. Point it to your GitHub repository (using `github_cred`)
3. Trigger the build
<img width="952" height="448" alt="build-success2" src="https://github.com/user-attachments/assets/bc4ca783-a2d8-4df7-bcb2-231283ca035c" />
   

### Step 2: After a successful build, check pods

```bash
kubectl get pods
```
<img width="959" height="382" alt="build-success" src="https://github.com/user-attachments/assets/fa1e9547-fa3f-45ea-8c2b-5e4775b689ad" />
<img width="871" height="230" alt="pods-run" src="https://github.com/user-attachments/assets/73d084e2-d1e9-4a97-bb81-4453201cc946" />

### Step 3: Get the Ingress URL

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

Access the application using the `EXTERNAL-IP` (AWS ELB DNS name) from the output above. This Load Balancer URL enables path-based routing to your microservices.

**Example Ingress URL:**
```
a7aedb2d9a7f94113bb344cb3755101e-9ff7166868077096.elb.ap-south-1.amazonaws.com
```
<img width="938" height="434" alt="applicationpage1" src="https://github.com/user-attachments/assets/238d6900-89d6-46d2-bf85-e3db5b80793f" />


> 🌐 Different application pages are served based on URL paths — this is **path-based routing** via the Ingress resource.

### ✅ Phase 10 Summary
The Jenkins CI/CD pipeline successfully builds Docker images, pushes them to DockerHub, and deploys to the EKS cluster. The application is accessible externally via the NGINX Ingress Load Balancer URL with path-based routing.

---
<img width="959" height="446" alt="applicationpage2" src="https://github.com/user-attachments/assets/3a06de5a-9e8e-4bb9-a849-86d06090a8db" />
## Phase 11 – Monitoring with Prometheus

> Install and configure Prometheus on a monitoring server (can be a separate EC2 instance).

### Step 1: Update the system

```bash
sudo apt update
```

### Step 2: Create a dedicated Prometheus system user

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /bin/false prometheus
```

**Explanation of flags:**

| Flag              | Purpose                                         |
|-------------------|-------------------------------------------------|
| `--system`        | Creates a system account (not a regular user)   |
| `--no-create-home`| No home directory needed for service accounts   |
| `--shell /bin/false` | Prevents interactive login as this user      |

### Step 3: Download and extract Prometheus

```bash
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```

### Step 4: Create required directories

```bash
sudo mkdir -p /data /etc/prometheus
```

### Step 5: Navigate into the extracted directory

```bash
cd prometheus-2.47.1.linux-amd64/
```

### Step 6: Move Prometheus binaries to system path

```bash
# Move prometheus binary and promtool (used to validate config and rules)
sudo mv prometheus promtool /usr/local/bin/
```

### Step 7: Move console libraries and config to Prometheus config directory

```bash
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

### Step 8: Go back to root/home directory

```bash
cd
```

### Step 9: Verify the installation

```bash
prometheus --version
prometheus --help
```

### Step 10: Create the Prometheus systemd service

```bash
sudo vi /etc/systemd/system/prometheus.service
```

Paste the following content into the file:

```ini
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
```

### Step 11: Enable and start Prometheus

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

### Access Prometheus

```
http://<monitoring-server-ip>:9090
```

### ✅ Phase 11 Summary
Prometheus is installed as a system service running under a dedicated non-login user. It is configured to scrape metrics and is accessible on port 9090. This sets the foundation for connecting Grafana dashboards for full observability of the Kubernetes cluster and application.

---
### Home page of Grafana
<img width="938" height="483" alt="grafana-home" src="https://github.com/user-attachments/assets/ef230428-ec3b-4d27-aad7-4ea26dfcbd1b" />
### Dashboards
<img width="959" height="433" alt="grafana3-dashboard-jenkins" src="https://github.com/user-attachments/assets/5bbac909-4c0b-42c5-912c-332916f594dc" />
<img width="959" height="508" alt="grafana3-dashboard-prometheus" src="https://github.com/user-attachments/assets/78af2039-9cb4-4e38-bd1e-9dd9a1f124cb" />


## 🔒 Security Notes

- Never hardcode GitHub tokens or passwords in scripts or pipeline files. Use Jenkins credentials with IDs.
- Use scoped IAM roles over `AdministratorAccess` in production environments.
- Rotate GitHub Personal Access Tokens regularly.

---

## 📌 Quick Reference – Key URLs & Credentials

| Resource         | URL / Value                          |
|------------------|--------------------------------------|
| Jenkins UI       | `http://<ec2-ip>:8080`               |
| Prometheus UI    | `http://<monitoring-ip>:9090`        |
| DockerHub Cred ID | `dockerhub_cred`                   |
| GitHub Cred ID   | `github_cred`                        |
| EKS Cluster Name | `BlueRay1-cluster`                   |
| AWS Region       | `ap-south-1`                         |
| Node Type        | `t3.small`                           |
| Jenkins Initial Password Path | `/var/lib/jenkins/secrets/initialAdminPassword` |
