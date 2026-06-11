# AWS Cost Reduction Policy

Date: 2026-06-07 (last updated: 2026-06-11)

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
| MongoDB Atlas | `Prod` cluster | Persistent database layer (free-tier M0). Lives outside AWS. Network Access is open to `0.0.0.0/0` (see 2026-06-11 update) — no IP allow-list to maintain. Never touch from the AWS console. |

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
| `web2app-manager-production` VPC | Old EKS VPC, no instances or load balancers. The CIDR-`10.0.0.0/16` instance (vpc-006da044559a9461d) still held a live NAT gateway (`nat-008f993e827cbdb83`, ~$35/month) and was **fully deleted on 2026-06-11** — NAT, EIP, 6 subnets, IGW, route tables, and security groups. (An earlier EKS VPC vpc-04eaaaa5a8c6640f3 was removed 2026-06-07.) |
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

**After:** The NAT Gateway and its EIP were deleted. Each k8s node egresses through the IGW via a dedicated Elastic IP.

The private route table (`rtb-026e08b8e28fc2234`) was updated: `0.0.0.0/0` now routes to the Internet Gateway (`igw-0e21a5531e6a86a5f`) instead of the NAT Gateway.

> **Note (2026-06-11):** The specific node EIPs originally recorded here (`34.229.27.174`, `184.72.181.81`) were transient. The Terraform reconciliation of the NAT removal recreated the node EIPs, and egress now uses `34.195.115.249` (control-plane) and `3.229.31.254` (worker). Because Atlas is now open to `0.0.0.0/0`, these IPs are no longer load-bearing — do not treat them as stable. See the 2026-06-11 update below.

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

---

## Update — 2026-06-11

Follow-up work after the NAT removal landed, reconciling Terraform with reality and cleaning up leftover cost.

**Terraform reconciliation of the NAT removal.** Until now `modules/network` still defined the NAT gateway, so a `terraform apply` would have recreated it. The Terraform was corrected to match reality: the `aws_nat_gateway`/`aws_eip.nat` resources were removed, the private route table now targets the IGW, and per-node EIPs (`aws_eip.control_plane` / `aws_eip.worker[*]`) plus associations were added in `modules/k8s-cluster`. Because the CI apply ran before the existing node EIPs were imported, Terraform minted **new** EIPs — current egress is `34.195.115.249` (control-plane) and `3.229.31.254` (worker).

**MongoDB Atlas → `0.0.0.0/0`.** The `Prod` cluster is free-tier (M0), which does not support VPC peering or PrivateLink. Rather than maintain an IP allow-list against churning node EIPs, Atlas Network Access was opened to `0.0.0.0/0`. Security now rests on credentials + TLS only; this is acceptable pre-real-clients but **should be revisited** (peering/PrivateLink on an M10+, or a tight allow-list) before onboarding real customer data.

**EIP `prevent_destroy` removed.** Since egress-IP stability no longer matters, the `prevent_destroy` guards on the node EIPs were dropped — allocations may now be recycled freely on teardown.

**Cleanup — resources deleted (~$46/month recovered):**

| Resource | ID | Monthly cost |
|---|---|---|
| Stray NAT gateway (web2app VPC) | `nat-008f993e827cbdb83` | ~$35 |
| NAT's EIP | `eipalloc-0976d8efd6631452c` (`52.202.184.46`) | ~$3.65 |
| Orphaned node EIP | `eipalloc-0eee494a4cb9d3953` (`34.229.27.174`) | ~$3.65 |
| Orphaned node EIP | `eipalloc-062edbbd5b53392cf` (`184.72.181.81`) | ~$3.65 |
| `web2app-manager-production` VPC (`vpc-006da044559a9461d`) + subnets/IGW/route tables/SGs | — | $0 (hygiene) |

After cleanup, the only Elastic IPs in the account are the two attached to the live k8s nodes. No NAT gateways exist in the account.
