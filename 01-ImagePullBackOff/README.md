# ImagePullBackoff

When a kubelet starts creating containers for a Pod using a container runtime, it might be possible the container is in Waiting state because of ImagePullBackOff.

The status ImagePullBackOff means that a container could not start because Kubernetes could not pull a container image for reasons such as 

- Invalid image name or 
- Pulling from a private registry without imagePullSecret. 

The BackOff part indicates that Kubernetes will keep trying to pull the image, with an increasing back-off delay.

Kubernetes raises the delay between each attempt until it reaches a compiled-in limit, which is 300 seconds (5 minutes).

# Kubernetes Production Troubleshooting Runbook: `ImagePullBackOff`

## 1. Overview

### What is `ImagePullBackOff`?

`ImagePullBackOff` is a Kubernetes Pod status indicating that Kubernetes was unable to pull the specified container image from the configured container registry.

Kubernetes will retry pulling the image, but because previous attempts failed, it progressively increases the delay between retries.

Common causes include:

* Incorrect image name
* Incorrect image tag
* Image does not exist in the registry
* Incorrect registry URL
* Private registry authentication failure
* Missing or incorrect `imagePullSecrets`
* Insufficient IAM permissions for Amazon ECR
* Incorrect AWS account or region
* Network/DNS connectivity problems
* Registry rate limiting
* Image architecture incompatibility
* The CI/CD pipeline built or pushed the wrong image/tag

---

# 2. First Step: Check the Pod

Start by identifying the affected Pod:

```bash
kubectl get pods -n <namespace>
```

Example:

```bash
kubectl get pods -n production
```

You may see:

```text
NAME                        READY   STATUS             RESTARTS   AGE
payment-api-7d8f9c8b7f      0/1     ImagePullBackOff   0          5m
```

Get more information:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Example:

```bash
kubectl describe pod payment-api-7d8f9c8b7f -n production
```

### Most important section: `Events`

Look at the bottom of the output.

For example:

```text
Events:
  Type     Reason     Message
  ----     ------     -------
  Normal   Pulling    Pulling image "myrepo/payment-api:1.2.3"
  Warning  Failed     Failed to pull image
  Warning  Failed     Error: ErrImagePull
  Warning  BackOff    Back-off pulling image
```

The **Events** section usually provides the most useful clue about why the image pull failed. Kubernetes specifically recommends `kubectl describe pod` when troubleshooting private-registry image-pull failures.

---

# 3. Check the Image Name

Check the image configured in the Deployment:

```bash
kubectl get deployment <deployment-name> -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Example:

```bash
kubectl get deployment payment-api -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Output:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api:1.2.3
```

Verify each component:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com
│
└── Registry

payment-api
│
└── Repository

1.2.3
│
└── Tag
```

Check for:

* Typographical errors
* Incorrect registry
* Incorrect AWS account
* Incorrect repository name
* Incorrect environment
* Incorrect region
* Incorrect tag

For example:

```text
payment-api:1.2.3
```

is different from:

```text
payment-api:1.2
```

and:

```text
payment-api:latest
```

---

# 4. Verify That the Image and Tag Exist

The image may be correctly configured in Kubernetes but may not actually exist in the registry.

For Docker-compatible registries, test the image from a machine that has access to the registry:

```bash
docker pull <registry>/<repository>:<tag>
```

Example:

```bash
docker pull docker.io/mycompany/payment-api:1.2.3
```

For Amazon ECR:

```bash
docker pull \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api:1.2.3
```

If the image cannot be pulled manually, investigate the registry, authentication, repository, or tag before troubleshooting Kubernetes itself.

---

# 5. Verify the Dockerfile

The `Dockerfile` determines how the container image is built.

Check:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Verify that:

* The base image exists.
* The programming-language version is compatible with the application.
* Required dependencies are installed.
* The correct application files are copied.
* The correct build arguments are being used.
* The image builds successfully.
* The resulting image is tagged correctly.

For example, if the application requires Python 3.12, but the Dockerfile uses:

```dockerfile
FROM python:3.8
```

the image may build successfully but the application could fail at runtime because of incompatible dependencies or language features.

### Important distinction

A bad application version or Dockerfile usually causes a **build-time or container-startup problem**, not necessarily `ImagePullBackOff`.

`ImagePullBackOff` primarily means Kubernetes cannot obtain the image.

Therefore, troubleshoot in this order:

```text
Can Kubernetes find the image?
        |
        v
Can Kubernetes authenticate?
        |
        v
Can Kubernetes download the image?
        |
        v
Can the container start?
        |
        v
Does the application work?
```

---

# 6. Check `imagePullSecrets`

For private registries, Kubernetes may need a Secret containing registry credentials.

Check the Pod:

```bash
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.spec.imagePullSecrets[*].name}'
```

Example output:

```text
dockerhub-secret
```

Check whether the Secret exists:

```bash
kubectl get secret dockerhub-secret -n production
```

If it doesn't exist:

```text
Error from server (NotFound)
```

then Kubernetes cannot use that Secret.

Kubernetes requires the image-pull Secret to exist in the **same namespace** as the Pod.

---

# 7. Check the Secret Type

Run:

```bash
kubectl get secret dockerhub-secret -n production \
  -o jsonpath='{.type}'
```

For a Docker registry Secret, you normally expect:

```text
kubernetes.io/dockerconfigjson
```

You can inspect the Secret metadata without exposing the credential contents:

```bash
kubectl describe secret dockerhub-secret -n production
```

Avoid printing or sharing decoded registry passwords/tokens.

---

# 8. Docker Hub Private Repository

For a private Docker Hub repository, Kubernetes needs credentials that have permission to pull the repository.

Example:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<token> \
  -n production
```

Then reference it in the Deployment:

```yaml
spec:
  template:
    spec:
      containers:
        - name: payment-api
          image: docker.io/mycompany/payment-api:1.2.3

      imagePullSecrets:
        - name: dockerhub-secret
```

Kubernetes documents this `imagePullSecrets` approach for private registries.

### Better operational approach

Instead of manually adding the Secret to every Deployment, you can associate the Secret with the ServiceAccount used by the Pods:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api-sa
  namespace: production
imagePullSecrets:
  - name: dockerhub-secret
```

Then configure the Deployment to use that ServiceAccount:

```yaml
spec:
  template:
    spec:
      serviceAccountName: payment-api-sa
```

---

# 9. Amazon ECR Troubleshooting

For ECR, first verify the complete image URL.

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api:1.2.3
```

Check:

```text
AWS Account ID
       |
       v
123456789012

AWS Region
       |
       v
ap-south-1

Repository
       |
       v
payment-api

Tag
       |
       v
1.2.3
```

Verify the repository:

```bash
aws ecr describe-repositories \
  --repository-names payment-api \
  --region ap-south-1
```

Verify the image/tag:

```bash
aws ecr describe-images \
  --repository-name payment-api \
  --region ap-south-1
```

You can specifically look for a tag:

```bash
aws ecr describe-images \
  --repository-name payment-api \
  --image-ids imageTag=1.2.3 \
  --region ap-south-1
```

---

# 10. ECR IAM Permissions

For ECR, make sure the identity used by the **Kubernetes image-pulling mechanism** has the required permissions.

For pulling from a private ECR repository, the relevant permissions commonly include:

```text
ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:BatchGetImage
ecr:GetDownloadUrlForLayer
```

AWS documents `ecr:GetAuthorizationToken` as required to authenticate to a private ECR registry, and ECR image access is controlled through IAM.

### Important distinction

Do not confuse these two identities:

```text
Jenkins IAM Role
       |
       |-- Builds image
       |-- Pushes image to ECR
       |
       v
      ECR


Kubernetes IAM / Node Role / Pod Identity
       |
       |-- Pulls image from ECR
       |
       v
      ECR
```

The Jenkins role needs permissions to **push** the image.

The identity used by the Kubernetes nodes/workload needs permissions to **pull** the image.

These are separate responsibilities.

---

# 11. EKS + ECR

If you are running Kubernetes on Amazon EKS, determine which AWS identity is responsible for the image pull.

Depending on your EKS configuration, this can involve:

* Node IAM role
* EKS Pod Identity
* IAM Roles for Service Accounts (IRSA)
* Other configured AWS credential mechanisms

Check the nodes:

```bash
kubectl get nodes
```

Then inspect a node:

```bash
kubectl describe node <node-name>
```

For EKS, also verify the IAM configuration associated with the relevant node/workload.

The key question is:

> **Which AWS identity is Kubernetes actually using when it attempts to pull this ECR image?**

Then verify that identity has the required ECR permissions.

---

# 12. Jenkins: Build and Push Verification

Jenkins is normally responsible for building and pushing the image.

A typical pipeline looks like:

```text
Developer
    |
    v
Git Repository
    |
    v
Jenkins
    |
    +--> Build application
    |
    +--> Build Docker image
    |
    +--> Tag image
    |
    +--> Authenticate to registry
    |
    +--> Push image
    |
    v
Container Registry
    |
    v
Kubernetes
    |
    +--> Pull image
    |
    v
Pod
```

Jenkins provides credential integration for registries and Pipeline steps such as `withDockerRegistry`; credentials should be stored in Jenkins Credentials rather than hard-coded in the Jenkinsfile.

---

# 13. Example Jenkins Pipeline for Docker Hub

Example:

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'docker.io/mycompany/payment-api'
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Build') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Push') {
            steps {
                withDockerRegistry(
                    credentialsId: 'dockerhub-credentials',
                    url: 'https://index.docker.io/v1/'
                ) {
                    sh '''
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }
    }
}
```

The Jenkins credential ID:

```text
dockerhub-credentials
```

must correspond to a credential configured in Jenkins.

Jenkins supports storing credentials centrally and referencing them by credential ID from Pipeline code.

---

# 14. Jenkins + Amazon ECR

A typical flow is:

```text
Jenkins
   |
   | AWS IAM credentials/role
   v
Amazon ECR Authentication
   |
   v
docker push
   |
   v
ECR Repository
```

Example authentication:

```bash
aws ecr get-login-password \
  --region ap-south-1 | \
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Then:

```bash
docker build \
  -t payment-api:1.2.3 .
```

Tag:

```bash
docker tag payment-api:1.2.3 \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api:1.2.3
```

Push:

```bash
docker push \
123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api:1.2.3
```

AWS documents ECR authentication and image push workflows in its ECR documentation.

---

# 15. Verify Jenkins Actually Pushed the Correct Image

A very common production problem is:

```text
Jenkins successfully builds image
        |
        v
Jenkins pushes image with tag 105
        |
        v
Kubernetes Deployment expects tag 104
        |
        v
ImagePullBackOff
```

Always compare:

### Jenkins

```text
Image:
payment-api:105
```

### Kubernetes

```bash
kubectl get deployment payment-api -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

If Kubernetes shows:

```text
payment-api:104
```

while Jenkins pushed:

```text
payment-api:105
```

then the problem is the deployment/versioning process rather than the Kubernetes registry authentication.

---

# 16. Check the Kubernetes Deployment

Get the complete Deployment:

```bash
kubectl get deployment payment-api -n production -o yaml
```

Look for:

```yaml
spec:
  template:
    spec:
      containers:
        - name: payment-api
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/payment-api:1.2.3
```

Check:

```text
Registry
Repository
Image
Tag
imagePullSecrets
ServiceAccount
```

---

# 17. Check the ServiceAccount

Find the ServiceAccount:

```bash
kubectl get pod <pod-name> -n production \
  -o jsonpath='{.spec.serviceAccountName}'
```

Then:

```bash
kubectl get serviceaccount <service-account> -n production -o yaml
```

For example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api-sa
  namespace: production
imagePullSecrets:
  - name: dockerhub-secret
```

If using an AWS workload identity mechanism, also verify the corresponding AWS identity configuration.

---

# 18. Check Network Connectivity

If the image name and authentication are correct, investigate network connectivity.

Potential problems:

```text
Kubernetes Node
      |
      X
      |
Container Registry
```

Check:

* DNS resolution
* Internet connectivity
* NAT Gateway
* Proxy configuration
* Firewall rules
* Security Groups
* Network ACLs
* VPC endpoints
* Private registry connectivity
* Registry availability

For ECR in a restricted/private AWS environment, verify that the required AWS connectivity and endpoints are correctly configured for your architecture.

---

# 19. Common Error Messages and Their Meaning

### `manifest unknown`

Usually means:

```text
Image or tag does not exist
```

Check:

```bash
docker pull <image>:<tag>
```

and verify the repository/tag.

---

### `pull access denied`

Usually indicates:

```text
Authentication or authorization problem
```

Check:

* Registry credentials
* `imagePullSecrets`
* Repository permissions
* IAM permissions

---

### `unauthorized`

Usually indicates:

```text
Registry authentication failed
```

Check the credentials and registry configuration.

---

### `FailedToRetrieveImagePullSecret`

Usually means Kubernetes cannot retrieve the specified image-pull Secret.

Check:

```bash
kubectl get secret <secret-name> -n <namespace>
```

The Secret must exist in the Pod's namespace.

---

### `no basic auth credentials`

Usually indicates that credentials required by the registry are missing or unavailable.

Check:

```yaml
imagePullSecrets:
  - name: registry-secret
```

---

### `repository does not exist`

Check:

```text
Registry
Repository name
AWS account
AWS region
```

For ECR:

```bash
aws ecr describe-repositories \
  --repository-names <repository> \
  --region <region>
```

---

### `context deadline exceeded`

Investigate:

```text
Network
DNS
Firewall
Proxy
NAT
Registry connectivity
```

---

# 20. Recommended Troubleshooting Decision Tree

Use this sequence during an incident:

```text
             ImagePullBackOff
                    |
                    v
       kubectl describe pod
                    |
                    v
          Check Events section
                    |
        +-----------+-----------+
        |                       |
        v                       v
  Image not found         Authentication
        |                  / Authorization
        |                       |
        v                       v
 Check image name          Check credentials
 Check repository          Check imagePullSecrets
 Check tag                 Check IAM
        |                       |
        +-----------+-----------+
                    |
                    v
             Registry access
                    |
                    v
             Network / DNS
                    |
                    v
             Image download
                    |
                    v
             Container starts
                    |
                    v
              Application
```

---

# 21. Production Troubleshooting Checklist

## Kubernetes

* [ ] Pod is in the correct namespace.
* [ ] Deployment references the correct image.
* [ ] Registry URL is correct.
* [ ] Repository name is correct.
* [ ] Image tag exists.
* [ ] `imagePullSecrets` is correctly configured when required.
* [ ] Image-pull Secret exists in the correct namespace.
* [ ] ServiceAccount is correct.
* [ ] Pod events have been checked.
* [ ] Node has registry connectivity.

## Docker Registry

* [ ] Repository exists.
* [ ] Image exists.
* [ ] Correct tag exists.
* [ ] Registry is available.
* [ ] Credentials are valid.
* [ ] Credentials have pull permission.

## Amazon ECR

* [ ] Correct AWS account.
* [ ] Correct AWS region.
* [ ] Correct ECR repository.
* [ ] Image/tag exists.
* [ ] `ecr:GetAuthorizationToken` is available.
* [ ] Required image-pull permissions are available.
* [ ] Correct IAM identity is being used by the Kubernetes workload/nodes.
* [ ] AWS network connectivity is available.

## Jenkins

* [ ] Correct source branch/version.
* [ ] Correct Dockerfile.
* [ ] Correct application/runtime version.
* [ ] Docker build succeeded.
* [ ] Correct image name was used.
* [ ] Correct image tag was used.
* [ ] Registry authentication succeeded.
* [ ] Image push succeeded.
* [ ] Jenkins credentials are valid.
* [ ] Jenkins AWS permissions are correct for ECR.
* [ ] Kubernetes Deployment references the same image/tag that Jenkins pushed.

---

# 22. Fast Production Investigation

When you're on-call, don't immediately start changing configuration.

Run these commands first:

```bash
kubectl get pod <pod> -n <namespace>
```

```bash
kubectl describe pod <pod> -n <namespace>
```

```bash
kubectl get deployment <deployment> -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.imagePullSecrets[*].name}'
```

```bash
kubectl get serviceaccount <service-account> -n <namespace> -o yaml
```

Then classify the failure:

```text
Wrong image/tag?
        |
        +--> Fix deployment/CI-CD

Image doesn't exist?
        |
        +--> Fix build/push pipeline

Authentication failure?
        |
        +--> Fix imagePullSecret / IAM

Authorization failure?
        |
        +--> Fix registry permissions

Network failure?
        |
        +--> Fix DNS / NAT / firewall / endpoint

Image pulls successfully but container fails?
        |
        +--> Stop troubleshooting ImagePullBackOff
             and investigate container startup/runtime
```

---

# 23. Key Interview Point

A strong way to explain `ImagePullBackOff` in an interview is:

> "`ImagePullBackOff` means Kubernetes is unable to pull the container image and is backing off before retrying. I first check the Pod events using `kubectl describe pod` to identify the exact error. Then I verify the image registry, repository, tag, and whether the image exists. If the registry is private, I verify `imagePullSecrets` and their namespace. For ECR, I verify the AWS account, region, repository, and the IAM identity used by the Kubernetes workload or nodes, including the required ECR pull permissions. I also verify that Jenkins actually built and pushed the same image and tag referenced by the Kubernetes Deployment. Finally, if authentication and image configuration are correct, I investigate DNS and network connectivity to the registry."

---

# 24. Useful Official Documentation

* Kubernetes private-registry/image-pull documentation: [Kubernetes — Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/?utm_source=chatgpt.com)
* Jenkins credentials documentation: [Jenkins — Using Credentials](https://www.jenkins.io/doc/book/using/using-credentials/?utm_source=chatgpt.com)
* Jenkins Docker Pipeline documentation: [Jenkins — Using Docker with Pipeline](https://www.jenkins.io/doc/book/pipeline/docker/?utm_source=chatgpt.com)
* Amazon ECR image pull documentation: [AWS — Pulling an Image from an Amazon ECR Private Repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-pull-ecr-image.html?utm_source=chatgpt.com)
* Amazon ECR IAM documentation: [AWS — IAM Permissions for ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-push-iam.html?utm_source=chatgpt.com)
