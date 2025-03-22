# **Boardgame CI/CD Pipeline with Jenkins, Kubernetes, SonarQube, Nexus & Trivy**

## **📌 Overview**
This project sets up a **CI/CD pipeline** using **Jenkins, Minikube, SonarQube, Nexus, Trivy, and Prometheus** to automate the build, testing, security scanning, and deployment of a **Java Spring Boot** application using **Blue-Green Deployment**.
a complete CI/CD (Continuous Integration/Continuous
Delivery) pipeline, while demonstrating most of the DevOps flow Chain practices, and making
use of modern frameworks and technologies.

✅ **Technologies Used:**
- **Jenkins** (CI/CD Orchestration)
- **Maven** (Build Management)
- **Nexus** (Artifact Repository)
- **SonarQube** (Code Quality Analysis)
- **Trivy** (Security & Vulnerability Scanning)
- **Docker** (Containerization)
- **Kubernetes (Minikube)** (Deployment & Orchestration)
- **Prometheus & Grafana** (Monitoring & Metrics)
- **Ansible** (Automated Environment Setup)

---
---

## **🚀 CI/CD Pipeline Flow**  
1️⃣ **Cleanup Workspace** - Deletes all files from the workspace to ensure a clean build environment.  
2️⃣ **Install Kubectl** - Installs `kubectl` if not already present to interact with the Kubernetes cluster.  
3️⃣ **Code Checkout** - Clones the latest source code from the GitHub repository.  
4️⃣ **Compile & Build** - Uses Maven to compile the Java application, ensuring there are no syntax errors.  
5️⃣ **Run Unit Tests** - Executes unit tests using Maven Surefire to validate individual components.  
6️⃣ **Run Integration Tests** - Runs integration tests with `mvn verify` to check how components interact.  
7️⃣ **SonarQube Analysis** - Analyzes the code for bugs, security vulnerabilities, and code smells using SonarQube.  
8️⃣ **SonarQube Quality Gate** - Checks the SonarQube quality gate status, failing the pipeline if issues are found.  
9️⃣ **Build JAR Package** - Packages the application into a `.jar` file using Maven.  
🔟 **Package & Versioning** - Updates the application version dynamically and uploads the JAR to Nexus for artifact storage.  
1️⃣1️⃣ **Upload Test Reports** - Stores unit and integration test reports in Nexus for future reference.  
1️⃣2️⃣ **Build Docker Image** - Creates a Docker image for the application.  
1️⃣3️⃣ **Trivy Security Scan** - Scans the built Docker image for security vulnerabilities before deployment.  
1️⃣4️⃣ **Manual Security Approval** - Requests manual approval if critical vulnerabilities are found in the Trivy scan.  
1️⃣5️⃣ **Docker Push** - Pushes the built Docker image to Docker Hub with a unique version tag and latest tag.  
1️⃣6️⃣ **Approval Before Deployment** - Requests manual approval before deploying the application to Kubernetes.  
1️⃣7️⃣ **Deploy to Minikube** - Deploys the application using Kubernetes manifests to the Minikube cluster.  
1️⃣8️⃣ **Blue-Green Deployment** - Deploys both Blue and Green environments, ensuring zero-downtime updates.  
1️⃣9️⃣ **Traffic Switching** - Switches Nginx routing between Blue and Green deployments to update the live version.  
2️⃣0️⃣ **Email Notifications** - Sends success or failure emails with build logs using Jenkins email notifications.  

---
# **📂 Infrastructure Setup with Ansible**  
The repository also contains an **Ansible** playbook to automate the installation and configuration of the required CI/CD tools:  

📁 **Folder Structure:**  
```
├── inventories/
│   └── hosts
├── playbooks/
│   └── setup-ci-cd.yml
├── roles/
│   ├── jenkins/
│   │   └── tasks/
│   │       └── main.yml
│   ├── nexus/
│   │   ├── tasks/
│   │   │   └── main.yml
│   ├── sonarqube/
│   │   ├── tasks/
│   │   │   └── main.yml
│   └── minikube/
│       └── tasks/
│           └── main.yml
```
This Ansible playbook automates:  
✅ Installing and configuring Jenkins  
✅ Deploying Nexus Repository Manager in a Docker container  
✅ Deploying SonarQube in a Docker container  
✅ Installing and configuring Minikube  

### **🔹 Run the Playbook**
1. **Modify the inventory file** (`ansible/inventories/hosts`) with the target server details. ( default set to localhost for all running locally on the same machine )
2. **Run the setup:**
```sh
cd ansible
ansible-playbook -i inventories/hosts playbooks/setup-ci-cd.yml
```
``` 


## **🔑 Accessing CI/CD Tools**  
After running the **Ansible playbook**, the following services will be available:

🔹 **Jenkins**  
- URL: Exposed via **ngrok** (see below)  
- Default username: `admin`  
- Password: Retrieved from Jenkins initial admin password file `/var/lib/jenkins/secrets/initialAdminPassword`  
- **Plugins equired for the pipeline to function correctly. Some are installed automatically when Jenkins is first set up, but the following additional plugins need to be installed manually**:  
  - Pipeline  
  - Blue Ocean  
  - SonarQube Scanner  
  - Nexus Artifact Uploader  
  - Trivy Plugin  
  - GitHub Webhook Plugin  
  - Email Extension Plugin  
  - Pipeline Maven Integration
  - Config File Provider
  - SonarQube Scanner
  - Kubernetes CLI
  - Kubernetes
  - Docker
  - Docker Pipeline Step

These plugins can be installed via Manage Jenkins → Manage Plugins in the Jenkins UI.

🔹 **Nexus Repository Manager**  
- URL: `http://localhost:8081`  
- Default username: `admin`  
- Default password: Retrieved from `/nexus-data/admin.password`  
- Repository used: `maven-releases` for storing versioned JAR files  

🔹 **SonarQube**  
- URL: `http://localhost:9000`  
- Default username: `admin`  
- Default password: `admin` (should be changed after first login)  
- SonarQube token is configured in Jenkins credentials  

## **🔗 Webhook Configuration (GitHub to Jenkins)**  
A **GitHub Webhook** is configured to trigger the Jenkins pipeline on every code push to the repository.  

1. Jenkins is exposed publicly using **ngrok**:  
   ```sh
   ngrok http 8080
   ```  
   Copy the **ngrok URL** (e.g., `https://random-string.ngrok.io`)  

2. Configure the webhook in GitHub:  
   - Navigate to **GitHub Repository → Settings → Webhooks**  
   - Click **Add Webhook**  
   - Payload URL: `https://<ngrok-url>/github-webhook/`  
   - Content type: `application/json`  
   - Select event trigger: **Just the push event**  

3. In Jenkins:  
   - Go to **Manage Jenkins → Configure System**  
   - Under **GitHub**, add credentials and enable **GitHub hook trigger for GITScm polling**  

Now, every **git push** to the repository will automatically trigger the Jenkins pipeline.





## **📂 Folder Structure**
```
.
├── ansible/                 # Ansible automation
├── k8s/                     # Kubernetes manifests for deployment
│   ├── namespace.yaml
│   ├── boardgame-deployment.yaml
│   ├── boardgame-service.yaml
│   ├── boardgame-blue-deployment.yaml
│   ├── boardgame-green-deployment.yaml
│   ├── nginx-configmap.yaml
│   ├── nginx-deployment.yaml
│   ├── nginx-service.yaml
│   ├── service-monitor.yaml
├── src/                     # Java Spring Boot application source code
├── Dockerfile               # Dockerfile for building the application image
├── Jenkinsfile              # Jenkins Pipeline script
├── README.md                # This README file for the web app 
```

## **📧 Notifications & Monitoring**  
✅ **Jenkins Email Notifications** - Sends build success/failure alerts.  
✅ **Prometheus & Grafana** - Monitors Jenkins and Kubernetes cluster metrics.  ( installation steps provided in another readme file : README-PROMETHEUS.MD )
✅ **Trivy Reports** - Provides security vulnerability reports for Docker images.  

## **📌 How to Use**  
1️⃣ Clone the repository  
2️⃣ Run the **Ansible playbook** to set up the CI/CD environment  
3️⃣ Configure Jenkins credentials for Nexus, SonarQube, and Docker Hub  
4️⃣ Configure the **GitHub Webhook** to trigger the Jenkins pipeline  
5️⃣ Monitor build results and security scans in Jenkins  
6️⃣ Deploy and manage the application in **Minikube**  
