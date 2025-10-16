# terraform-aws-ak-containerized

A production‑ready Terraform module that deploys a containerized HTTP application on AWS using ECS Fargate behind an internal ALB, fronted by Amazon API Gateway (HTTP API) via a VPC Link. It optionally provisions networking (VPC/subnets) and a Redis cluster for application state.

This module is intended for publishing as a reusable public module. It encapsulates best practices for: immutable image builds to ECR, least‑privilege security groups, health checks, CloudWatch logging, and a clean API surface.

Highlights
- ECS Fargate service (no servers to manage)
- Internal Application Load Balancer (ALB)
- API Gateway v2 (HTTP API) with VPC Link -> ALB integration
- Optional VPC creation, or plug into an existing VPC
- Optional Redis cluster for state/memory (exposed as AK_REDIS_URL)
- Cloud Map HTTP namespace and Service Connect integration
- CloudWatch Logs for API Gateway and ECS

Supported
- Terraform >= 1.9.5
- AWS provider ~> 6.11.0
- Docker provider ~> 3.6.2

Architecture
- API Gateway (HTTP API) exposes a public route: POST /api/{api_version}/{agent_endpoint}
- VPC Link connects API Gateway to an internal ALB
- ALB targets the ECS service in private subnets
- ECS service runs a single container (your app image) on Fargate
- Optional Redis cluster URL is injected via environment variables as AK_REDIS_URL

What this module creates
- API Gateway v2 HTTP API and Stage
- API Gateway VPC Link to internal ALB
- CloudWatch Log Group for API Gateway access logs
- ECS Cluster and ECS Service (via terraform-aws-modules/ecs/aws)
- Cloud Map HTTP namespace and Service Connect configuration
- Internal ALB, Target Group, and Listener
- Security Groups for ALB and ECS service
- VPC/Subnets (optional) via an external VPC module
- ECR build & push for your container image via a helper module

Usage
Basic example (new VPC created by the module dependencies)

module "ak_containerized" {
  source = "github.com/your-org/terraform-aws-ak-containerized?ref=v0.1.0"

  region             = "us-east-1"
  product_alias      = "acme"
  env_alias          = "dev"
  module_name        = "agent"
  package_path       = "./app"   # path to your app root with Dockerfile

  # optional overrides
  api_version        = "v1"
  agent_endpoint     = "chat"
  ecs_container_port = 8000
  environment_variables = {
    APP_ENV = "dev"
  }
  tags = {
    Project = "AcmeAgent"
  }
}

Using an existing VPC

module "ak_containerized" {
  source = "github.com/your-org/terraform-aws-ak-containerized?ref=v0.1.0"

  region        = "us-east-1"
  product_alias = "acme"
  env_alias     = "prod"
  module_name   = "agent"
  package_path  = "./app"

  # Attach to an existing VPC
  vpc_id             = "vpc-0123456789abcdef0"
  private_subnet_ids = ["subnet-aaa", "subnet-bbb"]

  create_redis_cluster = true
}

After apply, the output agent_invoke_url provides the fully qualified API endpoint for POST requests, e.g.:

https://abcdefghij.execute-api.us-east-1.amazonaws.com/agents/api/v1/chat

Note: The module configures the API Gateway integration to forward requests to the ALB path /run. Ensure your containerized app serves the POST /run endpoint accordingly. By default, the ECS health check hits GET {ecs_health_check_path} on the container (default: /health).

Inputs
- region (string) Required. AWS region for all resources.
- product_alias (string) Required. Short product alias used in names and tags.
- env_alias (string) Required. Environment alias (e.g., dev, staging, prod).
- product_display_name (string) Optional. Display name for API description. Default: "An Agent Kernel deployment".
- module_name (string) Required. Logical module/workload name used in resource names.
- package_path (string) Required. Path to the container app root (must include a Dockerfile). Used to build/push image to ECR.
- environment_variables (any) Optional. Extra environment variables to inject into the container. Default: {}. AK_REDIS_URL is added automatically when Redis is enabled.
- api_version (string) Optional. API version path segment. Default: v1.
- agent_endpoint (string) Optional. API endpoint path segment. Default: chat.
- tags (map(string)) Optional. Tags applied to resources. Default: {}.
- vpc_cidr (string) Optional. CIDR for a new VPC (when vpc_id is not provided). Default: 10.0.0.0/16.
- public_subnet_cidrs (list(string)) Optional. Public subnet CIDRs for new VPC. Default: [10.0.1.0/24, 10.0.2.0/24].
- private_subnet_cidrs (list(string)) Optional. Private subnet CIDRs for new VPC. Default: [10.0.3.0/24, 10.0.4.0/24].
- vpc_id (string) Optional. Existing VPC ID. If set, no new VPC is created.
- private_subnet_ids (list(string)) Optional. Required when using existing VPC. Private subnet IDs for ECS/ALB.
- create_redis_cluster (bool) Optional. Whether to create a Redis cluster and expose AK_REDIS_URL. Default: false.
- ecs_cpu (number) Optional. Fargate CPU units per task. Default: 256.
- ecs_memory (number) Optional. Fargate memory (MiB) per task. Default: 512.
- ecs_desired_count (number) Optional. Desired ECS service task count. Default: 1.
- ecs_container_port (number) Optional. Container port to expose. Default: 8000.
- ecs_health_check_path (string) Optional. Health check path on container for target group. Default: /health.
- container_type (string) Optional. Container orchestrator type. Supported: ecs, eks. Default: ecs.

Outputs
- alb_dns_name: Internal ALB DNS name.
- cluster_arn: ARN of the ECS cluster.
- api_gateway_id: API Gateway HTTP API ID.
- api_gateway_stage: Name of the API Gateway stage (default: agents).
- agent_invoke_url: Fully qualified invoke URL for POST requests to your app (includes stage + /api/{api_version}/{agent_endpoint}).

Providers and Version Pinning
- Terraform >= 1.9.5
- AWS provider 6.11.0
- Docker provider 3.6.2

Prerequisites
- An AWS account and credentials with permissions to create the listed resources, including ECR, ECS, API Gateway v2, CloudWatch Logs, VPC networking, Security Groups, and ALB.
- A Dockerfile at package_path. The module builds and pushes the image to ECR using a helper module.
- If using an existing VPC, ensure provided private_subnet_ids belong to vpc_id and have required route/NAT access for pulling images.

Important notes and caveats
- API routing: The module configures API Gateway -> VPC Link -> ALB and overwrites the path to /run at the ALB. Your application should implement POST /run accordingly. The public API route created is POST /api/{api_version}/{agent_endpoint} and is mapped to ALB /run.
- Internal ALB: The ALB is created as internal=true. API Gateway exposes the public edge. The ALB is not internet-facing.
- Redis: When create_redis_cluster = true, AK_REDIS_URL is added as an environment variable to your container. Your app should read it if it uses Redis.
- Logs: API Gateway access logs are enabled to CloudWatch Logs; ECS uses awslogs driver with group /ecs/{product_alias}-{env_alias}-{module_name}.
- Security Groups: ALB SG allows HTTP 80 from within the VPC CIDR; ECS service SG only allows traffic from the ALB SG on ecs_container_port.

Make a request
- After apply, run a POST request to the output agent_invoke_url.
- Example curl:
  curl -X POST "$(terraform output -raw agent_invoke_url)" \
    -H 'Content-Type: application/json' \
    -d '{"message":"Hello"}'

Costs
- Resources incur AWS charges (API Gateway, ECS, Fargate, ALB, CloudWatch Logs, VPC components, and optional Redis). Review pricing before use.

Limitations and external dependencies
- The module references the following external modules:
  - VPC: app.terraform.io/yaalalabs/ak-vpc/aws (private registry). You need access to this module or replace it with your own.
  - Redis: ../common/redis (relative). In a public registry release, replace with a published Redis module or vendor it.
  - ECR/Image build: ../common/ecr (relative). In a public registry release, replace with a published ECR module or vendor it.
- If you publish this module to the public registry, ensure you update these sources to public, versioned modules.

Development
- Lint and validate: terraform fmt -check && terraform validate
- Plan/apply: terraform init && terraform plan && terraform apply
- Pin provider versions and module versions to ensure reproducibility.

Versioning
- Follow SemVer for module releases. Pin module version in your root module (e.g., ?ref=v0.1.0 or >= 0.1.0, < 0.2.0).

Contributing
- Issues and PRs are welcome. Please include: context, expected vs. actual behavior, and reproduction steps.

License
- See LICENSE in this repository.
