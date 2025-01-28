# MERN Stack Three-tier Application End-to-end DevsecOps Implementation.

*This project demonstrates an end-to-end DevOps implementation for deploying and managing a MERN stack three-tier application on AWS infrastructure.*

![Index 1 Banner](assets/index-1.jpg)

![Index 2 Banner](assets/index-2.jpg)

## Project Overview

### 1. Infrastructure Setup:

- Use Terraform to automate:

    - Private VPC.
    - EKS Cluster with two worker nodes.
    - Bastion (Jump) Server for secure EKS management.

- Jenkins is manually installed on an EC2 instance to trigger Terraform scripts.

### 2. CI/CD Pipeline:

- Jenkins CI includes:

    - Code checkout from GitHub.
    - Code quality analysis using SonarQube.
    - Dependency and file scanning.
    - Docker image creation and pushing to ECR.
    - Container image scanning with Trivy.
    - Kubernetes manifest updates.

- Argo CD handles continuous delivery to deploy the updated application to the EKS cluster.

### 3. Ingress Controller, Custom Domain & Monitoring:

- Configure AWS Application Load Balancer (ALB) for the EKS cluster.

- Custom DNS mapping and integration with AWS Route 53.

- Metrics collection using Prometheus.

- Visualization with Grafana.

![Three-Tier Banner](assets/three-tier-workflow.gif)

## Step 1. Create a Jenkins server on Ec2

- Create a security group allowing only ports `8080` (jenkins) and `9090` (sonarqube).

- Launch an instance `Jenkins Server` with the following details:

    - Ubuntu image
    - t2.2xlarger
    - Proceed without a key-pair
    - Default Vpc
    - Select the previously created security group
    - Storage: 30GB
    - Advanced: 
    - Select/Create an IAM role with `EC2` -> `Administrative-Access`
    - User Data: Enter the script `Jenkins-Server-TF/tools-install.sh`

- Connect to the instance using `Session Manager`; 

    ```bash
    # Verify

    sudo su ubuntu
    cd
    sudo htop
    java --version
    jenkins --version
    docker --version
    docker ps
    terraform --version
    whereis terraform   # Output: /usr/bin/terraform
    trivy --version
    kubectl version --client
    aws --version
    eksctl version
    helm version
    ```

- Access Jenkins UI at `<jenkins-server-public-ip>:8000`

    - Get the password using `systemctl status jenkins.service`
    - Install suggested plugins
    - Create First Admin User

## Step 2. Create Infrastructure on Aws using Terraform

- Install necessary plugins on Jenkins server at `Dashboard` -> `Manage Jenkins` -> `Plugins` -> `Available Plugins`.

    - `Aws Credentils`
    - `Pipeline: Aws Steps`
    - `Terraform`
    - `Pipeline: Stage View`

- Store Aws credentials at `Dashboard` -> `Manage Jenkins` -> `Credentials` -> `global` -> `Add Credentials`.

    - Kind: AWS Credentials
    - Scope: Global (Jenkins, node, items, child items, etc) 
    - ID: aws-creds 
    - Description: aws-creds

- Configure Terraform at `Dashboard` -> `Manage Jenkins` -> `Tools` -> `Terraform Installations` -> `Add Terraform` 

    - Name: terraform
    - Install directory: /usr/bin/terraform

- Create a pipeline at `Dashboard` -> `Create a Job` -> `Pipeline` -> `Pipeline Script`.

    - Get the script from `https://github.com/aqeeladil/https://github.com/aqeeladil/mern-end-to-end-devops-implementation/blob/main/Infrastructure/Jenkinsfile`.
    - Click on `Build now`.
    - Click on `Build with Parameters` -> `Apply` -> `Build`.
    - Now, after few minutes, EKS cluster `dev-medium-eks-cluster` will be created in the configured VPC.

## Step 3. Configure `Jenkins Server` for Eks Cluster 

```bash
aws configure
aws eks update-kubeconfig --name dev-medium-eks-cluster --region us-east-1 

# Verify
kebectl get nodes   # here you will not be able to access the cluster because its in a private subnet inside a different Vpc.
```

## Step 4. Create a Jump Server in the same VPC as Eks Cluster.

- As Eks cluster will be created in the private subnet, so we need a jump server in the same vpc to access it from the outside (`Jenkins Server`).

- Launch an EC2 Instance `Jump Server` with the following details:

    - Ubuntu image
    - t2.medium
    - Proceed without a key-pair
    - Vpc: `dev-medium-vpc`.
    - Subnet: `dev-medium-subnet-public-1`
    - Create a default security group
    - Storage: 30GB
    - Advanced: 
    - Select/Create an IAM role with `EC2` -> `Administrative-Access`
    - User Data: Enter the script `Infrastructure/jump-tools.sh`

- Connect to the instance using `Session Manager`; 

    ```bash
    # Verify

    sudo su ubuntu
    cd
    sudo htop
    kubectl version --client
    aws --version
    eksctl version
    helm version
    ```

- Access the Eks Cluster

    ```bash
    aws configure
    aws eks update-kubeconfig --name dev-medium-eks-cluster --region us-east-1 

    # Verify 
    kubectl get nodes
    kubectl get all
    ```

- Create a Service Account using the already created IAM Policy and OIDC Provider.

    ```bash
    eksctl create iamserviceaccount \
        --cluster=dev-medium-eks-cluster \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --role-name AmazonEKSLoadBalancerControllerRole \  --attach-policy-arn=arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy \
        --region=us-east-1 \
        --approve \
        --override-existing-serviceaccounts

    kubectl get sa -n kube-system      # Output: aws-load-balancer-controller
    ```

- Deploy the AWS Load Balancer Controller using Helm

    ```bash
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update eks
    helm repo list

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=dev-medium-eks-cluster \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller \

    # Verify
    kubectl get deployment -n kube-system aws-load-balancer-controller
    ```

- Install & Configure ArgoCD using Helm

    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

    # Verify
    kubectl get pods -n argocd

    # Expose the argoCD server as LoadBalancer
    kubectl get svc -n argocd    # Output: argocd-server
    # kubectl edit svc argocd-server -n argocd
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

    # You can validate visiting the `loadbalancer` service on the AWS Console.
    # To access the argoCD, copy the LoadBalancer DNS and hit on your favorite browser.

    # Username: admin
    # Get the password
    kubectl get secrets -n argocd
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
    ```

## Step 5. Configure Sonarqube on the `Jenkins Server`

- Access Sonarqube at `http://<jenkins-server-public-ip>:9000/`

    - The username and password will be admin

    - Update the password

- `Administration` -> `Security` -> `Users` -> `Update tokens` -> `Generate`

    - Copy the token `three-tier` & keep it somewhere safe.

- Configure webhooks for quality checks.

    - `Administration` -> `Configuration` -> `Webhooks` -> `Create`

        - Name: jenkins
        - URL: `http://<jenkins-server-public-ip>:8080/sonarqube-webhook/`

- Create a Project for Frontend code.

    - `http://<jenkins-server-public-ip>:9000/projects/create`

    - Click on `Manually`

        - Project display name: three-tier-frontend
        - Project key: three-tier-frontend
        - Branch: main

    - `Setup` -> `Locally` -> `Use existing token` -> `Continue` -> `Other` -> `Linux`.

    - Copy the provided command and use it in the Jenkins Frontend Pipeline where Code Quality Analysis will be performed.

- Create a Project for Backend code.

    - `http://<jenkins-server-public-ip>:9000/projects/create`

    - Click on `Manually`

        - Project display name: three-tier-backend
        - Project key: three-tier-backend
        - Branch: main

    - `Setup` -> `Locally` -> `Use existing token` -> `Continue` -> `Other` -> `Linux`.

    - Copy the provided command and use it in the Jenkins Backend Pipeline where Code Quality Analysis will be performed.

- Store the Sonar Credentials in Jenkins.

    - `Dashboard` -> `Manage Jenkins` -> `Credentials` -> `Global`.   

        - Kind: Secret text
        - Scope: Global (Jenkins, node, items, child items, etc) 
        - ID: sonar-token
        - Description: sonar-token

## Step 6. Create & Configure ECR Repositories.

- Create two `Private` repositories `backend` and `frontend` on AWS ECR.

- Add AWS Account ID in the Jenkins credentials because of the ECR repo URI.

    - `Dashboard` -> `Manage Jenkins` -> `Credentials` -> `Global`.   

        - Kind: Secret text
        - Scope: Global (Jenkins, node, items, child items, etc) 
        - ID: ACCOUNT_ID
        - Description: ACCOUNT_ID

- Provide the ECR image name for frontend which is `frontend` only.

    - `Dashboard` -> `Manage Jenkins` -> `Credentials` -> `Global`.   

        - Kind: Secret text
        - Scope: Global (Jenkins, node, items, child items, etc) 
        - Secret: frontend
        - ID: ECR_REPO1
        - Description: ECR_REPO1

- Provide the ECR image name for the backend which is `backend` only.

    - `Dashboard` -> `Manage Jenkins` -> `Credentials` -> `Global`.   

        - Kind: Secret text
        - Scope: Global (Jenkins, node, items, child items, etc)
        - Secret: backend
        - ID: ECR_REPO2
        - Description: ECR_REPO2

- Store the GitHub Personal access token in Jenkins credentials to push the deployment file which will be modified in the pipeline itself for the ECR image.

    - `Dashboard` -> `Manage Jenkins` -> `Credentials` -> `Global`.   

        - Kind: Secret text
        - Scope: Global (Jenkins, node, items, child items, etc)
        - Username: aqeeladil
        - ID: github
        - Description: github

## Step 7: Install & Configure the required Plugins

- `Dashboard` -> `Manage Jenkins` -> `Plugins` -> `Available Plugins`

    ```bash
    Docker
    Docker Commons
    Docker Pipeline
    Docker API
    docker-build-step
    NodeJS
    OWASP Dependency-Check
    SonarQube Scanner
    ```

- Configure the installed plugins: `Dashboard` -> `Manage Jenkins` -> `Tools`

    - Search for the `Sonarqube Scanner Installations` and provide the configuration details.
        - Name: sonar-scanner
        - Install automatically: from Maven central
        
    - Search for `NodeJS Installations` and provide the configuration details.
        - Name: nodejs
        - Install automatically: from nodejs.org

    - Search for `Dependency-Check Installations` and provide the configuration details.
        - Name: DP-check
        - Install automatically: from github.com

    - Search for `Docker Installations` and provide the configuration details.
        - Name: docker
        - Install automatically: from docker.com

- Set the path for Sonarqube in Jenkins: `Dashboard` -> `Manage Jenkins` -> `System` -> `SonarQube installations`

    - Name: sonar-server
    - Server URL: `http://<jenkins-server-public-ip>:9000/`
    - Server Auth Token: `sonar-token`

- Create Jenkins Pipeline to deploy the Backend Code: `Jenkins Dashboard` -> `New Item` -> `Pipeline`   
        
    - Name: Three-Tier-Backend-Application
    - Copy paste the code from `Jenkins-Pipeline-Code/Jenkinsfile-Backend`.
    - Click on the `Build Now`.

- Create Jenkins Pipeline to deploy the Frontend Code: `Jenkins Dashboard` -> `New Item` -> `Pipeline`   
        
    - Name: Three-Tier-Frontend -Application
    - Copy paste the code from `Jenkins-Pipeline-Code/Jenkinsfile-Frontend `.
    - Click on the `Build Now`.

## Step 8. Configure ArgoCD

- If the Source Code Repository is Private

    - `Settings` -> `Repositories` -> `CONNECT REPO USING HTTPS`

        - Type: git
        - Project: default
        - Repo URL: `https://github.com/aqeeladil/mern-end-to-end-devops-implementation.git`
        - Username: aqeeladil
        - Password: <github-personal-access-token>

    - If your Connection Status is Successful it means repository connected successfully.

- Create a namespace in `Jump Server` using: `kubectl create namespace three-tier`.

- Create argocd applications for the database, backend, frontend and ingress.

    - Database:

        - three-tier-database
        - default
        - automatic
        - self-heal
        - `https://github.com/aqeeladil/mern-end-to-end-devops-implementation.git`
        - `Kubernetes-Manifests-file/Database/`
        - kubernetes.default.svc
        - three-tier

        ```bash
        #Verify 
        kubectl get all -n three-tier

        # Persistent Volume & Persistent Volume Claim are configured. So, if the pods get deleted then, the data won’t be lost. The Data will be stored on the host machine.

        kubectl get pvc -n three-tier
        ```

    - Backend: 

        - three-tier-backend
        - default
        - automatic
        - self-heal
        - `https://github.com/aqeeladil/mern-end-to-end-devops-implementation.git`
        - `Kubernetes-Manifests-file/Backend/`
        - kubernetes.default.svc
        - three-tier

    - Frontend: 

        - three-tier-frontend
        - default
        - automatic
        - self-heal
        - `https://github.com/aqeeladil/mern-end-to-end-devops-implementation.git`
        - `Kubernetes-Manifests-file/Frontend/`
        - kubernetes.default.svc
        - three-tier

    - Ingress: 

        - three-tier-ingress
        - default
        - automatic
        - self-heal
        - `https://github.com/aqeeladil/mern-end-to-end-devops-implementation.git`
        - `Kubernetes-Manifests-file/`
        - kubernetes.default.svc
        - three-tier

- Once the Ingress application is deployed. It will create an Application Load Balancer

## Step 9. Load balancer Integrtion with Route53

- `Route53` -> `Hosted Zones` -> `aqeeladil.site` -> `Create record`

    - Record type: A-Routes traffic to an ipv4 and some Aws resources.
    - Alias
    - Routes traffic to: Alias to Application and Classic Load Balancer.
    - US-East (N.Virginia)
    - <your-alb-load-balancer-id>
    - Simple Routing

- Verify all the created resource using: 

    ```
    kubectl get all -n three-tier
    kubectl get ing -n three-tier
    ```

- Access the application at `http://aqeeladil.site`

## Step 10. On the `Jump Server`, set up Monitoring for EKS Cluster.

```bash
# Install Prometheus 
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Install Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana

# Verify
helm repo list
kubectl get deploy

# Access the Prometheus and Grafana consoles 
kubectl get svc
kubectl edit svc prometheus-server      # Change the Service type from ClusterType to LoadBalancer
kubectl edit svc stable-grafana         # Change the Service type from ClusterType to LoadBalancer

# Verify 
kubectl get svc prometheus-server
kubectl get svc grafana  
# you will see the LoadBalancers DNS name under `EXTERNAL-IP` section.
# You can also validate from aws load balancer console.
```

- Access Prometheus Dashboar at `<Prometheus-LB-DNS>:9090`.

    - `Status` -> `Target`.

    - You will see a lot of Targets

- Access Grafana Dashboard at `<Grafana-LB-DNS>.

    - Retrieve username & password: 
        ```bash
        kubectl get secrets
        kubectl get secret grafana -n default -o jsonpath="{.data.admin-user}" | base64 --decode
        kubectl get secret grafana -n default -o jsonpath="{.data.admin-password}" | base64 --decode
        ```

    - `Data Sources` -> `Add data source` -> `Prometheus`

        - In the Connection, paste your <Prometheus-LB-DNS>:9090.
        - If the URL is correct, then you will see a green notification/

- On Grafana, create a dashboard to visualize Kubernetes Cluster Logs.

    - `Dashboard` -> `New` -> `Import` -> 

        - Provide `6417` ID and click on Load
        - `6417` is a unique ID from Grafana which is used to Monitor and visualize Kubernetes Data
        - Select the data source that you have created earlier and click on Import.

    - You can view your Kubernetes Cluster Data.




























