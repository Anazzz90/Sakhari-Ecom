# 0030. Production deployment topology

**Status:** Accepted

## Context

The infrastructure documentation established AWS `me-central-2`, RDS, Redis, S3, CDN, and a single backend deployable, but intentionally left the concrete production compute service, network topology, CI/CD tool, and backup parameters open.

## Problem

Implementation cannot produce a production-shaped system consistently while the deployment substrate remains abstract.

## Options Considered

1. EC2 instance with Docker Compose.
2. AWS ECS Fargate behind an Application Load Balancer.
3. EKS/Kubernetes.
4. Serverless functions.

## Decision

Use AWS ECS Fargate for the production backend, behind an Application Load Balancer.

Production topology:

- VPC with public subnets for ALB and private subnets for ECS tasks, RDS, and Redis.
- ECS Fargate service runs the NestJS modular monolith as one deployable container.
- RDS PostgreSQL runs Multi-AZ in Production, Single-AZ allowed only in Development/Staging.
- Redis uses managed ElastiCache in private subnets.
- S3 stores media assets; CloudFront serves public cacheable assets.
- AWS Secrets Manager stores production secrets.
- GitHub Actions is the CI/CD platform.
- Production PostgreSQL backup retention is 35 days with point-in-time recovery enabled.
- Restore verification is monthly before launch stabilization, then quarterly after stable operations.

## Rationale

ECS Fargate keeps the deployment model container-native without making a solo operator manage servers or Kubernetes. Multi-AZ RDS in Production is the minimum acceptable reliability posture for real money, inventory, and audit data. GitHub Actions matches the already-decided GitHub-based workflow and avoids adding another delivery platform.

## Consequences

- The backend artifact is a Docker image promoted through environments.
- Network and secret handling are AWS-native.
- Production has higher baseline cost than Single-AZ/single-VM deployment.
- Horizontal scaling is available by increasing ECS task count once event dispatch is outbox-backed per ADR-0029.

## Future Reconsideration Conditions

Reconsider if AWS Fargate cost or platform limits become a measured constraint, or if team/traffic growth justifies EKS or service extraction.

## Related Documents

- Related architecture principle(s): Principle 4.1, Principle 4.2, Principle 4.12
- Related ADRs: 0002, 0003, 0004, 0016, 0024, 0028, 0029

