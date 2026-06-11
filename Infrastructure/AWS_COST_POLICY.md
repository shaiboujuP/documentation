# AWS Cost Reduction Policy

Date: 2026-06-07

This document defines which AWS resources are safe to delete, which must be kept, and the architectural decisions made to reduce ongoing costs. Apply this policy before any infrastructure teardown and when auditing unused resources.

---

## Guiding Principle

The environment is a low-cost daily workspace. Runtime resources (compute, networking) are created at the start of the day and destroyed at the end. Persistent resources (storage, configuration, registries) survive across destroy cycles and are never deleted unless the corresponding service is permanently retired.

---

## Resource Classification

### Never Delete — Persistent Baseline

These resources cost near-zero or nothing, and destroying them breaks deployments or loses data.

| Resource | Name / ID | Why |
|---|---|---|
| VPC | `flokit-prod-vpc` (vpc-0d1528dd3c0f04516) | All k8s networking lives here. Deleting it tears down subnets, route tables, and security groups. |
| Subnets | `flokit-prod-private-1/2`, `flokit-prod-public-1/2` | Required for EC2 placement. Free resource. |
| Internet Gateway | `igw-0e21a5531e6a86a5f` | After the NAT removal (see below), this is the only outbound path for k8s nodes. |
| Route Tables | `flokit-prod-private`, `flokit-prod-public` | After 2026-06-07 the private route table points directly to the IGW. Do not add a NAT route back. |
| ECR Repositories | All `flokit-prod-*` repos | Contain production images. Deleting a repo loses all tags. |
| S3 Buckets | `flokit-prod-bitewise-photos`, `flokit-prod-dashboard-frontend`, `flokitai-terraform-state`, `web2app-manager-terraform-state` | Terraform state and app storage. Deleting state buckets breaks all future Terraform runs. |
| Secrets Manager secrets | All `flokit-prod/*` secrets | App runtime credentials. Free while unused; $0.40/month when accessed. Do not delete without first confirming the consuming service is retired. |
| IAM users, roles, policies | `flokit-terraform`, `flokit-root`, and associated roles | Deleting these breaks CI, Terraform, and ECR image pushes. |
| ACM certificates | All `*.flokitai.com` certs | Free. Cert-manager manages these; deleting them forces a Let's Encrypt re-issue and causes brief TLS downtime. |
| Route 53 / Cloudflare DNS records | All `flokitai.com` backend records | Free. Managed by Terraform. |
| MongoDB Atlas | `Prod` cluster | Persistent database layer. Lives outside AWS. Never touch from the AWS console. |

---

### Delete Daily — Runtime Resources

These are created by `scripts/deploy.sh` and destroyed by `scripts/destroy.sh`. They account for the majority of AWS spend.

| Resource | Approx. Cost If Always On | Notes |
|---|---|---|
| EC2 instances (`flokit-prod-k8s-control-plane`, `flokit-prod-k8s-worker-1`) | $24.53/month (before free-trial credit) | `t4g.small` ARM/Graviton. Free trial eligible through December 2026. |
| Elastic IPs on EC2 instances | ~$0 while associated to running instances; $3.65/month each if left unattached | Assigned to the k8s nodes since 2026-06-07. Release them before or immediately after instance termination. |
| NLB (`flokit-prod-k8s-nlb`) | $20.81/month | No traffic = no NLCU charge, but the hourly fee (~$0.008/hr) still runs. Destroy it at end of day. |
| EBS root volumes | $4.80/month for 60 GB | Destroyed automatically with the instances. |

**Important:** The NAT Gateway has been permanently removed (see section below). Do not recreate it as a runtime resource. The daily cost saving is ~$35/month.

---

### Safe to Delete — Orphaned / Unused Resources

These have no active dependency and can be removed at any time.

| Resource | Why It Is Safe |
|---|---|
| `web2app-manager` ECR repo (0 images) | Empty repo, superseded by `flokit-prod-web2app-manager`. Deleted on 2026-06-07. |
| `web2app-manager-production` VPC (vpc-04eaaaa5a8c6640f3) | Old EKS VPC with no instances, no load balancers, and no NAT. Deleted on 2026-06-07. |
| Old `web2app-manager` IAM roles | `web2app-manager-eks-node`, `web2app-manager-pod`, `web2app-manager-production-cluster-*`, and the EKS OIDC provider. No live EKS cluster references them. Delete after confirming no Terraform state references them. |
| ECR images older than the last 3 tags per repo | Old images cost storage. Apply lifecycle policies (see below). |
| Unattached Elastic IPs | Charged at $3.65/month each even if not used. Run `aws ec2 describe-addresses` and release any with no `AssociationId`. |
| Stopped EC2 instances | A stopped instance still holds its EBS volume and any attached EIP. Terminate rather than stop unless a specific restart is planned within the same day. |

---

### Do Not Create — Avoid These Resources

| Resource | Reason |
|---|---|
| NAT Gateway | Removed on 2026-06-07. Costs ~$35/month fixed even with no traffic. k8s nodes now use public EIPs on the IGW path. Do not add a NAT route back to the private route table. |
| EKS cluster | Charges $0.10/hour ($73/month) for the managed control plane regardless of usage. Use k3s. |
| RDS / DocumentDB | Not part of the stack. All databases use MongoDB Atlas. |
| ElastiCache (managed) | Not provisioned. In-cluster Redis on k3s covers current needs. |
| Multi-AZ NAT Gateways | One NAT per AZ is an anti-pattern for this environment. Never create more than one NAT Gateway, and ideally zero (see NAT decision below). |
| Additional EIPs left unattached | Each unattached EIP costs $3.65/month. Allocate only when immediately associating to a running instance. |

---

## NAT Gateway Removal — 2026-06-07

**Before:** Both k8s nodes were in private subnets routing outbound traffic through a NAT Gateway (`nat-07bfeb32e4ff1b312`, EIP `44.196.213.243`). Monthly cost: ~$35.

**After:** The NAT Gateway and its EIP were deleted. Each k8s node was assigned a dedicated Elastic IP:

| Instance | EIP |
|---|---|
| `flokit-prod-k8s-control-plane` (i-0cfb9dcb26be62f08) | `34.229.27.174` |
| `flokit-prod-k8s-worker-1` (i-056aef04d4cbd870a) | `184.72.181.81` |

The private route table (`rtb-026e08b8e28fc2234`) was updated: `0.0.0.0/0` now routes to the Internet Gateway (`igw-0e21a5531e6a86a5f`) instead of the NAT Gateway.

**Atlas Network Access:** The old NAT EIP (`44.196.213.243`) was the stable egress IP registered in MongoDB Atlas. After this change, both EC2 EIPs must be added to Atlas Network Access for the `Prod` project. Verify Atlas allows `34.229.27.174` and `184.72.181.81` before running database-backed services.

**Security note:** The nodes are now directly internet-reachable. Security groups must allow inbound traffic only on the required ports (NodePorts 30080/30443 from the NLB, SSH/SSM management only). No broad `0.0.0.0/0` inbound rules.

---

## ECR Image Lifecycle Policy

Apply the following lifecycle policy to all `flokit-prod-*` ECR repositories to prevent unbounded storage growth. The `flokit-prod-agentic-pmf-test` repo already has 37 images.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only the last 5 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": [""],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged images after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    }
  ]
}
```

---

## Monthly Cost Reference (Post-NAT-Removal, Always-On)

| Item | Estimate/month |
|---|---|
| EC2: 2 × `t4g.small` (before free-trial credit) | $24.53 |
| NLB hourly + 1 NLCU | $20.81 |
| 2 × EC2 EIPs (attached, running) | $0 |
| EBS gp3, 60 GB | $4.80 |
| Secrets Manager (13 secrets) | $5.20 |
| ECR / S3 / CloudWatch low-volume | $2–5 |
| **Estimated always-on total (before Atlas)** | **~$57–63/month** |

With daily destroy (8 h/day, 22 workdays): runtime cost drops to ~24% of always-on, so approximately **~$14–15/month** in runtime charges plus persistent baseline.

Previous always-on estimate was ~$90–98/month. The NAT Gateway removal saves approximately **$35/month**.
