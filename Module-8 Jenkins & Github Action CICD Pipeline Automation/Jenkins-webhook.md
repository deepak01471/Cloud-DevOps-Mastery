# Jenkins CI/CD Pipeline for a Dockerized Node.js Application: Manual Trigger vs Automatic Trigger Using GitHub Webhooks
In this article, we will build and understand a complete CI/CD workflow using Jenkins, Docker, GitHub, and a Node.js application.

We will cover:

- Creating a Jenkins pipeline
- Building a Docker image
- Deploying a container
- Triggering builds manually
- Triggering builds automatically
- GitHub Personal Access Tokens
- Fine-grained vs Classic Tokens
- Jenkins credentials
- GitHub webhooks
- Required Jenkins plugins
- Common errors and troubleshooting
The goal is to understand not only how to configure everything but also why each component is needed.

---

# Architecture Overview
The complete flow looks like this:

```text
Developer
    |
    | Git Push
    v
GitHub Repository
    |
    | Webhook
    v
Jenkins
    |
    | Build Docker Image
    v
Docker
    |
    | Run Container
    v
Application Running
```
Without webhooks:

```text
Developer
    |
    | Git Push
    v
GitHub Repository

Jenkins Build Now (Manual Trigger)
    |
    v
Build and Deploy
```
With webhooks:

```text
Developer
    |
    | Git Push
    v
GitHub Repository
    |
    v
Webhook
    |
    v
Jenkins
    |
    v
Build and Deploy Automatically
```
---

# Prerequisites
Before starting, ensure you have:

- Ubuntu Server
- Jenkins installed
- Docker installed
- Git installed
- GitHub repository
- Node.js application with Dockerfile
Verify installations:

```bash
jenkins --version
docker --version
git --version
```
---

# Installing Docker on Jenkins Server
Install Docker:

```bash
sudo apt update
sudo apt install docker.io -y
```
Enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```
Verify:

```bash
docker --version
```
---

# Allow Jenkins to Use Docker
By default Jenkins cannot execute Docker commands.

Add Jenkins user to Docker group:

```bash
sudo usermod -aG docker jenkins
```
Restart Jenkins:

```bash
sudo systemctl restart jenkins
```
Verify:

```bash
sudo su - jenkins
docker ps
```
If Docker works without sudo, Jenkins is ready.

---

# Creating the Pipeline
Initially we created a Jenkins pipeline that manually clones the repository.

Example:

```groovy
pipeline {
    agent any

    stages {

        stage('Clone Repository') {
            steps {
                sh '''
                    mkdir -p devops
                    cd devops
                    rm -rf Node.js-App-Deploy-Github-Action
                    git clone -b main https://github.com/username/repository.git
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    cd devops/Node.js-App-Deploy-Github-Action
                    docker build -t node-app .
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker run -d -p 8000:8080 node-app
                '''
            }
        }
    }
}
```
This works, but every deployment requires manually clicking:

```text
Build Now
```
---

# Problem with Multiple Deployments
Suppose the application is already running.

Running:

```bash
docker run -d -p 8000:8080 node-app
```
again will fail because port 8000 is already occupied.

Error:

```text
Bind for 0.0.0.0:8000 failed
```
---

# Better Deployment Approach
Before starting a new container, remove the old one.

```bash
docker rm -f node-app-container || true
```
Then start a new container:

```bash
docker run -d --name node-app-container -p 8000:8080 node-app
```
---

# Understanding docker rm -f node-app-container || true
Let's break it down.

## docker rm
Removes a container.

```bash
docker rm node-app-container
```
Works only if container is stopped.

---

## -f
Force remove.

```bash
docker rm -f node-app-container
```
This:

1. Stops container
2. Removes container
---

## ||
OR operator.

Syntax:

```bash
command1 || command2
```
If command1 fails, command2 executes.

---

## true
Always returns success.

```bash
true
```
Exit code:

```text
0
```
---

## Final Meaning
```bash
docker rm -f node-app-container || true
```
If container exists:

```text
Remove it
```
If container doesn't exist:

```text
Ignore error and continue
```
This prevents Jenkins from failing.

---

# Manual Triggering
The simplest approach is manual execution.

Navigate to:

```text
Jenkins Job
|
└── Build Now
```
Advantages:

- Easy to understand
- Good for learning
Disadvantages:

- Requires human intervention
- Not real CI/CD
---

# Automatic Triggering
The goal of CI/CD is:

```text
Code Push
    |
    v
Automatic Build
    |
    v
Automatic Deployment
```
This is where GitHub webhooks come into play.

---

# Required Jenkins Plugins
Install the following plugins:

## Git Plugin
Allows Jenkins to work with Git repositories.

## GitHub Plugin
Provides GitHub integration.

## GitHub Integration Plugin
Enables webhook-based triggering.

## Pipeline Plugin
Allows Jenkinsfile execution.

## Credentials Plugin
Stores secrets securely.

---

# GitHub Authentication
Public repositories may clone without authentication.

Private repositories require authentication.

GitHub no longer supports account passwords for Git operations.

Use a Personal Access Token.

---

# Classic Personal Access Token
Older token type.

Advantages:

- Simple
- Easy to configure
Disadvantages:

- Broad permissions
- Less secure
Example scopes:

```text
repo
workflow
admin:repo_hook
```
---

# Fine-Grained Personal Access Token
Newer and recommended approach.

Advantages:

- Repository-level access
- Better security
- Granular permissions
Example:

```text
Repository Access:
Only selected repositories
```
Permissions:

```text
Contents: Read and Write
Metadata: Read
Webhooks: Read and Write
```
---

# Fine-Grained vs Classic Token
| Feature | Fine-Grained | Classic |
| ----- | ----- | ----- |
| Security | High | Lower |
| Repository Scope | Specific | Broad |
| Permission Control | Granular | Broad |
| Recommended | Yes | Legacy |
For modern projects, prefer Fine-Grained tokens.

---

# Adding GitHub Token to Jenkins
Navigate to:

```text
Manage Jenkins
|
Credentials
```
Select:

```text
Global Credentials
```
Choose:

```text
Add Credentials
```
Kind:

```text
Username with Password
```
Example:

```text
Username: GitHub Username
Password: Personal Access Token
```
ID:

```text
github-creds
```
Save.

---

# Pipeline Script vs Pipeline Script from SCM
Many beginners get confused here.

## Pipeline Script
Pipeline stored inside Jenkins UI.

Example:

```groovy
pipeline {
    agent any
}
```
Advantages:

- Quick setup
Disadvantages:

- Not version controlled
- Difficult to maintain
---

## Pipeline Script from SCM
Pipeline stored in GitHub repository.

Repository structure:

```text
project/
|
|-- Dockerfile
|-- package.json
|-- app.js
|-- Jenkinsfile
```
Jenkins automatically downloads Jenkinsfile.

Advantages:

- Version controlled
- Industry standard
- Easier maintenance
Recommended approach.

---

# Configuring Pipeline from SCM
Create Jenkins job.

Select:

```text
Pipeline
```
Under Definition:

```text
Pipeline script from SCM
```
SCM:

```text
Git
```
Repository URL:

```text
https://github.com/username/repository.git
```
Branch:

```text
*/main
```
Script Path:

```text
Jenkinsfile
```
Save.

---

# Creating GitHub Webhook
Navigate to:

```text
GitHub Repository
|
Settings
|
Webhooks
|
Add Webhook
```
Payload URL:

```text
http://JENKINS_PUBLIC_IP:8080/github-webhook/
```
Content Type:

```text
application/json
```
Event:

```text
Just the push event
```
Save webhook.

---

# Configuring Jenkins Trigger
Open job configuration.

Under Build Triggers:

Select:

```text
GitHub hook trigger for GITScm polling
```
Save.

---

# Testing the Webhook
Push code:

```bash
git add .
git commit -m "testing webhook"
git push origin main
```
Expected flow:

```text
GitHub Push
    |
    v
Webhook
    |
    v
Jenkins
    |
    v
Pipeline Starts
```
No manual click required.

---

# Common Troubleshooting
## Webhook Returns 404
Cause:

```text
Wrong webhook URL
```
Correct:

```text
http://SERVER-IP:8080/github-webhook/
```
---

## Webhook Returns 403
Cause:

```text
Authentication or security issue
```
Verify:

- GitHub plugin
- GitHub integration plugin
---

## Webhook Returns 200 But Build Doesn't Start
Common cause:

```text
Pipeline Script instead of Pipeline Script from SCM
```
or

```text
Repository mapping issue
```
---

## Dockerfile Not Found
Example:

```text
unable to evaluate symlinks in Dockerfile path
```
Cause:

Wrong working directory.

Check:

```bash
pwd
ls -la
```
Verify Dockerfile location.

---

## Permission Denied While Running Docker
Cause:

```text
Jenkins not in docker group
```
Fix:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
---

# Final Jenkinsfile
```groovy
pipeline {
    agent any

    stages {

        stage('Build Image') {
            steps {
                sh '''
                    docker build -t node-app .
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f node-app-container || true
                    docker run -d --name node-app-container -p 8000:8080 node-app
                '''
            }
        }
    }
}
```
---

# Conclusion
A Jenkins pipeline can be triggered manually or automatically. Manual triggering is useful for learning and testing, but real CI/CD begins when code pushes automatically trigger builds and deployments.

The recommended production approach is:

1. Store the Jenkinsfile in GitHub.
2. Use Pipeline Script from SCM.
3. Configure GitHub credentials using a Personal Access Token.
4. Enable GitHub webhook integration.
5. Use Docker for packaging and deployment.
6. Remove old containers before deploying new versions.
With this setup, every code push automatically builds a Docker image, deploys a fresh container, and updates the application without requiring any manual intervention.

