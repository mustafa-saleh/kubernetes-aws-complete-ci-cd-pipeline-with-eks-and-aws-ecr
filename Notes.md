# Kubernetes on AWS - EKS

## 1 - Container Services on AWS

Worder nodes options

- EKS with EC2 (self managed)
- EKS with Nodegroup (semi managed)
- EKS with Fargate (fully managed)

## 2 - Create EKS cluster with AWS Management Console

Detailed Steps to create an EKS cluster. Step-by-Step Process

1) create EKS IAM Role
2) create VPC for Worker Nodes
3) create EKS cluster (Control Plane Nodes)
4) connect kubectl with EKS cluster
5) create EC2 IAM Role for Node Group
6) create Node Group and attach to EKS cluster
7) configure auto-scaling
8) deploy our application to our EKS cluster

### create EKS IAM Role

Create new role "eks-cluster-role" with "AmazonEKSClusterPolicy" and assign it to an EKS cluster managed by AWS to allow AWS access to create and manage resources on your behalf.

### create VPC for Worker Nodes

- EKS cluster needs specific networking configuration
- Worker Nodes need specific Firewall configurations for Control Plane-Worker communication
- Best Practice = Configure public and private subnet
- Through IAM Role you give K8s permission to change VPC configurations

Use cloudformation template "eks-worker-node-vpc-stack" to create cluster VPC with Public and private subnets. docs to create VPC "https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html". Cloudformation template "https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml"

### create EKS cluster (Control Plane Nodes)

create the custom eks cluster "eks-cluster-test", choose "eks-worker-node-vpc-stack" as cluster VPC

### connect kubectl with EKS cluster

```bash
# check config & region
aws configure list

# create kube config file to connect to the cluster 
aws eks update-kubeconfig --name <cluster-name>

# check cluster details
kubectl cluster-info
```

### create EC2 IAM Role for Node Group

- Kubelet is the main worker process - scheduling, managing Pods and communicate with other AWS services
- That's why Kubelet on Worker Node needs permission
- Create Role for Node Group

create a new EC2 role "eks-node-group-role" with 3 permission policies (AmazonEKSWorkerNodePolicy, AmazonEC2containerRegistryReadOnly, AmazonEKS_CNI_Policy)

### create Node Group and attach to EKS cluster

Create a new Node Group "eks-node-group" with the role "eks-node-group-role". (min=2, desired=2, max=2)

This will create 2 EC2 instances & an ASG

With NodeGroup all the necessary worker processes (container runtime, kubelet, kube-proxy) are installed 

```bash
# check nodes created
kubectl get nodes
```

## 3 - Configure Autoscaling in EKS cluster

AWS doesn't automatically scale the resources

### K8s Cluster Autoscaler

It automatically scales nodes up or down based on workload demand (k8s autoscaler needs permission to create & delete EC2 instances)

K8s autoscaler will have a service account (k8s component ) for authentication 

It's tied to RBAC, control what the service account can do with roles & role bindings

AWS roles normally works well for AWS services, but in this case we've to provide permission for a K8s component to call AWS API and manage other AWS services. To solve this we can establish trust between AWS & k8s with OIDC (OpenID Connect) identity provider, after that we need to create an IAM role (Web Identity) that allows create/delete EC2 instances. Web Identity allows third-party users/services that were authenticated by configured identity provider to assume that role

### configure auto-scaling

Save infrastructure cost with an auto scaler 

Steps:

- Auto Scaling Group
- Create custom policy
- Setup OIDC (OpenID Connect) identity provider
- Create IAM Role & attach policy
- Deploy K8s autoscaler

#### Auto Scaling Group

Created with cluster node group

#### Create custom policy 

Create custom policy "ClusterAutoScalerPolicy" with EC2 & ASG permissions & attach to a new IAM Role

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

#### Setup OIDC (OpenID Connect) identity provider

The K8s cluster will have an "OpenID connect provider Url". When a service account inside the cluster tries to assume an AWS Role, AWS will allow it only if the token is valid and issued by the cluster (token verified via "OpenID connect provider Url").

In AWS IAM, add a new OpenID "Identity Provider" with the "OpenID connect provider Url" & audience "sts.amazonaws.com" 

STS stands for "AWS Security Token Service" which issue temporary credentials for authenticated users.

#### Create IAM Role & attach policy

Create a Web Identity Role "EKSServiceAccountRole" with the above OIDC & audience "STS" & attach "ClusterAutoScalerPolicy"

#### Deploy K8s autoscaler

Before deploying the autoscaler, ensure that the ASG are tagged for the cluster autoscaler, below tags are required

- k8s.io/cluster-autoscaler/enabled
- k8s.io/cluster-autoscaler/<cluster-name>

Deploy the cluster autoscaler with the following "https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml"

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam:ACCOUNTID:role/EKSServiceAccountRole
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources:
      - "namespaces"
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["resource.k8s.io"]
    resources: ["deviceclasses", "resourceslices", "resourceclaims"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities", "volumeattachments"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
        cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      serviceAccountName: cluster-autoscaler
      containers:
        # autoscaler version should match k8s version, autoscaler versions can be found in github k8s repo -> tags, if k8s version is 1.36 then autoscaler version will be v1.36.<autoscaler-highest-patch-number>
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.32.1
          name: cluster-autoscaler
          env:
            - name: AWS_REGION
            value: "<YOUR_AWS_REGION>"
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt # /etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
              readOnly: true
          imagePullPolicy: "Always"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
```

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml

# check deployment
kubectl get deployment -n kube-system cluster-autoscaler

# check pods
kubectl get pod -n kube-system

# check the node on which autoscaler is running, check "Node" column
kubectl get pod <autoscaler-pod-name> -n kube-system -o wide

# check autoscaler logs
kubectl logs <autoscaler-pod-name> -n kube-system

# update the node group scaling config (min=1, desired=1, max=3 nodes) to test the autoscaler
kubectl get nodes -n kube-system
kubectl logs <autoscaler-pod-name> -n kube-system

# deploy pods to cluster to check that autoscaler will run and scale up the nodes to schedule the new pods
kubectl get nodes -n kube-system
kubectl logs <autoscaler-pod-name> -n kube-system
```

Autoscaler might take some time to detect the need for autoscaling, so there's a tradeoff between using autoscaler and choosing the number of nodes in advance

### deploy our application to our EKS cluster

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  # an ELB on AWS will be created  
  type: LoadBalancer
```

```bash
kubectl apply -f nginx.yaml
```

scale the nginx deployment and increase the replicas to 20 to see that the autoscaler will scale up the nodes

```bash
kubectl get nodes
kubectl logs <autoscaler-pod-name> -n kube-system
```

scale the nginx deployment and decrease the replicas to 1 to see that the autoscaler will scale down the nodes

```bash
kubectl get nodes
kubectl logs <autoscaler-pod-name> -n kube-system
```

## 4 - Create Fargate Profile for EKS Cluster

Fargate is serverless, no EC2 instances in AWS account

Fargate creates a virtual machine for each pod

Fargate limitations:

- No support for stateful sets yet (StatefulSets expect stable, long-lived node assignments and predictable local storage)
- No support for daemon sets yet (DaemonSet relies on the concept of a physical or virtual worker node (like an EC2 instance) to schedule one copy of a pod on every node in the cluster. Fargate hides the node layer completely)

We can have both Node Group & Fargate profiles in the same cluster

Deploy to Fargate:

1) Create Fargate Role

Similar to Node Group Worker Role, Pods/Kubelet provisioned by Fargate needs permissions to connect to AWS, pull images, provision resources, ...etc

Create a new IAM role "eks-fargate-role" for service EKS "Fargate pod" with "AmazonEKSFargatePodExecutionRolePolicy"

1) Create Fargate Profile (Pod Selection Rule)

Specify which pods should use Fargate when they are launched

Create new Fargate Profile "dev-profile", the selection rules can either be defined based on k8s components namespace (namespace: dev) or labels (profile: fargate or environment: dev)

3) Deploy app to Fargate

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 2
  selector: 
    matchLabels:
      app: nginx
      profile: fargate
  template:
    metadata:
      labels:
        app: nginx
        profile: fargate
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80 
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

```bash
kubectl get pod

kubectl get node

kubectl create ns dev

kubectl apply -f nginx-fargate.yaml

# -W is wait flag, track status until pod is ready
kubectl get pod -n dev -w

# new fargate node for each pod
kubectl get nodes
```

You can create multiple Node Group or multiple Fargate profile for the cluster to separate the resources

To cleanup

- delete node group & fargate profile
- delete EKS cluster
- delete IAM roles
- delete cloudformation stack

## 5 - Create EKS cluster with eksctl command line tool

eksctl (eks-control) is a command line tool for working with EKS clusters that automates many individual tasks

Installation guide - https://github.com/eksctl-io/eksctl

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

eksctl use AWS config to connect to AWS (~/.aws/config), run `aws configure` & check `aws config list`

Create the cluster with below command

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

```bash
# check cluster is created
eksctl get cluster
```

you can also create the cluster by using eksctl config yaml

```bash
eksctl create cluster -f cluster.yaml
```

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-west-2
  version: "1.30"
nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
    ssh:
      allow: false
```

Above will create the Node Group, EC2 instances, Cluster & all necessary components. It'll take some time to complete 

kubectl will be automatically configured to connect to the newly created cluster (~/.kube/config)

Once cluster is ready, check AWS console for the newly created resources (IAM Roles, VPC, ASG, EC2, ..etc)

to delete the cluster

```bash
eksctl delete cluster <name>
```

## 6 - Deploy to EKS Cluster from Jenkins Pipeline

Create an empty K8s cluster using eksctl

Deploy Jenkins as a container & do the following

- install kubectl command line tool inside jenkins container
- install aws-iam-authenticator tool inside jenkins container
- create kubeconfig file to connect to the cluster (similar to ~/.kube/config)
- add AWS credentials on jenkins for AWS account authentication
- adjust jenkins file to configure EKS cluster deployment

1) install kubectl command line tool inside jenkins container

Run jenkins as a container and ssh to the server

execute root shell on jenkins container to install kubectl 

```bash
docker ps

# root shell on jenkins container
docker exec -u 0 -it <container-id> bash

# Install kubectl on Jenkins server
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl; chmod +x ./kubectl; mv ./kubectl /usr/local/bin/kubectl

# check installation
kubectl version
```

2) install aws-iam-authenticator tool inside jenkins container

```bash
# install aws-iam-authenticator
curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.6.11/aws-iam-authenticator_0.6.11_linux_amd64

chmod +x ./aws-iam-authenticator

mv ./aws-iam-authenticator /usr/local/bin

aws-iam-authenticator help
```

3) create kubeconfig file to connect to the cluster (similar to ~/.kube/config)

Below is a sample config file according to AWS documentation, you fill the placeholders from the (~/.kube/config) or the AWS console

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

Move the file inside the jenkins container in the directory (/var/jenkins_home/.kube/config)

```sh
# Copy config file to Jenkins server
docker cp config "YOUR DOCKER CONTAINER ID":/var/jenkins_home/.kube/
```

4) add AWS credentials on jenkins for AWS account authentication

We need credentials for AWS user (create IAM user for jenkins)

In Jenkins, create multi branch pipeline

Clone project repo & checkout the branch "deploy-to-k8s" https://gitlab.com/twn-devops-bootcamp/latest/11-eks/java-maven-app/-/blob/deploy-on-k8s/Jenkinsfile?ref_type=heads

In pipeline, add new credentials of type "secret text" for aws "access-key-id" & "secret-access-key". copy the values from (~/aws/credentials)

```text
jenkins_aws_access_key_id
jenkins-aws_secret_access_key
```

5) adjust jenkins file to configure EKS cluster deployment

```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    stages {
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                }
            }
        }
        stage('deploy') {
            // aws-iam-authenticator is executed in the background & following variables needs to be set
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
            }
            steps {
                script {
                   echo 'deploying docker image...'
                   sh 'kubectl create deployment nginx-deployment --image=nginx'
                }
            }
        }
    }
}
```

Push the changes to git, configure the pipeline to run only the branch "deploy-to-k8s" & run the pipeline to deploy to k8s

```bash
# check nginx is deployed to cluster
kubectl get pod
```

## 7 - BONUS: Deploy to LKE Cluster from Jenkins Pipeline

Create K8s cluster "test" on linode and then

- Install kubectl command line tool inside jenkins container
- Install kubernetes cli jenkins plugin (execute kubectl with kubeconfig credentials)
- configure jenkins file to deploy to LKE cluster

Download the cluster kubeconfig file "test-kubeconfig.yaml"

Connect to the cluster with the following

```bash
export KUBECONFIG=/path/to/test-kubeconfig.yaml

kubectl get node
```

In jenkins, add a new pipeline credential "lke-credentials" of type "secret file" and upload the "test-kubeconfig.yaml"

Install kubernetes cli jenkins plugin. restart jenkins to complete the installation, since jenkins is running as a container, you might need to ssh the server & restart the container

Create a new git branch "deploy-to-lke" and update the jenkins file with the following

```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    stages {
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    echo 'deploying docker image...'
                    withKubeConfig([credentialsId: 'lke-credentials', serverUrl: '<LINODE_K8S_API_ENDPOINT_URL>']) {
                        sh 'kubectl create deployment nginx-deployment --image=nginx'
                   }
                }
            }
        }
    }
}
```

push the changes & run the pipeline

```bash
# check nginx is deployed to cluster
kubectl get pod
```

## 8 - Jenkins Credentials Note on Best Practices

Instead of adding credentials in jenkins to connect to clusters on AWS & LKE, it's better to create a jenkins user (service account) in k8s with limited permissions to deploy the application

- deploy to specific namespace
- create deployment & service
- no delete, edit, create other components

Then create credentials for jenkins user with user tokens

## 9 - Complete CI/CD Pipeline with EKS and DockerHub

Create K8s cluster on AWS using eksctl

For instruction on how to build the pipeline & create the docker image, refer to module 8 "https://github.com/mustafa-saleh/demo-module-8-build-automation-and-ci-cd-with-jenkins"

To deploy the "java-maven-app" to K8s, create a deployment "kubernetes/deployment.yaml" & service "kubernetes/service.yaml" files 

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
---
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

Install "envsubst" in jenkins container

```bash
ssh root@server-ip

docker exec -it -u 0 <container-id> bash

# install envsubst
apt-get install gettext-base

envsubst --version
```

Create a secret in k8s to connect docker hub & pull the image

```bash
kubectl create secret docker-registry aws-registry-key \
--docker-server=docker.io \
--docker-username= \
--docker-password=

kubectl get secret
```

jenkins file

```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        DOCKER_REPO_SERVER = 'docker.io'
        DOCKER_REPO = "username/java-maven-app"
        // DOCKER_REPO_SERVER = '330673547330.dkr.ecr.eu-central-1.amazonaws.com'
        // DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
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
                        sh 'echo $PASS | docker login -u $USER --password-stdin ${DOCKER_REPO_SERVER}'
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
                   // envsubst resolve env vars defined in deployment & service files
                   sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                   sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update'){
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'gitlab-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "git remote set-url origin https://${USER}:${PASS}@gitlab.com/twn-devops-bootcamp/latest/11-eks/java-maven-app.git"
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

Push the changes & run the pipeline

```bash
# check result
kubectl get pod

kubectl get service

# check the image registry is docker hub
kubectl describe pod <pod-name>
```

## 10 - Complete CI/CD Pipeline with EKS and ECR

Steps

- Create ECR repository
- Create credentials in jenkins for ECR
- Adjust image building and tagging
- Create K8s secret to connect to ECR

Create private AWS ECR repository "java-maven-app". In case of microservices, n repository will be created one for each service

Create new credentials "ecr-credentials" in jenkins of type "username with password" (username: AWS, password: <ecr-password>)

```bash
# get ecr-password, valid for 12 hours
aws ecr get-login-password --region region

aws ecr get-login-password --region region | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.region.amazonaws.com
```

Create a secret in k8s to connect to ECR & pull the image

```bash
kubectl create secret docker-registry aws-registry-key \
--docker-server=<ecr-url> \
--docker-username=AWS \
--docker-password=<ecr-password>

kubectl get secret
```

Update the imagePullSecret in deployment to "aws-registry-key"

modify jenkins file 

Push the changes & run the pipeline

```bash
# check result
kubectl get pod

kubectl get service

# check the image registry is ECR
kubectl describe pod <pod-name>
```


