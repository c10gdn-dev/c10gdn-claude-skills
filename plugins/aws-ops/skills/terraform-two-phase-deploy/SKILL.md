---
name: terraform-two-phase-deploy
description: Use when a Terraform deploy includes a container image (Lambda, ECS/Fargate) that must exist in a registry (ECR) before the compute resource referencing it can be created. Prevents the "image does not exist" apply failure on a brand-new deploy. Trigger on "Lambda apply fails because the image doesn't exist yet", "chicken-and-egg ECR/Lambda problem", or when scaffolding a new containerized serverless deploy from scratch.
---

# Terraform two-phase deploy (image-dependent compute)

## The problem

`terraform apply` on a fresh account fails when a `aws_lambda_function` (or ECS task
definition) references an image tag in an ECR repo that doesn't exist yet — but the
ECR repo itself is *also* created by this same Terraform config. You can't push an
image to a repo Terraform hasn't created, and Terraform won't create the Lambda
without a valid image reference.

## The fix: gate the image-dependent resource behind a variable

```hcl
variable "lambda_image_pushed" {
  type    = bool
  default = false
}

resource "aws_lambda_function" "api" {
  count         = var.lambda_image_pushed ? 1 : 0
  image_uri     = "${aws_ecr_repository.this.repository_url}:api"
  package_type  = "Image"
  # ...
}
```

Any resource that references the Lambda (Function URL, CloudFront origin, IAM
attachments, EventBridge target) needs the same `count = var.lambda_image_pushed ? 1 : 0`
guard, or a `try(aws_lambda_function.api[0].arn, null)` reference so plan doesn't
error when the count is 0.

## Deploy sequence

```bash
# Phase 1 — base infra only (ECR, IAM, VPC, DynamoDB, etc.)
terraform apply

# Build & push the image now that the ECR repo exists
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
docker build --platform linux/amd64 --provenance=false -t <account>.dkr.ecr.<region>.amazonaws.com/<repo>:api .
docker push <account>.dkr.ecr.<region>.amazonaws.com/<repo>:api

# Phase 2 — now create the image-dependent resources
terraform apply -var="lambda_image_pushed=true"
```

## Gotchas

- `--platform linux/amd64 --provenance=false` is required when building on Apple
  Silicon for Lambda/Fargate compatibility — omitting `--provenance=false` produces
  a manifest list Lambda can't run, with a confusing runtime error rather than a
  build error.
- If a later `terraform apply` is run *without* `-var="lambda_image_pushed=true"`,
  Terraform will happily destroy the Lambda again (count drops back to 0). Keep the
  var in a `.tfvars` file or CI pipeline default once past phase 1, don't rely on
  remembering the flag by hand.
- Same pattern applies to ECS Fargate task definitions referencing an ECR image —
  gate the `aws_ecs_service` (not just the task definition) since the service will
  fail to stabilize with a nonexistent image.
