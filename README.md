# MERN Stack Three-tier Application End-to-end DevsecOps Implementation.

**This project demonstrates an end-to-end DevOps implementation for deploying and managing a MERN stack three-tier application. The pipeline incorporates infrastructure setup, CI/CD processes, application deployment, and monitoring on AWS EKS (Elastic Kubernetes Service).**

### Tools and Technologies

- **Infrastructure as Code (IaC):** Terraform.
- **CI/CD:** Jenkins and ArgoCD.
- **Cloud Provider:** AWS (EKS, EC2, Route 53, S3, etc.).
- **Containerization:** Docker.
- **Monitoring:** Prometheus and Grafana.
- **Code Quality Analysis:** SonarQube.

### Project Workflow

- **Infrastructure Automation**: Using Terraform to set up VPC, EKS cluster, and other resources.
- **CI/CD Pipeline**: 
  - **Jenkins**: Continuous Integration.
  - **ArgoCD**: Continuous Delivery.
- **Application Deployment**: Deploying a three-tier MERN stack application (frontend, backend, and MongoDB) with Ingress and custom domain.
- **Monitoring**: Prometheus for metrics collection and Grafana for visualization.

![Three-Tier Banner](assets/three-tier-workflow.gif)

### Repository Structure

```plaintext
├── terraform
│   ├── modules
│   │   ├── eks
│   │   ├── vpc
│   │   ├── iam
│   └── main.tf
├── manifests
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   ├── mongo-deployment.yaml
│   └── ingress.yaml
├── helm
│   └── monitoring
│       ├── prometheus
│       └── grafana
├── Jenkinsfile
├── README.md
```

## 1. Infrastructure Setup with Terraform

### Objectives
Automate the setup of:
- A private **AWS VPC**.
- An **EKS Kubernetes cluster** with worker nodes.
- A **Jump server** to access the private EKS cluster.

### Steps

- Create a Terraform configuration file (`main.tf`) to define your infrastructure.
- Define the AWS provider and necessary resources such as VPC, subnets, and security groups.
- Create an EC2 instance for Jenkins and other necessary instances.
- Create an EKS Cluster using Terraform.
- Output the necessary details like VPC ID, Subnet IDs, and EKS Cluster details.
- Initialize Terraform and apply the configuration.

#### Step 1: Launch an EC2 Instance for Jenkins

- Open the AWS Console and navigate to **EC2**.

- Launch an Ubuntu-based EC2 instance with the following configurations:
   - Instance type: `t2.medium` or larger.
   - Assign the instance an **IAM role** with administrator access (not recommended for production; follow least-privilege practices).
   - Allow the following ports in the security group:
      - 8080 (Jenkins)
      - 9090 (SonarQube)
      - 3000 (Grafana)
      - 22 (for SSH access, if needed).
   
#### Step 2: Add the following **user-data script** to install necessary tools:

```bash
#!/bin/bash

# Update and upgrade packages:
sudo apt update && sudo apt upgrade -y

# Install **Java** (required for Jenkins and SonarQube):
sudo apt install openjdk-11-jdk -y

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ |sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Install Docker:
sudo apt install docker.io -y
sudo usermod -aG docker $USER
sudo systemctl start docker

# Install Terraform:
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyringshashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorpcom $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y

# Install AWS CLI:
sudo apt install awscli -y

# Install SonarQube using Docker:
sudo docker run -d --name sonarqube -p 9000:9000 sonarqube:lts
```

#### Step 3: Write Terraform Scripts

- Write modular Terraform configurations to:
   - Create a **VPC** with public and private subnets, route tables, and gateways.
   - Deploy an **EKS cluster** with IAM roles, worker nodes, and OIDC.

- Update variables in the `dev.tfvars` file to suit your environment:
   ```hcl
   region = "us-east-1"
   cluster_name = "dev-eks-cluster"
   node_instance_type = "t3.medium"
   ```

- Run Terraform via Jenkins: Create a Jenkins pipeline with the following steps:
   - `terraform init`
   - `terraform validate`
   - `terraform plan`
   - `terraform apply -var-file=dev.tfvars`

#### Step 4: Configure the Jump Server

```bash
# Launch another EC2 instance in the same VPC to act as the Jump server.

# Install necessary tools using user-data scripts:
sudo apt update && sudo apt install -y awscli kubectl helm
   

# Verify access to the private EKS cluster:
aws eks update-kubeconfig --name dev-eks-cluster --region us-east-1
kubectl get nodes
```

## 2. CI/CD Pipeline

Jenkins pipeline setup:
   - Code checkout from GitHub.
   - Code quality analysis using SonarQube.
   - Dependency checks using OWASP.
   - Image security scanning using Trivy
   - Container image creation and scanning.
   - Image pushing to Amazon ECR.
   - Kubernetes manifest update with the new image version.
   - ArgoCD configuration for Continuous Delivery.

### Continuous Integration (Jenkins)

#### Step 1: Setup Jenkins

1. Access Jenkins on `http://<jenkins-ec2-public-ip>:8080`.
2. Install the following plugins:
   - AWS Credentials.
   - Pipeline.
   - SonarQube Scanner.
3. Configure credentials in Jenkins:
   - Add AWS Access Key and Secret Key as a secret text credential.
   - Add a SonarQube token as another secret text credential.

#### Step 2: Configure SonarQube

1. Access SonarQube on `http://<jenkins-ec2-public-ip>:9000`.
2. Login with default credentials (`admin`/`admin`).
3. Create projects for `frontend` and `backend`.
4. Generate and save tokens for each project.

#### Step 3: Create Jenkins Pipelines

1. Apply a **Pipeline Job** for the backend:
   - Add a `Jenkinsfile` with the following stages:  
     - Pulling code from GitHub.
     - Performing code quality analysis using SonarQube.
     - Scanning dependencies with OWASP.
     - Building Docker images for **frontend** and **backend**.
     - Pushing images to **AWS ECR**.
     - Scanning images with **Trivy**.
     - Updating Kubernetes manifests with new image tags.
2. Repeat the steps for the frontend pipeline.

### Continuous Delivery (ArgoCD)

#### Step 1: Install ArgoCD

```bash
# Create a namespace for ArgoCD:
kubectl create namespace argocd

# Install ArgoCD using its manifest:
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   
# Expose the ArgoCD server:
kubectl edit svc argocd-server -n argocd

# Change `type: ClusterIP` to `type: LoadBalancer`.

# Login using the default credentials (admin password is stored in a secret).
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode

# Access ArgoCD at the LoadBalancer URL.
```

#### Step 2: Create Applications

```bash
# Configure ArgoCD to sync with a GitHub repository containing Kubernetes manifests.

# Create an ArgoCD application for the backend: 
kubectl apply -f backend-app.yaml

# Repeat the steps for frontend, MongoDB, and Ingress resources.
kubectl apply -f frontend-app.yaml
kubectl apply -f mongodb-app.yaml
kubectl apply -f ingress-app.yaml
```

## 3. Custom Domain and Ingress Configuration

### Configure Route 53

- Create a hosted zone for your domain in AWS Route 53.
- Update your domain registrar's name servers to point to the Route 53 hosted zone.

### Ingress Controller

```bash
# Install AWS Load Balancer Controller using Helm:
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller -n kube-system

# Link the Load Balancer Controller with EKS using an IAM role and policies.

# Apply Ingress resource `ingress.yaml` to route traffic to the application pods.
```

### Deploy the Application

- Use Kubernetes manifests to deploy frontend, backend, and MongoDB pods.
- Use ConfigMaps and Secrets for environment variables.

### TLS Configuration

- Add SSL certificates (optional) to secure the application.

## 4. Monitoring with Prometheus and Grafana

Use **Helm** to deploy:
- **Prometheus**: For collecting Kubernetes metrics.
- **Grafana**: For visualizing metrics.

### Installation

```bash
# Install Prometheus:
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
# OR
#  install prometheus prometheus-community/prometheus -n monitoring --create-namespace

helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana -n monitoring
```

### Configure Monitoring

- Access Grafana and add Prometheus as a data source.
- Import pre-built Kubernetes dashboards from Grafana's library.

### Observability

- Visualize application performance metrics using Grafana dashboards.
- Set up alerts for performance thresholds using Prometheus.

## 5. Final Testing and Validation

1. Commit changes to the GitHub repository.
2. Verify the Jenkins pipeline execution and deployment in ArgoCD.
3. Access the application using the custom DNS.
4. Validate metrics in Grafana dashboards.

## 6. Troubleshooting and Best Practices

### Common Issues
- **EKS Cluster Connectivity:** Ensure kubeconfig is correctly set up.
- **Pod Failures:** Check pod logs and describe resources (`kubectl describe pod`).
- **Load Balancer Issues:** Verify Ingress configurations and IAM roles.

### Best Practices
- Follow the principle of least privilege for IAM roles.
- Enable SSL for secure communication.
- Use Helm or GitOps practices for Kubernetes resource management.

## References
- [Terraform Documentation](https://www.terraform.io/docs)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
