# ☸️ Amazon EKS Deployment

A hands-on Amazon EKS project focused on deploying, exposing, scaling, monitoring, updating, and troubleshooting a containerized application running on Kubernetes in AWS.

Rather than stopping once the workload was running, I used this project to follow the full path from application source code to Docker, Amazon ECR, Kubernetes, AWS load balancing, autoscaling, and CloudWatch visibility.

![Amazon EKS Deployment Architecture](screenshots/00-eks-architecture.png)

## 🎯 Project Goals

The goal of this project was to move beyond basic Kubernetes deployment and explore how Kubernetes operates when running on AWS.

I wanted to practice:

- Creating an Amazon EKS cluster with managed EC2 worker nodes
- Deploying and exposing a Kubernetes application
- Building and versioning a custom Docker image
- Storing container images in Amazon ECR
- Performing Kubernetes rolling updates
- Testing Kubernetes self-healing
- Scaling application Pods manually
- Configuring Horizontal Pod Autoscaling
- Generating load and observing automatic scale-up and scale-down
- Monitoring resource utilization
- Sending EKS control-plane logs to Amazon CloudWatch
- Understanding the difference between Pod scaling and node scaling

## 🏗️ Architecture

Application traffic followed this path:

```text
Internet
   ↓
AWS Load Balancer
   ↓
Kubernetes LoadBalancer Service
   ↓
EKS Deployment
   ↓
ReplicaSet
   ↓
Application Pods
   ↓
EC2 Managed Worker Nodes
```

The EKS environment used two Availability Zones with managed EC2 worker nodes configured for private networking.

Application releases followed a separate path:

```text
HTML / CSS
    ↓
Docker Build
    ↓
Versioned Container Image
    ↓
Amazon ECR
    ↓
EKS Deployment Update
    ↓
New ReplicaSet
    ↓
Rolling Update
    ↓
Updated Application Pods
```

## 🧰 Technologies Used

- Amazon EKS
- Amazon EC2
- Amazon ECR
- Amazon VPC
- Elastic Load Balancing
- Amazon CloudWatch
- Kubernetes
- Docker
- nginx
- eksctl
- kubectl
- AWS CLI
- Linux / WSL
- Git
- GitHub

## ☁️ Building the EKS Cluster

The EKS cluster was created with `eksctl` using a managed node group distributed across two Availability Zones.

```yaml
managedNodeGroups:
  - name: mandy-workers
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 3
    privateNetworking: true
```

The configuration created two `t3.medium` EC2 worker nodes with the ability for the managed node group to scale to three nodes.

The worker nodes used private networking while Kubernetes workloads were managed through the Amazon EKS control plane.

![EKS worker nodes](screenshots/01-eks-worker-nodes-ready.png)

## 🚀 Deploying the Application

The application was deployed using a Kubernetes `Deployment` with three initial replicas.

```yaml
spec:
  replicas: 3
```

CPU and memory requests and limits were also configured for the container:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "250m"
    memory: "128Mi"
```

These values provided resource boundaries for the workload and later became important when configuring Horizontal Pod Autoscaling.

![Initial deployment](screenshots/02-initial-deployment-and-pods.png)

## 🔄 Deployment, ReplicaSets, and Pods

One of the goals of the project was to understand the relationship between the Kubernetes objects managing the application.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

The Deployment defines the desired application state.

A ReplicaSet maintains the requested number of Pods.

When the Deployment's Pod template changes, Kubernetes creates a new ReplicaSet and performs a rolling update instead of replacing every running Pod at once.

![Deployment and ReplicaSets](screenshots/03-deployment-replicasets-and-pods.png)

## 🌎 Exposing the Application

The application was exposed using a Kubernetes Service with type `LoadBalancer`.

```yaml
spec:
  type: LoadBalancer
```

AWS provisioned external load-balancing infrastructure for the Service, providing a public endpoint for the application.

This also demonstrated an important Kubernetes abstraction:

```text
External Endpoint
       ↓
Kubernetes Service
       ↓
Available Pods
```

The Service remained stable while the Pods behind it were deleted, recreated, scaled, and replaced during rolling updates.

## 🐳 Building a Custom Docker Image

The initial Kubernetes deployment used the standard nginx image.

I then created a custom HTML/CSS application and packaged it into my own nginx-based container image.

```dockerfile
FROM nginx:1.27-alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

The image was built locally and versioned rather than repeatedly overwriting the same image tag.

```text
Application Source
       ↓
Docker Build
       ↓
mandy-eks-demo:v1
       ↓
Amazon ECR
```

## 📦 Amazon ECR

A private Amazon ECR repository was created to store the application's container images.

The local Docker image was tagged with its ECR repository URI and pushed to AWS.

The EKS Deployment could then pull the private image from ECR using the AWS permissions associated with the worker infrastructure.

This connected the local container-development workflow to the Kubernetes deployment running in AWS.

## 🩹 Testing Kubernetes Self-Healing

To test Kubernetes desired-state reconciliation, I deliberately deleted one of the running application Pods.

Instead of manually recreating it, the ReplicaSet detected that the actual state no longer matched the desired state and immediately created a replacement.

The public application remained available during the test.

This reinforced one of the most important Kubernetes concepts I practiced in the project:

> Pods are disposable. Controllers maintain desired state.

## 📈 Manual Pod Scaling

The Deployment was manually scaled from three Pods to six.

```text
3 Pods
   ↓
6 Pods
```

All six Pods successfully scheduled across the existing two EC2 worker nodes.

![Manual pod scaling](screenshots/04-manual-pod-scaling.png)

The worker count remained at two because the existing nodes still had sufficient capacity.

That demonstrated an important distinction:

```text
Application Pod Scaling
          ≠
EC2 Node Scaling
```

Increasing the number of Pods does not automatically require additional EC2 instances when sufficient cluster capacity already exists.

## 📊 Monitoring Resource Usage

The Kubernetes Metrics Server provided visibility into CPU and memory consumption.

I inspected both worker-node and Pod-level utilization using:

```bash
kubectl top nodes
kubectl top pods
```

![Node and pod metrics](screenshots/05-node-and-pod-metrics.png)

This helped connect Kubernetes resource requests and limits with the workload's actual resource consumption.

## ⚡ Horizontal Pod Autoscaling

A Horizontal Pod Autoscaler was configured for the application with:

- Minimum replicas: `3`
- Maximum replicas: `6`
- CPU target: `50%`

The application requested:

```yaml
cpu: "100m"
```

per Pod.

This is important because HPA CPU utilization is calculated relative to the Pod's requested CPU rather than the total CPU capacity of the EC2 worker node.

### Generating Load

Temporary BusyBox Pods generated repeated requests to the application Service from inside the cluster.

As CPU utilization increased beyond the configured target, the HPA responded by increasing the number of application Pods.

The Deployment scaled from three Pods to four.

![HPA scale up](screenshots/06-hpa-scale-up-under-load.png)

After the load generators were removed, CPU utilization dropped.

Following the HPA stabilization period, Kubernetes reduced the Deployment back to its three-Pod baseline.

![HPA scale down](screenshots/07-hpa-scale-down-to-baseline.png)

The complete behavior looked like:

```text
Traffic Increases
       ↓
CPU Utilization Rises
       ↓
HPA Detects Target Exceeded
       ↓
Additional Pod Created
       ↓
Traffic Removed
       ↓
CPU Utilization Falls
       ↓
HPA Returns to Baseline
```

## 🖥️ Amazon CloudWatch Visibility

EKS control-plane logging was initially disabled.

I enabled the API and audit log types to provide additional visibility into cluster activity.

![CloudWatch logging](screenshots/08-cloudwatch-api-audit-logging.png)

Amazon EKS then published API server and audit log streams to CloudWatch.

![CloudWatch log streams](screenshots/09-cloudwatch-log-streams.png)

This connected Kubernetes administration with AWS-native operational visibility and demonstrated how activity occurring at the managed control-plane layer can still be observed through CloudWatch.

## 🔁 Versioned Application Release

After successfully deploying the first custom application image, I modified the HTML/CSS and created a second version.

Instead of overwriting `v1`, I built and pushed a new `v2` image.

```text
Application v1
      ↓
Modify Source
      ↓
Build Docker Image :v2
      ↓
Push :v2 to Amazon ECR
      ↓
Update EKS Deployment
      ↓
New ReplicaSet
      ↓
Rolling Update
      ↓
Application v2
```

Kubernetes created another ReplicaSet and gradually replaced the previous application Pods.

The public load-balancer endpoint did not change during the release.

This demonstrated how the application version could change independently of the external Service used to reach it.

## 🎉 Final Application

The final `v2` application displayed several characteristics of the environment directly in the interface:

- `3–6` application Pods managed by HPA
- `2` EC2 worker nodes
- Multi-AZ AWS architecture
- Amazon EKS running Kubernetes `1.36`

![Final EKS application](screenshots/11-final-v2-application.png)

## 🧠 What I Learned

### Kubernetes manages desired state

Deleting a Pod did not require manually rebuilding it. The ReplicaSet recognized that a replica was missing and restored the desired state.

### Pods and nodes are different scaling layers

Scaling from three to six Pods did not increase the EC2 worker count because the existing workers had enough capacity.

The application HPA controlled Pod replicas while the EKS managed node group had its own separate capacity configuration.

```text
HPA:
3 → 6 Pods

Managed Node Group:
2 → 3 EC2 Nodes
```

### Resource requests affect more than scheduling

The CPU request provided a scheduling requirement for the container, but it also became the reference point used by HPA when calculating CPU utilization.

### Kubernetes Services provide stability

Pods changed repeatedly throughout the project.

They were deleted, recreated, scaled, and replaced during rolling updates.

The Service provided a consistent networking layer in front of those changing Pods.

### Container versioning supports safer releases

Keeping separate `v1` and `v2` images preserved the previous application version instead of overwriting it.

That provides a much cleaner foundation for controlled deployments and rollback strategies.

### AWS and Kubernetes operate at different layers

Kubernetes handled:

- Deployments
- ReplicaSets
- Pods
- Services
- Desired state
- Pod autoscaling

AWS provided:

- Managed Kubernetes control plane
- EC2 worker infrastructure
- VPC networking
- External load balancing
- ECR container storage
- CloudWatch visibility

Understanding how those layers connect was one of the most valuable parts of the project.

## 📁 Repository Structure

```text
amazon-eks-deployment/
├── app/
│   ├── Dockerfile
│   └── index.html
├── screenshots/
│   ├── 00-eks-architecture.png
│   └── ...
├── .gitignore
├── deployment.yaml
├── eks-cluster.yaml
├── service.yaml
└── README.md
```

## 🔐 Security Considerations

No AWS credentials are stored in this repository.

The public Kubernetes manifest uses an ECR placeholder:

```yaml
image: YOUR_ECR_REPOSITORY_URI:v2
```

This prevents an account-specific ECR URI from being embedded in the public repository.

The worker nodes were configured with private networking, and AWS access relied on IAM rather than credentials stored in application files.

Resource requests and limits were also configured for the application containers.

## 🧹 Cleanup

Because EKS environments create billable AWS infrastructure, cleanup was treated as part of the project rather than an afterthought.

The cleanup process included removing:

```text
Kubernetes LoadBalancer Service
        ↓
External AWS Load Balancer
        ↓
HPA and Application Deployment
        ↓
EKS Cluster and Managed Node Group
        ↓
VPC / NAT Infrastructure
        ↓
ECR Repository
        ↓
CloudWatch Log Group
```

Resources were verified during teardown rather than assuming deletion had completed successfully.

## 💚 Final Takeaway

This project started as an Amazon EKS deployment, but the most valuable learning happened after the application was running.

I followed the workload through the complete path, built and versioned my own container images, watched Kubernetes maintain desired state, tested manual and automatic scaling, inspected real resource usage, performed rolling releases, enabled AWS-native logging, and cleaned up the cloud infrastructure afterward.

The project helped connect concepts that are often learned separately into one working system:

**application → container → registry → Kubernetes → AWS infrastructure → scaling → monitoring**

Built, scaled, broken, fixed, monitored, and probably caffeinated. ☁️☸️💚