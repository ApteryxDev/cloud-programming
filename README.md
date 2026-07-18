<div align="center">

# ☁️ Cloud Programming : Secure AWS Static Site

**A fonctional, serverless static website delivered through AWS CloudFront and provisioned with CloudFormation.**

[![AWS](https://img.shields.io/badge/AWS-Cloud_Platform-232F3E?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![CloudFormation](https://img.shields.io/badge/IaC-AWS_CloudFormation-FF4F8B?logo=amazonaws&logoColor=white)](template.yaml)
[![S3](https://img.shields.io/badge/Storage-Amazon_S3-569A31?logo=amazons3&logoColor=white)](https://aws.amazon.com/s3/)
[![CloudFront](https://img.shields.io/badge/CDN-Amazon_CloudFront-8C4FFF?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudfront/)
[![HTTPS](https://img.shields.io/badge/Transport-HTTPS_only-success?logo=letsencrypt&logoColor=white)](#-security-by-design)

*IU International University of Applied Sciences · DLBSEPCP01_E · Cloud Programming*

</div>

---

## 🎯 Project objective

This repository implements a **highly available, low-maintenance static website on AWS**. The infrastructure is declared in one CloudFormation template so the environment can be reviewed, reproduced, updated, and removed consistently.

The solution addresses the portfolio brief through managed AWS services:

- **Global delivery:** CloudFront caches content close to visitors.
- **Elastic scale:** S3 and CloudFront scale without server provisioning.
- **Secure origin:** the S3 bucket remains private and is accessible only through CloudFront Origin Access Control.
- **Repeatability:** the complete infrastructure is described as code.
- **Observability:** a CloudWatch alarm monitors CloudFront 5xx errors.

> The course task asks for an understandable and replicable architecture, supported by Infrastructure as Code, that considers availability, worldwide latency, autoscaling, and security.

## 🧭 Architecture at a glance

```text
                           HTTPS
┌──────────────┐       ┌──────────────────────┐
│ Global user  │ ────▶ │ Amazon CloudFront    │
└──────────────┘       │ CDN · cache · TLS    │
                       └──────────┬───────────┘
                                  │ signed request (OAC)
                                  ▼
                       ┌──────────────────────┐
                       │ Private Amazon S3    │
                       │ index.html · versioned│
                       └──────────────────────┘

                       ┌──────────────────────┐
                       │ Amazon CloudWatch    │
                       │ 5xx error-rate alarm │
                       └──────────────────────┘
```

### Request flow

1. A visitor requests the website through the CloudFront URL.
2. CloudFront redirects HTTP traffic to HTTPS and checks its cache.
3. On a cache miss, CloudFront signs the origin request through **Origin Access Control (OAC)**.
4. The private S3 bucket returns the requested object.
5. CloudFront compresses, caches, and serves the response.

## 🧩 AWS components

| Component | Responsibility | Design rationale |
|---|---|---|
| **Amazon S3** | Stores `index.html` | Durable managed object storage with versioning and blocked public access |
| **Amazon CloudFront** | Delivers and caches the website | Reduces latency, provides HTTPS, compression, IPv6, and managed scaling |
| **Origin Access Control** | Authenticates CloudFront to S3 | Keeps the origin private and uses SigV4-signed requests |
| **S3 bucket policy** | Restricts origin access | Grants `s3:GetObject` only to the deployed CloudFront distribution |
| **Amazon CloudWatch** | Monitors availability signals | Raises an alarm when the CloudFront 5xx error rate exceeds the parameterized threshold |
| **AWS CloudFormation** | Provisions the stack | Makes the solution readable, repeatable, and easy to tear down |

## 🗂️ Repository structure

```text
cloud-programming/
├── template.yaml   # CloudFormation: S3, CloudFront, OAC, policy and alarm
├── index.html      # Sample static webpage
└── README.md       # Architecture, deployment and operational guidance
```

## ✨ Infrastructure highlights

| Area | Implementation |
|---|---|
| **Availability** | Managed S3 storage and CloudFront distribution; no single application server |
| **Performance** | CloudFront caching, compression, HTTP/2, IPv6, and optimized cache policy |
| **Security** | Private bucket, blocked public ACLs/policies, OAC, HTTPS redirect, TLS 1.2+ |
| **Cost control** | Serverless static hosting and `PriceClass_100` for a lower-cost US/Europe edge footprint |
| **Resilience** | S3 object versioning and custom CloudFront handling for 403/404 responses |
| **Monitoring** | Parameterized CloudWatch alarm for the distribution's 5xx error rate |
| **Maintainability** | One declarative YAML template with explicit parameters and outputs |

## 🔐 Security by design

The S3 bucket is **not a public website endpoint**. All public access controls are enabled, object ownership is enforced by the bucket owner, and CloudFront is the only permitted reader. The bucket policy validates the CloudFront distribution ARN before granting access.

Viewer traffic is redirected to HTTPS, and the distribution enforces the `TLSv1.2_2021` minimum protocol policy.

## 🚀 Deploy

### Prerequisites

- An AWS account
- AWS CLI installed
- Credentials configured with permission to create CloudFormation, S3, CloudFront, IAM-related resource policies, and CloudWatch resources

Confirm the CLI session:

```bash
aws sts get-caller-identity
```

### 1. Validate the template

```bash
aws cloudformation validate-template \
  --template-body file://template.yaml
```

### 2. Create or update the stack

Choose a globally unique, DNS-compatible bucket name:

```bash
export STACK_NAME="cloud-programming-site"
export BUCKET_NAME="cloud-programming-<your-unique-suffix>"

aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name "$STACK_NAME" \
  --parameter-overrides \
      BucketName="$BUCKET_NAME" \
      DefaultRootObject=index.html \
      ErrorRateThreshold=5
```

### 3. Upload the webpage

```bash
aws s3 cp index.html "s3://$BUCKET_NAME/index.html" \
  --content-type text/html
```

### 4. Retrieve the public URL

```bash
SITE_URL=$(aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].Outputs[?OutputKey==`SiteURL`].OutputValue' \
  --output text)

echo "$SITE_URL"
```

CloudFront distribution creation can take several minutes. A newly uploaded object may also require a brief propagation period.

## ✅ Verify the deployment

```bash
curl -I "$SITE_URL"
```

Expected checks:

- the URL uses `https://`;
- the response is served through CloudFront;
- the root path returns the uploaded page;
- direct anonymous access to the S3 object is denied;
- the CloudFormation stack exposes `SiteBucketName`, `DistributionId`, and `SiteURL` outputs.

<details>
<summary><strong>Optional: invalidate CloudFront after an update</strong></summary>

```bash
DISTRIBUTION_ID=$(aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].Outputs[?OutputKey==`DistributionId`].OutputValue' \
  --output text)

aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths "/*"
```

</details>

## 🧹 Tear down

S3 buckets must be empty before CloudFormation can delete them.

```bash
aws s3 rm "s3://$BUCKET_NAME" --recursive
aws cloudformation delete-stack --stack-name "$STACK_NAME"
aws cloudformation wait stack-delete-complete --stack-name "$STACK_NAME"
```

## ⚖️ Design decisions and trade-offs

### Why S3 + CloudFront?

A static site does not require an EC2 instance, load balancer, container platform, or custom autoscaling policy. S3 and CloudFront provide the required storage, delivery, and elasticity with less operational overhead.

### Why Origin Access Control?

OAC lets CloudFront sign requests to a private S3 origin. This avoids exposing the bucket publicly and supersedes the older Origin Access Identity approach for this architecture.

### Why `PriceClass_100`?

The template intentionally selects the lower-cost CloudFront price class, which uses edge locations in the United States, Canada, and Europe. This is a cost-conscious portfolio choice, but it is not the broadest possible global edge footprint. A production system targeting uniformly low latency worldwide could use a broader price class.

### What would be added in production?

- a custom domain in Route 53;
- an ACM certificate for that domain;
- AWS WAF rules;
- access logging and dashboards;
- deployment automation through CI/CD;
- budget alarms and cost allocation tags;
- a response-headers policy for browser security headers.

## 📌 Portfolio alignment

The repository supports the implementation and finalisation aspects of the course portfolio by providing:

- a working Infrastructure-as-Code definition;
- concise architecture and service rationale;
- reproducible installation and removal instructions;
- security, scalability, availability, cost, and monitoring considerations;
- explicit reflection on trade-offs and possible improvements.

## 🛠️ Built with

<p align="center">
  <img src="https://skillicons.dev/icons?i=aws" alt="AWS" height="44" />
  &nbsp;&nbsp;
  <img src="https://cdn.simpleicons.org/amazoncloudwatch/FF4F8B" alt="Amazon CloudWatch" height="42" />
  &nbsp;&nbsp;
  <img src="https://cdn.simpleicons.org/yaml/CB171E" alt="YAML" height="42" />
  &nbsp;&nbsp;
  <img src="https://cdn.simpleicons.org/html5/E34F26" alt="HTML5" height="42" />
</p>

---

<div align="center">

**Alp Ege Yalcin** · Matriculation No. **42308653**  
DLBSEPCP01_E : Cloud Programming  
IU International University of Applied Sciences

</div>
