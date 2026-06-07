# cloud-programming — AWS static site

Portfolio submission for IU course **Cloud Programming (DLBSEPCP01_E)**. For Alp Ege Yalcin student n°42308653


This repository contains the Infrastructure as Code and the sample web
page for **Phase 2** of the portfolio: a globally available static
website hosted on AWS, defined entirely in a single CloudFormation
template.

## What's in this repo

| File             | What it is                                                        |
|------------------|-------------------------------------------------------------------|
| `template.yaml`  | The full CloudFormation template (S3 + CloudFront + OAC + CW alarm) |
| `index.html`     | A minimal "hello world" page that the site serves                 |
| `README.md`      | This file                                                         |

## What gets deployed

When you deploy the template, AWS creates:

- An **S3 bucket** that stores the site files. It is private — public
  access is blocked.
- A **CloudFront distribution** in front of the bucket. CloudFront
  caches the files at ~400 edge locations around the world, terminates
  HTTPS for the visitor, and forwards cache misses to S3.
- An **Origin Access Control** (OAC) — the modern AWS-recommended way
  to let CloudFront read from a private S3 bucket without ever making
  the bucket public.
- A **bucket policy** that grants CloudFront the right to read objects
  through the OAC.
- A **CloudWatch alarm** that fires if the CloudFront 5xx error rate
  exceeds the configured threshold.


## Architecture

See the Phase 1 PDF for the full architecture diagram, or page 2 of
the Phase 2 slide deck.

## How to deploy

You need an AWS account with the CLI configured (`aws configure`).

```bash
# 1. Create the stack. Pick a globally unique BucketName.
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name cloud-programming-site \
    --parameter-overrides BucketName=cloud-programming-<your-suffix>

# 2. Upload the page.
aws s3 cp index.html s3://cloud-programming-<your-suffix>/index.html

# 3. Get the URL.
aws cloudformation describe-stacks \
    --stack-name cloud-programming-site \
    --query 'Stacks[0].Outputs[?OutputKey==`SiteURL`].OutputValue' \
    --output text
```

The first deploy takes about 15 minutes because CloudFront rolls the
distribution out to its global edge network. Subsequent updates are
much faster.

## How to tear it down

```bash
# Empty the bucket first (CloudFormation refuses to delete non-empty buckets).
aws s3 rm s3://cloud-programming-<your-suffix> --recursive

# Then delete the stack.
aws cloudformation delete-stack --stack-name cloud-programming-site
```

## Notes for the tutor

- Single template, no external dependencies.
- Uses Origin Access Control (the current AWS recommendation), not the
  older Origin Access Identity.
- CloudWatch 5xx alarm included as a small monitoring example.
- TLS 1.2+ enforced, HTTPS-only via CloudFront.
- Static site is fronted by CloudFront for global low latency; the
  brief's "highly available", "low latency worldwide" and "autoscaling"
  requirements are addressed by the managed services themselves.

## Course context

Author: Alp Ege Yalcin
Course: DLBSEPCP01_E – Cloud Programming Matriculation n°42308653
Institution: IU International University of Applied Sciences
# cloud-programming
