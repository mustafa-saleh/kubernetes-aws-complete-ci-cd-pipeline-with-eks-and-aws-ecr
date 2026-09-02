# Kubernetes on AWS - Complete CI/CD Pipeline with EKS and AWS ECR Registry

This repository demonstrates an end-to-end CI/CD workflow for a Java Maven application deployed to Amazon EKS from Jenkins, while storing release images in a private AWS ECR repository.

## Overview

Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.

Amazon Elastic Kubernetes Service (Amazon EKS) is a managed Kubernetes service that runs a Kubernetes control plane on AWS and keeps it highly available and scalable.

Jenkins is a self-contained, open-source automation server used to automate building, testing, and delivering/deploying software through repeatable pipelines.

Amazon Elastic Container Registry (ECR) is a fully managed AWS container image registry used to store, manage, scan, and version private Docker/OCI images for secure deployments.

In this project, Jenkins performs version bumping, application build, Docker image build/push, Kubernetes deployment to EKS, and pushes version updates back to GitHub. 🚀

### Amazon EKS key features

- Managed and highly available Kubernetes control plane
- Multiple compute options: managed node groups (EC2) and Fargate
- Integration with AWS IAM, VPC networking, and load balancing
- Scalable infrastructure for production-grade workloads

## Demo Project

Complete CI/CD Pipeline with EKS and AWS ECR

## Technologies used

- Kubernetes
- Jenkins
- AWS EKS
- AWS ECR
- Java
- Maven
- Linux
- Docker
- Git

## Project Description

- Create private AWS ECR Docker repository
- Adjust Jenkinsfile to build and push Docker image to AWS ECR
- Integrate deploying to K8s cluster in the CI/CD pipeline from AWS ECR private registry
- So the complete CI/CD project we build has the following configuration:
    - a. CI step: Increment version
    - b. CI step: Build artifact for Java Maven application
    - c. CI step: Build and push Docker image to AWS ECR
    - d. CD step: Deploy new application version to EKS cluster
    - e. CD step: Commit the version update

## Repository structure

```text
.
├── Dockerfile
├── Jenkinsfile
├── README.md
├── images/
│   ├── aws-ecr-image-console.png
│   ├── eksctl-cluster-cloudformation-stacks-console.png
│   ├── eksctl-cluster-create-terminal.png
│   ├── eksctl-cluster-nodes-console.png
│   ├── java-maven-pod-running-terminal.png
│   ├── jenkins-credentials.png
│   └── jenkins-pipeline-build-success.png
├── kubernetes/
│   ├── deployment.yaml
│   └── service.yaml
├── pom.xml
└── src/
    ├── main/java/com/example/Application.java
    ├── resources/static/index.html
    └── test/java/AppTest.java
```

## Architecture overview

```mermaid
flowchart LR
    Dev[Developer Pushes Code to GitHub] --> Jenkins[Jenkins Multibranch Pipeline]
    Jenkins -->|mvn clean package| Artifact[Java JAR Artifact]
    Jenkins -->|docker build + docker push| ECR[Private AWS ECR Repository]
    Jenkins -->|kubectl apply with envsubst| EKS[Amazon EKS Cluster]
    EKS --> Deployment[Kubernetes Deployment]
    Deployment --> Pods[Java Maven App Pods]
    Service[Kubernetes Service] --> Pods
```

### Runtime flow

- Jenkins checks out the branch and increments Maven version.
- Jenkins builds the application JAR and container image.
- Jenkins authenticates to AWS ECR and pushes a versioned image tag.
- Jenkins authenticates to EKS with AWS credentials and kubeconfig.
- Jenkins applies Deployment and Service manifests to roll out the new version.
- Kubernetes pulls the private image using imagePullSecrets.

## Implementation Guide

### 1. Prerequisites

- AWS account with permissions for EKS, IAM, EC2, VPC, and CloudFormation
- AWS CLI configured
- kubectl installed locally
- eksctl installed locally
- Docker installed on Jenkins host
- Jenkins running and reachable
- GitHub repository with branch strategy for Jenkins pipeline

Check AWS profile and region:

```bash
# check config & region
aws configure list
```

Install eksctl:

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### 2. Provision EKS Cluster

Create a cluster using:

```bash
eksctl create cluster \
--name demo-cluster \
--version 1.36 \
--region us-east-1 \
--nodegroup-name demo-nodes \
--node-type t2.micro \
--nodes 2 \
--nodes-min 1 \
--nodes-max 3
```

Verify cluster creation:

```bash
# check cluster is created
eksctl get cluster
```

Alternative using config file:

```bash
eksctl create cluster -f cluster.yaml
```

Update local kubeconfig and verify API server connectivity:

```bash
# create kube config file to connect to the cluster 
aws eks update-kubeconfig --name <cluster-name>

# check cluster details
kubectl cluster-info
```

Screenshots:

![EKS cluster creation from terminal](images/eksctl-cluster-create-terminal.png)
![CloudFormation stacks created by eksctl](images/eksctl-cluster-cloudformation-stacks-console.png)
![EKS nodes in AWS console](images/eksctl-cluster-nodes-console.png)

### 3. Prepare Jenkins Runtime for Kubernetes Deployment

Run Jenkins in Docker, then enter container as root and install kubectl:

```bash
docker ps

# root shell on jenkins container
docker exec -u 0 -it <container-id> bash

# Install kubectl on Jenkins server
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl; chmod +x ./kubectl; mv ./kubectl /usr/local/bin/kubectl

# check installation
kubectl version
```

Install aws-iam-authenticator inside the Jenkins container:

```bash
# install aws-iam-authenticator
curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.6.11/aws-iam-authenticator_0.6.11_linux_amd64

chmod +x ./aws-iam-authenticator

mv ./aws-iam-authenticator /usr/local/bin

aws-iam-authenticator help
```

Install envsubst for manifest variable substitution:

```bash
ssh root@server-ip

docker exec -it -u 0 <container-id> bash

# install envsubst
apt-get install gettext-base

envsubst --version
```

### 4. Configure kubeconfig for Jenkins

Create kubeconfig compatible with AWS authentication:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <certificate-data>
    server: <endpoint-url>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: aws
  name: aws
current-context: aws
users:
- name: aws
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: /usr/local/bin/aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - <cluster-name>
```

Copy kubeconfig into Jenkins home:

```bash
# Copy config file to Jenkins server
docker cp config "YOUR DOCKER CONTAINER ID":/var/jenkins_home/.kube/
```

### 5. Configure Credentials and Registry Access

Add AWS credentials in Jenkins as secret text values:

```text
jenkins_aws_access_key_id
jenkins-aws_secret_access_key
```

Create private AWS ECR repository "java-maven-app" and add Jenkins credentials for ECR as username/password:

- credential ID: ecr-credentials
- username: AWS
- password: ECR login password

Screenshot:

![AWS ECR repository image view](images/aws-ecr-image-console.png)

Generate ECR login password and authenticate:

```bash
# get ecr-password, valid for 12 hours
aws ecr get-login-password --region region

aws ecr get-login-password --region region | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.region.amazonaws.com
```

Create a secret in k8s to connect to ECR & pull the image:

```bash
kubectl create secret docker-registry aws-registry-key \
--docker-server=aws_account_id.dkr.ecr.region.amazonaws.com \
--docker-username= \
--docker-password=

kubectl get secret
```

Screenshot:

![Jenkins credentials configuration](images/jenkins-credentials.png)

### 6. CI/CD Pipeline Implementation

✅ This is the core automation workflow that connects CI and CD in one pipeline.

Pipeline stages implemented in this repository:

- Increment version in pom.xml
- Build Java artifact with Maven
- Build and push Docker image to private AWS ECR
- Deploy manifests to EKS using kubectl + envsubst
- Commit version bump back to remote branch

Current Jenkinsfile used in this project:

```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        DOCKER_REPO_SERVER = '911167908038.dkr.ecr.us-east-1.amazonaws.com'
        DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'ecr-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "docker build -t ${DOCKER_REPO}:${IMAGE_NAME} ."
                        sh "echo $PASS | docker login -u $USER --password-stdin ${DOCKER_REPO_SERVER}"
                        sh "docker push ${DOCKER_REPO}:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('deploy') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo 'deploying docker image...'
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update'){
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "git remote set-url origin https://${USER}:${PASS}@github.com/mustafa-saleh/kubernetes-aws-complete-ci-cd-pipeline-with-eks-and-aws-ecr.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
    }
}
```

### 7. Kubernetes Manifests Used in Deployment Stage

Deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  labels:
    app: $APP_NAME
spec:
  replicas: 2
  selector:
    matchLabels:
      app: $APP_NAME
  template:
    metadata:
      labels:
        app: $APP_NAME
    spec:
      imagePullSecrets:
        - name: aws-registry-key
      containers:
        - name: $APP_NAME
          image: $DOCKER_REPO:$IMAGE_NAME
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```

Service manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: $APP_NAME
spec:
  selector:
    app: $APP_NAME
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

Why envsubst is used:

- The same YAML templates are reused across builds.
- Jenkins injects APP_NAME, DOCKER_REPO, and IMAGE_NAME at runtime.
- Each pipeline run deploys a unique immutable image tag.

### 8. Validate Deployment and Release Result

Use the same command flow from notes to verify rollout and runtime health:

```bash
# check result
kubectl get pod

kubectl get service

# check the image registry is ECR
kubectl describe pod <pod-name>
```

Additional verification commands:

```bash
kubectl get pods -n default
kubectl get svc -n default
kubectl describe deployment <depl-name>
kubectl logs <pod-name>
```

## Key lessons learned

- EKS cluster provisioning with eksctl is fast, but production readiness still depends on IAM, networking, and access design.
- Running Jenkins in a container requires explicit installation of kubectl, aws-iam-authenticator, and envsubst to support Kubernetes CD.
- Private ECR integration requires both CI-side push credentials and cluster-side imagePullSecrets.
- Parameterized Kubernetes manifests plus envsubst reduce duplication and enforce consistent release behavior.
- Version bumping inside CI creates traceability between source version, image tag, Jenkins build number, and deployed workload.
- Credential scoping in Jenkins (AWS, ECR, GitHub) is critical for secure automation and least-privilege access.

## Final result

The pipeline in this repository successfully executes a full CI/CD cycle. ✅

- Increments Maven version
- Builds and tests the Java application
- Builds and pushes a private AWS ECR image
- Deploys the new image to Amazon EKS
- Commits version updates back to GitHub branch

Evidence screenshots:

![Jenkins pipeline successful run](images/jenkins-pipeline-build-success.png)
![Java Maven pod running on EKS](images/java-maven-pod-running-terminal.png)

## References

- AWS EKS overview: https://aws.amazon.com/eks/
- AWS EKS features: https://aws.amazon.com/eks/features/
- AWS EKS user guide: https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html
- eksctl: https://github.com/eksctl-io/eksctl
- kubectl install docs: https://kubernetes.io/docs/tasks/tools/
- Jenkins documentation: https://www.jenkins.io/doc/
- Amazon ECR user guide: https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html

