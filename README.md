# Enterprise DevSecOps Platform on Amazon EKS

# Overview

This project demonstrates a **production-grade Enterprise DevSecOps platform** deployed on **Amazon Elastic Kubernetes Service (Amazon EKS)** following modern cloud-native principles.

The platform hosts a containerized **Node.js backend**, and **MySQL database** on a highly available **multi-AZ Kubernetes cluster**.

Infrastructure is provisioned on AWS, while deployments are fully automated through **GitHub Actions**, incorporating multiple security scanning stages before application delivery.

Unlike many sample projects that rebuild applications for each environment, this platform promotes the **exact Docker image validated in QA** into production, ensuring deployment consistency and eliminating rebuild-related drift.

---

# Architecture Diagram

> **Note**
>
> The domain was purchased through **GoDaddy**, while DNS is hosted and managed using **Amazon Route 53**.

![Architecture](docs/images/architecture.png)

---

# High-Level Architecture

The platform is deployed inside a dedicated Amazon VPC spanning two Availability Zones for high availability.

User traffic enters through Amazon Route53, which manages the DNS records for the GoDaddy domain. HTTPS requests are secured using an AWS Certificate Manager (ACM) TLS certificate before reaching an Internet-facing Application Load Balancer.

The AWS Load Balancer Controller automatically provisions and manages the ALB based on Kubernetes Ingress resources.

Application workloads are deployed into private subnets across two managed EKS node groups, ensuring application servers remain inaccessible from the public internet.

Persistent application storage is dynamically provisioned using the Amazon EBS CSI Driver.

Sensitive application credentials never reside inside Git repositories. Instead, secrets are stored within AWS Secrets Manager and synchronized into Kubernetes using the External Secrets Operator.

Application container images are stored in a **private Docker Hub repository**, while deployments are orchestrated using GitHub Actions.

---

# Technology Stack

| Category | Technology |
|-----------|------------|
| Cloud Provider | AWS |
| Container Orchestration | Amazon EKS |
| Infrastructure as Code | Terraform |
| CI/CD | GitHub Actions |
| Source Control | GitHub |
| Container Runtime | Docker |
| Container Registry | Docker Hub (Private) |
| Frontend | Node.js |
| Database | MySQL |
| Ingress | AWS Load Balancer Controller |
| DNS | Amazon Route53 |
| Domain Registrar | GoDaddy |
| TLS Certificates | AWS Certificate Manager |
| Secrets Management | AWS Secrets Manager |
| Kubernetes Secrets | External Secrets Operator |
| Persistent Storage | Amazon EBS |
| Storage Provisioner | Amazon EBS CSI Driver |
| Authentication | GitHub OIDC |
| IAM | IRSA |
| Security Scanning | Gitleaks, Checkov, Trivy, SonarQube |
| SBOM | Anchore |

---

# Repository Structure

```
.
├── client/
│   ├── src/
│   ├── public/
│   └── Dockerfile
│
├── server/
│   ├── src/
│   ├── routes/
│   ├── controllers/
│   └── Dockerfile
│
├── terraform/
│   ├── vpc/
│   ├── eks/
│   ├── iam/
│   └── networking/
│
├── k8-manifests/
│   ├── qa/
│   │   ├── app-deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── mysql-statefulset.yaml
│   │
│   └── prod/
│       ├── app-deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── mysql-statefulset.yaml
│
├── .github/
│   └── workflows/
│       ├── qa-cicd.yml
│       └── prod-cd.yml
│
└── README.md
```

---

# Infrastructure Overview

The AWS infrastructure consists of:

- One Amazon VPC
- Two Availability Zones
- Two Public Subnets
- Two Private Subnets
- Internet Gateway
- NAT Gateway in each public subnet
- Amazon EKS Cluster
- Two Managed Node Groups
- Internet-facing Application Load Balancer
- Route53 Hosted Zone
- ACM TLS Certificate
- Amazon EBS CSI Driver
- External Secrets Operator
- AWS Secrets Manager
- IAM Roles for Service Accounts (IRSA)

This architecture ensures high availability, secure networking, and fault tolerance while keeping application workloads isolated within private subnets.

---

# AWS Infrastructure Components

## Amazon VPC

All resources are deployed inside a dedicated Virtual Private Cloud (VPC).

The VPC provides complete network isolation and acts as the security boundary for every AWS resource used in this project.

**CIDR Block**

```
10.0.0.0/16
```

---

## Availability Zones

The infrastructure spans **two Availability Zones**.

Benefits include:

- High Availability
- Fault Tolerance
- Reduced downtime
- Kubernetes pod distribution
- Node redundancy

If one Availability Zone experiences failure, workloads continue running in the remaining zone.

---

## Public Subnets

Each Availability Zone contains one public subnet.

Public subnets host components that require internet connectivity.

These include:

- NAT Gateway
- Application Load Balancer

Public subnet CIDRs

```
AZ-A
10.0.1.0/24

AZ-B
10.0.2.0/24
```

---

## Private Subnets

Application workloads are deployed inside private subnets.

No Kubernetes worker nodes receive public IP addresses.

Benefits include:

- Reduced attack surface
- Improved security
- Internet isolation
- Controlled outbound connectivity

Private subnet CIDRs

```
AZ-A
10.0.11.0/24

AZ-B
10.0.12.0/24
```

---

## Internet Gateway

The Internet Gateway enables communication between public AWS resources and the internet.

It allows:

- Public ALB access
- NAT Gateway internet connectivity
- Outbound communication from the VPC

Private workloads never communicate directly through the Internet Gateway.

---

## NAT Gateway

Each Availability Zone contains a dedicated NAT Gateway.

The NAT Gateway allows private Kubernetes nodes to access the internet without exposing them publicly.

Typical outbound traffic includes:

- Docker image pulls
- Package installation
- Operating system updates
- AWS API communication

Because worker nodes remain private, inbound internet traffic cannot reach them directly.

---

# Domain & DNS

## GoDaddy

The application domain is registered with **GoDaddy**.

GoDaddy functions only as the domain registrar.

DNS records are **not** managed there.

---

## Amazon Route 53

Route 53 manages the DNS hosted zone.

Responsibilities include:

- Domain resolution
- Alias record management
- Routing traffic to the Application Load Balancer

Separating domain registration from DNS hosting allows AWS networking services to be fully integrated while retaining ownership of the domain.

---

# AWS Certificate Manager (ACM)

All user traffic is encrypted using HTTPS.

AWS Certificate Manager provides and automatically renews the TLS certificate attached to the Application Load Balancer.

Benefits include:

- Automatic certificate renewal
- No certificate management overhead
- Secure HTTPS communication
- Native AWS integration

---

# Application Load Balancer

The Application Load Balancer is the public entry point into the Kubernetes cluster.

Responsibilities include:

- HTTPS termination
- Layer 7 routing
- Host-based routing
- Health checks
- Load balancing
- Integration with Kubernetes Ingress

The ALB itself is **not created manually**.

Instead, it is automatically provisioned by the AWS Load Balancer Controller whenever an Ingress resource is deployed.

---

# Amazon EKS Cluster

Amazon Elastic Kubernetes Service (EKS) serves as the orchestration platform for all containerized workloads.

The cluster manages:

- Scheduling
- Scaling
- Self-healing
- Networking
- Service discovery
- Rolling deployments

The control plane is fully managed by AWS, reducing operational overhead while maintaining Kubernetes compatibility.

---

# Managed Node Groups

The cluster contains two managed node groups.

```
Managed Node Group A
Availability Zone A

Managed Node Group B
Availability Zone B
```

AWS automatically manages:

- EC2 provisioning
- Auto Scaling integration
- Node replacement
- Node health monitoring
- Rolling updates

Deploying node groups across multiple Availability Zones improves application resilience.

---

# Application Pods

The application consists of three primary workloads:

### React Frontend

Provides the web interface served to end users.

Responsibilities:

- UI rendering
- API communication
- Static asset delivery

---

### Node.js Backend

Implements business logic and exposes REST APIs consumed by the frontend.

Responsibilities include:

- Request processing
- Database communication
- Authentication logic
- API responses

---

### MySQL StatefulSet

The database is deployed as a Kubernetes StatefulSet.

Unlike Deployments, StatefulSets preserve:

- Stable network identity
- Persistent storage
- Ordered startup and shutdown
- Stable pod naming

These characteristics make StatefulSets suitable for databases.

---

# Kubernetes Services

Services provide stable networking between Kubernetes workloads.

Rather than communicating directly with Pods, clients communicate with Services, which automatically route traffic to healthy backend Pods.

Benefits include:

- Service discovery
- Load balancing
- Stable IP addresses
- Pod replacement transparency

---

# Kubernetes Ingress

Ingress defines external HTTP and HTTPS routing rules.

Instead of exposing multiple Load Balancers, Kubernetes centralizes routing through a single Ingress resource.

Example responsibilities include:

- Host-based routing
- Path-based routing
- TLS configuration
- Backend service selection

---

# Persistent Storage

Stateless applications can be recreated at any time without losing data. Databases, however, require persistent storage to preserve information across pod restarts and node replacements.

To provide durable storage for MySQL, this project uses **Amazon Elastic Block Store (Amazon EBS)** together with the **Amazon EBS CSI Driver**.

This approach enables Kubernetes to dynamically provision storage volumes without requiring manual intervention.

---

# Amazon EBS CSI Driver

The Amazon EBS CSI (Container Storage Interface) Driver is installed as an Amazon EKS add-on.

Rather than manually creating EBS volumes through the AWS Console or Terraform for every database instance, Kubernetes requests storage automatically.

When a Persistent Volume Claim (PVC) is created, the CSI Driver performs the following operations:

1. Receives the storage request from Kubernetes.
2. Creates an Amazon EBS volume.
3. Attaches the volume to the EC2 worker node.
4. Mounts the volume inside the MySQL pod.
5. Detaches and reattaches the volume automatically if the pod moves to another node.

This eliminates manual storage management while allowing applications to request storage declaratively.

---

#  StorageClass

The cluster defines a StorageClass that determines how persistent storage should be provisioned.

Current configuration:

| Property | Value |
|----------|-------|
| Provisioner | Amazon EBS CSI Driver |
| Volume Type | gp2 |
| Volume Binding | WaitForFirstConsumer |

Using `WaitForFirstConsumer` delays volume creation until Kubernetes schedules the pod.

This ensures the EBS volume is created in the same Availability Zone as the node where the pod will run, preventing cross-AZ attachment issues.

---

#  Persistent Volumes

When the MySQL StatefulSet requests storage:

```
PersistentVolumeClaim
        │
        ▼
StorageClass
        │
        ▼
Amazon EBS CSI Driver
        │
        ▼
Amazon EBS Volume
        │
        ▼
Mounted into MySQL Pod
```

Even if:

- the MySQL container crashes,
- the pod is recreated,
- the node is replaced,

the data remains safely stored on the Amazon EBS volume.

---

#  MySQL StatefulSet

Unlike application pods, databases require stable identities and persistent storage.

For this reason, MySQL is deployed as a **StatefulSet** instead of a Deployment.

Benefits include:

- Stable pod names
- Stable DNS names
- Ordered startup
- Ordered shutdown
- Persistent volumes
- Data durability


---

#  Secrets Management

One of the core design principles of this project is **never storing secrets inside Git repositories**.

Instead of committing credentials into Kubernetes manifests, all application secrets are managed centrally in **AWS Secrets Manager**.

Examples include:

- Database username
- Database password
- Application secrets
- API credentials


---

#  AWS Secrets Manager

Sensitive application configuration is stored securely in AWS Secrets Manager.

Example secret:

```
qa/mysql-secret
```

The application never communicates directly with Secrets Manager.

Instead, a dedicated Kubernetes operator synchronizes secrets into the cluster.

Benefits include:

- Encryption at rest
- Fine-grained IAM access
- Centralized secret management
- Secret rotation support
- Version history

---

#  External Secrets Operator

The External Secrets Operator (ESO) continuously synchronizes secrets from AWS Secrets Manager into Kubernetes.

The synchronization process is automatic.

```
AWS Secrets Manager
          │
          ▼
External Secrets Operator
          │
          ▼
Kubernetes Secret
          │
          ▼
Application Pods
```

If a secret changes in AWS Secrets Manager, the operator automatically updates the corresponding Kubernetes Secret without requiring manual intervention.

This provides a secure and scalable approach to secret management.

---

#  Kubernetes Secrets

Although applications ultimately consume Kubernetes Secrets, these secrets are **not manually created**.

Instead, they are generated automatically by the External Secrets Operator.

Application pods reference the Kubernetes Secret exactly as they would any native Kubernetes resource, while the actual source of truth remains AWS Secrets Manager.

This separation simplifies application development while maintaining enterprise-grade security.

---

# 🛡️ IAM Roles for Service Accounts (IRSA)

One of the most important security features of this architecture is the use of **IAM Roles for Service Accounts (IRSA).**

Without IRSA, Kubernetes workloads typically require:

- AWS Access Key
- AWS Secret Key

inside Pods or Kubernetes Secrets.

This project completely avoids that approach.

Instead, AWS permissions are assigned directly to Kubernetes Service Accounts.

The authentication flow is:

```
Pod
 │
 ▼
Service Account
 │
 ▼
IAM Role (IRSA)
 │
 ▼
AWS STS
 │
 ▼
Temporary AWS Credentials
 │
 ▼
AWS Secrets Manager
```

The pod automatically receives short-lived AWS credentials without ever storing static credentials inside the cluster.


---

#  Least Privilege Access

The IAM role attached to the External Secrets Operator grants only the permissions required to retrieve secrets from AWS Secrets Manager.

No unnecessary AWS permissions are granted.

This follows the Principle of Least Privilege (PoLP), reducing the potential impact of compromised workloads.

---

#  Pod Security Groups

Application pods are associated with dedicated Pod Security Groups.

These security groups control network communication at the pod level rather than only at the worker node level.

Benefits include:

- Fine-grained traffic control
- Improved workload isolation
- Reduced lateral movement
- Enhanced security

---

#  High Availability

The infrastructure is designed to remain operational even if an Availability Zone becomes unavailable.

High availability is achieved through:

- Multi-AZ VPC
- Multi-AZ Managed Node Groups
- Multiple private subnets
- Multiple public subnets
- Redundant NAT Gateways
- Internet-facing Application Load Balancer
- Kubernetes self-healing
- Replica-based application deployments

If a worker node fails:

- Kubernetes automatically reschedules affected pods.
- Services continue routing traffic to healthy pods.
- The Application Load Balancer stops sending traffic to unhealthy targets.


---

#  Self-Healing

Amazon EKS continuously monitors application health.

If a container crashes:

- Kubernetes restarts the container.

If a pod fails:

- Kubernetes creates a replacement pod.

If a worker node becomes unhealthy:

- Managed Node Groups automatically replace the node.

This self-healing behavior ensures the desired application state is continuously maintained.

---

#  Scalability

The platform is designed to scale horizontally.

Application Deployments can be increased simply by raising the replica count.

Because Services automatically load balance across available pods, no infrastructure changes are required to distribute traffic.

Combined with the Application Load Balancer, this architecture can support increasing application demand while maintaining availability.

---
#  Enterprise DevSecOps CI/CD Pipeline

This project implements a complete **Enterprise DevSecOps CI/CD pipeline** using **GitHub Actions**.

The pipeline is designed around the principle of **Shift Left Security**, ensuring that security, code quality, testing, and compliance checks are executed before any deployment occurs.

The workflow follows a staged promotion model:

```
Developer
    │
    ▼
Push to QA Branch
    │
    ▼
QA CI/CD Pipeline
    │
    ├── Secret Scanning
    ├── IaC Scanning
    ├── Dockerfile Scanning
    ├── Kubernetes Manifest Scanning
    ├── Filesystem Vulnerability Scanning
    ├── Linting
    ├── Unit Testing
    ├── SonarQube Analysis
    ├── Docker Build
    ├── Image Vulnerability Scan
    ├── SBOM Generation
    ├── Push Image to Private Docker Hub
    ├── Update QA Manifest
    └── Deploy to QA EKS
            │
            ▼
QA Validation
            │
            ▼
Pull Request
            │
            ▼
Merge into Main
            │
            ▼
Production Promotion Pipeline
            │
            ├── Pull Tested QA Image
            ├── Retag Image
            ├── Push Production Tag
            ├── Update Production Manifest
            └── Deploy to Production EKS
```

---

#  QA Deployment Workflow

The QA pipeline is automatically triggered whenever code is pushed to the **qa** branch.

```yaml
on:
  push:
    branches:
      - qa
```


---

#  Stage 1 – Secret Scanning

The pipeline begins by detecting accidentally committed secrets using **Gitleaks**.

Examples include:

- AWS Keys
- GitHub Tokens
- Database Passwords
- API Keys
- Private Keys
- Authentication Tokens

Scanning source code before building Docker images prevents sensitive information from entering downstream environments.

---

#  Stage 2 – Infrastructure Security Scanning

Infrastructure definitions are scanned using **Checkov**.

Three different scan types are executed independently.

## Terraform Scan

Terraform configurations are analyzed for:

- Publicly exposed resources
- Weak IAM permissions
- Security Group misconfigurations
- Encryption settings
- Networking best practices

---

## Kubernetes Manifest Scan

Kubernetes manifests are scanned for common security issues such as:

- Privileged containers
- Missing resource limits
- Host networking
- Insecure capabilities
- Missing security contexts

---

## Dockerfile Scan

Dockerfiles are analyzed for insecure build practices including:

- Running containers as root
- Untrusted base images
- Missing HEALTHCHECK instructions
- Excessive privileges
- Package management issues

---

#  Stage 3 – Filesystem Vulnerability Scanning

Before building container images, **Trivy** scans both application directories.

Separate scans are executed for:

- Client
- Server

The scans identify:

- Vulnerable packages
- Known CVEs
- Critical dependencies
- High-risk libraries

Both JSON and human-readable reports are uploaded as GitHub Action artifacts.

---

#  Stage 4 – Code Quality

Application quality is verified through linting.

Separate lint jobs are executed for:

- Node.js frontend

Linting helps detect:

- Syntax errors
- Code style violations
- Potential programming mistakes

This improves maintainability and consistency.

---

#  Stage 5 – Unit Testing

After linting completes successfully, automated tests are executed.


Testing before containerization helps ensure application functionality before deployment.

---

#  Stage 6 – SonarQube Analysis

Once testing completes, the pipeline performs static code analysis using SonarQube.

The analysis evaluates:

- Bugs
- Code Smells
- Security Hotspots
- Vulnerabilities
- Technical Debt
- Code Coverage

The pipeline also performs a **Quality Gate** check before continuing.

This provides continuous insight into overall code quality.

---

#  Stage 7 – Docker Build

After all security and quality checks pass, the application image is built.

Two image tags are generated.

```
latest

<git-sha>
```

Using immutable Git commit hashes allows every deployment to be traced back to the exact source code revision.

---

#  Stage 8 – Container Image Scanning

Before publishing the image, Trivy scans the built container image.

The scan checks:

- Operating System packages
- Installed libraries
- Language dependencies
- Known vulnerabilities
- Critical CVEs

Only after image scanning completes is the image considered ready for publication.

---

#  Stage 9 – Software Bill of Materials (SBOM)

To improve software supply chain transparency, the pipeline generates Software Bills of Materials (SBOMs).

Two SBOMs are produced.

- Source Code SBOM
- Container Image SBOM

These artifacts provide complete inventories of dependencies included within the application.

---

#  Stage 10 – Docker Hub Private Registry

This project stores container images in a **private Docker Hub repository**.

The pipeline authenticates using GitHub Secrets before pushing images.

Published image tags include:

```
humayun27/nodejs-app:latest

humayun27/nodejs-app:<git-sha>
```

Maintaining a private registry ensures application images remain inaccessible to unauthorized users while providing centralized image storage.

---

#  Stage 11 – GitOps Manifest Update

Once the image has been published, GitHub Actions automatically updates the Kubernetes deployment manifest.

The image reference inside the QA deployment is replaced with the newly generated Git commit SHA.

Example:

Before

```yaml
image: humayun27/nodejs-app:latest
```

After

```yaml
image: humayun27/nodejs-app:26367361c959c67d4aa0d003d0544d9
```

The updated manifest is committed back to the **qa** branch.

This creates a GitOps-style deployment history where Git always reflects the exact version deployed to Kubernetes.

---

#  Stage 12 – QA Deployment

Finally, GitHub Actions authenticates with AWS using **GitHub OIDC**, generates a kubeconfig for the EKS cluster, and applies the Kubernetes manifests to the **qa** namespace.

After deployment, the pipeline verifies that the rollout completed successfully before marking the workflow as successful.

---

#  Promotion to Production

Unlike many CI/CD implementations, the production pipeline **does not rebuild the application**.

Instead, it promotes the exact Docker image that was already deployed and validated in the QA environment.

The workflow is:

1. A developer opens a Pull Request from the **qa** branch to the **main** branch.
2. After QA approval, the Pull Request is merged.
3. The production workflow pulls the `latest` image from the private Docker Hub repository.
4. The image is retagged using the format:

```text
prod-<git-sha>
```

5. The retagged image is pushed back to the private Docker Hub repository.
6. The production Kubernetes manifest is updated with the new production image tag.
7. GitHub Actions deploys the updated manifests to the **prod** namespace in Amazon EKS.
8. Kubernetes performs a rolling update and verifies the rollout.

This approach guarantees that **the exact artifact tested in QA is the one deployed to production**, eliminating the risk of environment drift caused by rebuilding images.

---

#  Secure AWS Authentication with GitHub OIDC

The deployment workflows authenticate to AWS using **GitHub OpenID Connect (OIDC)**.

Instead of storing long-lived AWS access keys as GitHub Secrets, GitHub requests a short-lived identity token, which AWS Security Token Service (STS) exchanges for temporary credentials.

Benefits include:

- No static AWS access keys
- Temporary credentials
- Automatic credential rotation
- Reduced credential leakage risk
- Least privilege access through IAM roles


---

#  Author

**Humayun Ch**

DevOps | Cloud | Kubernetes | AWS | Terraform | CI/CD | DevSecOps

If you found this project helpful, consider giving it a ⭐ on GitHub.
