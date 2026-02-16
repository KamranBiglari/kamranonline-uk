---
title: "Automating AWS ECR Credential Refresh in Kubernetes with Terraform"
meta_title: "Auto-Refresh ECR Credentials in Kubernetes - Terraform Solution"
description: "Learn how to automatically refresh AWS ECR credentials in Kubernetes every 6 hours using Terraform. A production-ready solution that works with any Kubernetes distribution, not just EKS."
date: 2026-02-16T12:00:00Z
image: "/images/blog/aws-ecr-credential-refresh/ecr-credential-refresh-kubernetes.png"
categories: ["Kubernetes", "AWS", "Terraform"]
author: "Kamran Biglari"
tags: ["kubernetes", "aws", "ecr", "terraform", "docker", "cronjob", "automation"]
draft: false
---

If you're running a Kubernetes cluster and pulling Docker images from AWS Elastic Container Registry (ECR), you've likely encountered this frustrating problem: **ECR authentication tokens expire every 12 hours**.

This means your pods can't pull images after the token expires, leading to failed deployments and restarts. While this is a security feature by design, it creates an operational headache for teams running production Kubernetes clusters.

## The Problem: ECR Tokens That Expire Every 12 Hours

When you authenticate with ECR using `aws ecr get-login-password`, AWS returns a temporary token valid for exactly 12 hours. After expiration:

- New pod deployments fail with `ImagePullBackOff` errors
- Pod restarts can't pull updated images
- Scaling operations fail silently
- Your on-call engineer gets paged at 3 AM

**Error message you'll see:**

```
Failed to pull image "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest":
rpc error: code = Unknown desc = Error response from daemon:
Get https://123456789012.dkr.ecr.us-east-1.amazonaws.com/v2/:
no basic auth credentials
```

## Common Solutions and Their Limitations

Several approaches exist to solve this problem:

### 1. Manual Token Refresh ❌
```bash
# Run this every 12 hours... manually
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
```

**Problem:** Not practical for production environments. Requires human intervention and doesn't scale.

### 2. AWS EKS with IRSA ✅
If you're running on Amazon EKS, you can use [IAM Roles for Service Accounts (IRSA)](/blog/terraform-aws-kubernetes-irsa) for seamless ECR access.

**Problem:** Only works on EKS. What about self-hosted clusters on Hetzner, DigitalOcean, GKE, or bare metal?

### 3. Third-Party Tools ⚠️
Tools like `aws-ecr-credential-helper` or `k8s-ecr-login-renew`.

**Problem:** Adds another dependency to manage, maintain, and potentially troubleshoot.

### 4. Custom Kubernetes Operators 🔧
Build a full-fledged operator with controller-runtime.

**Problem:** Overkill for a relatively simple problem. Adds complexity and maintenance burden.

For my self-hosted Kubernetes cluster (running on Hetzner Cloud with Talos Linux), I needed a **simple, reliable solution that would work outside of AWS infrastructure**.

## The Solution: A Kubernetes CronJob with Terraform

I built an automated ECR credential refresh system using Terraform that:

✅ **Runs every 6 hours** (well before the 12-hour expiration)
✅ **Updates credentials across all namespaces** automatically
✅ **Requires minimal resources** (50m CPU, 64Mi RAM)
✅ **Is fully declarative** and version-controlled with Terraform
✅ **Works with any Kubernetes cluster** (not just EKS)

**Terraform Module:** [registry.terraform.io/modules/KamranBiglari/ecr-k8s-credentials/aws](https://registry.terraform.io/modules/KamranBiglari/ecr-k8s-credentials/aws/latest)

**GitHub Repository:** [github.com/KamranBiglari/terraform-aws-ecr-k8s-credentials](https://github.com/KamranBiglari/terraform-aws-ecr-k8s-credentials)

## Architecture Overview

The solution consists of several components working together:

1. **IAM User** — Dedicated AWS user for ECR read-only access
2. **Kubernetes Namespace** — Isolated namespace for the credential updater
3. **Service Account + RBAC** — Cluster-wide permissions to update secrets
4. **CronJob** — Scheduled task that refreshes credentials every 6 hours
5. **AWS Credentials Secret** — Securely stored AWS access keys

**Execution Flow:**

```
┌─────────────────────────────────┐
│   CronJob (Every 6 hours)       │
│   Container: alpine/k8s         │
└──────────────┬──────────────────┘
               │
               ├─> 1. Fetch ECR token from AWS
               │      (aws ecr get-login-password)
               │
               ├─> 2. Discover all namespaces
               │      (kubectl get namespaces)
               │
               └─> 3. Create/Update docker-registry secret
                      in each namespace
                      (kubectl apply -f -)
```

## Implementation Details

### Step 1: IAM User and Policy

First, create a dedicated IAM user with **read-only** ECR permissions:

```hcl
resource "aws_iam_user" "ecr_k8s_user" {
  name = "${var.APP_NAME}-ecr-k8s-user"
  path = "/system/"

  tags = {
    Purpose = "ECR credential refresh for Kubernetes"
  }
}

resource "aws_iam_user_policy" "ecr_k8s_policy" {
  name = "${var.APP_NAME}-ecr-readonly"
  user = aws_iam_user.ecr_k8s_user.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:DescribeRepositories",
          "ecr:DescribeImages",
          "ecr:ListImages"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_access_key" "ecr_k8s_key" {
  user = aws_iam_user.ecr_k8s_user.name
}
```

**Security Note:** These are **read-only** permissions. The IAM user can pull images but **cannot push or modify** your ECR repositories.

### Step 2: Kubernetes Namespace and Secrets

Create a dedicated namespace to isolate the credential updater:

```hcl
resource "kubernetes_namespace" "ecr_updater" {
  metadata {
    name = "ecr-updater"
    labels = {
      name = "ecr-updater"
      app  = "credential-management"
    }
  }
}

resource "kubernetes_secret" "aws_credentials" {
  metadata {
    name      = "aws-ecr-credentials"
    namespace = kubernetes_namespace.ecr_updater.metadata[0].name
  }

  data = {
    AWS_ACCESS_KEY_ID     = aws_iam_access_key.ecr_k8s_key.id
    AWS_SECRET_ACCESS_KEY = aws_iam_access_key.ecr_k8s_key.secret
    AWS_REGION            = var.AWS_REGION
    AWS_ACCOUNT_ID        = data.aws_caller_identity.current.account_id
  }

  type = "Opaque"
}
```

### Step 3: RBAC Configuration

The CronJob needs **cluster-wide permissions** to update secrets in all namespaces:

```hcl
resource "kubernetes_service_account" "ecr_updater" {
  metadata {
    name      = "ecr-credential-updater"
    namespace = kubernetes_namespace.ecr_updater.metadata[0].name
  }
}

resource "kubernetes_cluster_role" "ecr_updater" {
  metadata {
    name = "ecr-credential-updater"
  }

  rule {
    api_groups = [""]
    resources  = ["secrets"]
    verbs      = ["get", "create", "patch", "update"]
  }

  rule {
    api_groups = [""]
    resources  = ["namespaces"]
    verbs      = ["get", "list"]
  }
}

resource "kubernetes_cluster_role_binding" "ecr_updater" {
  metadata {
    name = "ecr-credential-updater"
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = kubernetes_cluster_role.ecr_updater.metadata[0].name
  }

  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account.ecr_updater.metadata[0].name
    namespace = kubernetes_namespace.ecr_updater.metadata[0].name
  }
}
```

**Why ClusterRole?** The CronJob needs to:
- List all namespaces in the cluster
- Create/update secrets in any namespace (including future namespaces)

### Step 4: The CronJob Magic

The heart of the solution is a CronJob that runs every 6 hours:

```hcl
resource "kubernetes_cron_job_v1" "ecr_credential_refresh" {
  metadata {
    name      = "ecr-credential-refresh"
    namespace = kubernetes_namespace.ecr_updater.metadata[0].name
  }

  spec {
    schedule                      = "0 */6 * * *" # Every 6 hours
    successful_jobs_history_limit = 3
    failed_jobs_history_limit     = 3

    job_template {
      metadata {
        name = "ecr-credential-refresh"
      }

      spec {
        template {
          metadata {
            labels = {
              app = "ecr-credential-refresh"
            }
          }

          spec {
            service_account_name = kubernetes_service_account.ecr_updater.metadata[0].name
            restart_policy       = "OnFailure"

            container {
              name  = "ecr-credential-updater"
              image = "alpine/k8s:1.30.7"

              command = ["/bin/sh", "-c"]
              args = [
                <<-EOT
                #!/bin/sh
                set -e

                # Install AWS CLI
                echo "Installing AWS CLI..."
                apk add --no-cache aws-cli

                echo "Fetching ECR authorization token..."
                TOKEN=$(aws ecr get-login-password --region $AWS_REGION)

                echo "Creating Docker config JSON..."
                DOCKER_CONFIG=$(echo -n "{\"auths\":{\"$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com\":{\"username\":\"AWS\",\"password\":\"$TOKEN\"}}}" | base64 -w 0)

                # Get all namespaces and update secrets in each
                echo "Discovering all namespaces..."
                NAMESPACES=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}')

                echo "Found namespaces: $NAMESPACES"

                for NAMESPACE in $NAMESPACES; do
                  echo "Updating secret in namespace: $NAMESPACE"

                  # Create or update the secret
                  kubectl create secret docker-registry ecr-registry-credentials \
                    --docker-server=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com \
                    --docker-username=AWS \
                    --docker-password=$TOKEN \
                    --namespace=$NAMESPACE \
                    --dry-run=client -o yaml | kubectl apply -f -

                  echo "Secret updated successfully in $NAMESPACE"
                done

                echo "ECR credentials refresh completed successfully!"
                EOT
              ]

              env {
                name = "AWS_ACCESS_KEY_ID"
                value_from {
                  secret_key_ref {
                    name = kubernetes_secret.aws_credentials.metadata[0].name
                    key  = "AWS_ACCESS_KEY_ID"
                  }
                }
              }

              env {
                name = "AWS_SECRET_ACCESS_KEY"
                value_from {
                  secret_key_ref {
                    name = kubernetes_secret.aws_credentials.metadata[0].name
                    key  = "AWS_SECRET_ACCESS_KEY"
                  }
                }
              }

              env {
                name = "AWS_REGION"
                value_from {
                  secret_key_ref {
                    name = kubernetes_secret.aws_credentials.metadata[0].name
                    key  = "AWS_REGION"
                  }
                }
              }

              env {
                name = "AWS_ACCOUNT_ID"
                value_from {
                  secret_key_ref {
                    name = kubernetes_secret.aws_credentials.metadata[0].name
                    key  = "AWS_ACCOUNT_ID"
                  }
                }
              }

              resources {
                limits = {
                  cpu    = "100m"
                  memory = "128Mi"
                }
                requests = {
                  cpu    = "50m"
                  memory = "64Mi"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### How the Script Works

The shell script in the CronJob performs these steps:

1. **Installs AWS CLI** in the Alpine container (`apk add aws-cli`)
2. **Fetches a fresh ECR token** using AWS credentials (`aws ecr get-login-password`)
3. **Discovers all namespaces** in the cluster (`kubectl get namespaces`)
4. **Creates or updates** a `docker-registry` secret named `ecr-registry-credentials` in each namespace
5. **Logs progress** for debugging and monitoring

**Key Insight:** Using `kubectl apply` with `--dry-run=client -o yaml` allows us to create the secret if it doesn't exist **or** update it if it does — all in one idempotent command.

## Using the Credentials in Your Deployments

Once deployed, reference the secret in your pod specifications:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      imagePullSecrets:
        - name: ecr-registry-credentials  # Automatically created/updated every 6 hours
      containers:
        - name: my-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
          ports:
            - containerPort: 8080
```

**That's it!** Your pods will now pull images from ECR using automatically refreshed credentials.

## Benefits of This Approach

### 1. Infrastructure as Code

Everything is defined in Terraform, making it reproducible and version-controlled.

```bash
# Deploy to dev cluster
terraform apply -var-file=dev.tfvars

# Deploy to prod cluster
terraform apply -var-file=prod.tfvars
```

### 2. Works Everywhere

Not tied to EKS or any specific cloud provider. Works with self-hosted clusters on:
- Hetzner Cloud
- DigitalOcean
- Google Kubernetes Engine (GKE)
- Azure Kubernetes Service (AKS)
- Bare metal
- Raspberry Pi clusters

### 3. Minimal Resources

The CronJob uses only **50m CPU** and **64Mi RAM** — negligible overhead.

**Cost:** Essentially free. Runs for ~5 seconds every 6 hours.

### 4. Set It and Forget It

Once deployed, it runs automatically every 6 hours. No manual intervention needed.

### 5. Multi-Namespace Support

Automatically discovers and updates credentials in **all namespaces**, including new ones created after deployment.

### 6. Simple Debugging

Logs are straightforward to read. Check job history with:

```bash
kubectl get jobs -n ecr-updater
kubectl logs -n ecr-updater job/ecr-credential-refresh-xxxxx
```

## Monitoring and Troubleshooting

### Check CronJob Status

```bash
kubectl get cronjob -n ecr-updater

# Output:
# NAME                      SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# ecr-credential-refresh    0 */6 * * *   False     0        3h ago          30d
```

### View Job History

```bash
kubectl get jobs -n ecr-updater

# Output:
# NAME                              COMPLETIONS   DURATION   AGE
# ecr-credential-refresh-28474850   1/1           12s        6h
# ecr-credential-refresh-28474844   1/1           11s        12h
# ecr-credential-refresh-28474838   1/1           13s        18h
```

### Check Logs

```bash
# Get the latest job
kubectl get jobs -n ecr-updater --sort-by=.metadata.creationTimestamp

# View logs
kubectl logs -n ecr-updater job/ecr-credential-refresh-28474850
```

**Example successful log output:**

```
Installing AWS CLI...
Fetching ECR authorization token...
Creating Docker config JSON...
Discovering all namespaces...
Found namespaces: default kube-system production staging ecr-updater
Updating secret in namespace: default
secret/ecr-registry-credentials created
Updating secret in namespace: kube-system
secret/ecr-registry-credentials created
Updating secret in namespace: production
secret/ecr-registry-credentials configured
Updating secret in namespace: staging
secret/ecr-registry-credentials configured
Updating secret in namespace: ecr-updater
secret/ecr-registry-credentials created
ECR credentials refresh completed successfully!
```

### Verify Secrets Exist

```bash
# Check if secrets exist in all namespaces
kubectl get secrets --all-namespaces | grep ecr-registry-credentials

# Check a specific namespace
kubectl get secret ecr-registry-credentials -n production -o yaml
```

### Manual Trigger (for testing)

```bash
# Trigger the CronJob manually
kubectl create job -n ecr-updater manual-refresh --from=cronjob/ecr-credential-refresh

# Watch it run
kubectl logs -n ecr-updater job/manual-refresh -f
```

### Common Issues and Solutions

**Problem: "Error from server (Forbidden): secrets is forbidden"**

**Cause:** ServiceAccount doesn't have proper RBAC permissions.

**Solution:** Verify ClusterRoleBinding exists:
```bash
kubectl get clusterrolebinding ecr-credential-updater
```

**Problem: "Unable to connect to the server: x509: certificate signed by unknown authority"**

**Cause:** Container can't verify Kubernetes API server certificate (common in self-signed cert clusters).

**Solution:** Add `insecure-skip-tls-verify` flag to kubectl commands (not recommended for production) or mount the CA certificate.

**Problem: "An error occurred (AccessDeniedException) when calling GetAuthorizationToken"**

**Cause:** AWS credentials are invalid or expired.

**Solution:** Verify IAM user credentials:
```bash
kubectl get secret aws-ecr-credentials -n ecr-updater -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d
```

## Security Considerations

### 1. IAM Permissions: Read-Only Access

The IAM user has **read-only** access to ECR. It cannot:
- Push images
- Delete repositories
- Modify repository policies
- Change lifecycle policies

### 2. Kubernetes RBAC: Minimal Privileges

The ServiceAccount can **only**:
- Manage secrets (get, create, patch, update)
- List namespaces

It has **no other cluster permissions** (can't modify deployments, pods, configmaps, etc.).

### 3. Secret Management

AWS credentials are stored as Kubernetes secrets. For enhanced security, consider:

**Option A: External Secrets Operator**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
```

**Option B: Sealed Secrets**
```bash
kubeseal < aws-credentials.yaml > sealed-aws-credentials.yaml
```

**Option C: Vault**
```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "ecr-updater"
```

### 4. Network Policies

Restrict the CronJob's network access to only AWS ECR endpoints:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ecr-updater-netpol
  namespace: ecr-updater
spec:
  podSelector:
    matchLabels:
      app: ecr-credential-refresh
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Allow Kubernetes API
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              component: kube-apiserver
      ports:
        - protocol: TCP
          port: 6443
    # Allow AWS ECR endpoints
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

### 5. Audit Logging

Enable Kubernetes audit logging to track secret modifications:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
    namespaces: ["production", "staging"]
```

## Customization Options

### Change the Schedule

Modify the cron schedule to run more or less frequently:

```hcl
# Every 4 hours (more frequent)
schedule = "0 */4 * * *"

# Every 8 hours (less frequent)
schedule = "0 */8 * * *"

# Every 12 hours (risky - token expires every 12h)
schedule = "0 */12 * * *"  # Not recommended!

# At specific times
schedule = "0 2,8,14,20 * * *"  # At 2 AM, 8 AM, 2 PM, 8 PM
```

**Recommendation:** Keep it at 6 hours or less to have a safety buffer before the 12-hour expiration.

### Target Specific Namespaces

Modify the shell script to only update specific namespaces:

```bash
# Instead of discovering all namespaces
NAMESPACES="production staging development"

for NAMESPACE in $NAMESPACES; do
  echo "Updating secret in namespace: $NAMESPACE"
  # ... rest of the script
done
```

### Use Different Secret Name

Change `ecr-registry-credentials` to something else:

```bash
kubectl create secret docker-registry my-custom-secret-name \
  --docker-server=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$TOKEN \
  --namespace=$NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

Then reference it in your deployments:

```yaml
imagePullSecrets:
  - name: my-custom-secret-name
```

### Support Multiple AWS Accounts

If you have images in multiple AWS accounts:

```bash
# In the CronJob script
ACCOUNTS="123456789012 987654321098"

for ACCOUNT in $ACCOUNTS; do
  echo "Fetching token for account: $ACCOUNT"

  # Assume role in target account
  ASSUMED_ROLE=$(aws sts assume-role \
    --role-arn "arn:aws:iam::$ACCOUNT:role/ECRAccessRole" \
    --role-session-name ecr-refresh)

  # Extract credentials and fetch token
  # ... create secrets for each account
done
```

## Alternative: Initial Run Job

If you want to populate secrets **immediately** upon deployment (not waiting for the first CronJob run), add this Kubernetes Job:

```hcl
resource "kubernetes_job_v1" "ecr_credential_initial" {
  metadata {
    name      = "ecr-credential-initial"
    namespace = kubernetes_namespace.ecr_updater.metadata[0].name
  }

  spec {
    template {
      metadata {
        labels = {
          app = "ecr-credential-initial"
        }
      }
      spec {
        service_account_name = kubernetes_service_account.ecr_updater.metadata[0].name
        restart_policy       = "Never"

        container {
          name    = "ecr-credential-updater"
          image   = "alpine/k8s:1.30.7"
          command = ["/bin/sh", "-c"]
          args    = [
            # ... same script as CronJob
          ]

          # ... same env vars and resources
        }
      }
    }
  }

  wait_for_completion = true

  timeouts {
    create = "5m"
    update = "5m"
  }
}
```

This ensures secrets are available immediately after `terraform apply`.

## Performance and Resource Usage

### CPU and Memory

**Observed resource usage during execution:**

```
Metric              | Request | Limit  | Actual Usage
--------------------|---------|--------|-------------
CPU                 | 50m     | 100m   | ~30m
Memory              | 64Mi    | 128Mi  | ~45Mi
Execution Time      | -       | -      | 8-12 seconds
```

**Cost impact:** Essentially zero. Running for 10 seconds every 6 hours = 40 seconds per day = 0.046% CPU utilization.

### Network Traffic

**Per execution:**
- AWS API calls: ~5 KB (get-login-password)
- Kubernetes API calls: ~2 KB per namespace
- Total: ~20 KB for a cluster with 5 namespaces

**Monthly bandwidth:** ~3 MB (120 executions × 20 KB)

## Comparison with Alternative Solutions

| Feature | This Solution | aws-ecr-credential-helper | EKS IRSA | Custom Operator |
|---------|--------------|---------------------------|----------|-----------------|
| **Infrastructure** | CronJob | DaemonSet | EKS-only | Custom Deployment |
| **Setup Complexity** | Low | Medium | Low (EKS) | High |
| **Maintenance** | Minimal | Medium | Minimal | High |
| **Non-EKS Support** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| **Resource Usage** | Very Low | Medium | Low | Medium |
| **Dependencies** | Alpine + kubectl + awscli | Go binary | EKS | Custom code |
| **Multi-namespace** | ✅ Automatic | ⚠️ Manual config | ✅ Automatic | ✅ Automatic |
| **Debuggability** | ✅ Easy (logs) | ⚠️ Complex | ✅ Easy | ⚠️ Complex |
| **IaC Support** | ✅ Terraform native | ⚠️ Helm/manual | ✅ Terraform | ⚠️ Custom |

## Real-World Production Usage

This solution has been running in production on my Hetzner Cloud Kubernetes cluster (Talos Linux) for several months with:

- **5 namespaces** (production, staging, development, monitoring, logging)
- **30+ deployments** pulling from ECR
- **Zero downtime** related to credential expiration
- **Zero manual interventions** required
- **< 0.1%** cluster resource usage

**Reliability stats:**
- Success rate: 99.9%+
- Failed executions: 1 (temporary AWS API throttling, auto-recovered on next run)
- Manual interventions: 0

## Conclusion

This Terraform-based ECR credential refresh solution has been running in production without issues. It's simple, reliable, and works across any Kubernetes distribution.

### Key Advantages

✅ **Fully declarative** infrastructure as code
✅ **No vendor lock-in** (works outside EKS)
✅ **Minimal resource footprint** (~50m CPU, ~64Mi RAM)
✅ **Automatic multi-namespace support**
✅ **Easy to customize and debug**
✅ **Production-tested** reliability

### When to Use This Solution

**✅ Perfect for:**
- Self-hosted Kubernetes clusters (non-EKS)
- Hybrid cloud setups (Kubernetes on-prem, ECR in AWS)
- Multi-cloud architectures (GKE + ECR, AKS + ECR)
- Cost-conscious teams (avoiding EKS fees)
- GitOps workflows (everything in Terraform)

**⚠️ Consider alternatives if:**
- You're already running on EKS → Use [IRSA](/blog/terraform-aws-kubernetes-irsa) instead
- You have regulatory requirements against storing AWS credentials in-cluster → Use [external secrets](https://external-secrets.io/)
- You need sub-6-hour refresh cycles → Adjust the CronJob schedule

### Getting Started

**Quick Start (3 commands):**

```bash
# 1. Clone or copy the Terraform code
git clone https://github.com/KamranBiglari/terraform-aws-ecr-k8s-credentials

# 2. Configure variables
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your AWS region and app name

# 3. Deploy
terraform init
terraform apply
```

**Required variables:**
- `var.APP_NAME` — Your application prefix (e.g., "myapp")
- `var.AWS_REGION` — Your AWS region (e.g., "us-east-1")
- AWS credentials configured (via AWS CLI or environment variables)

**Complete Terraform Module:** [registry.terraform.io/modules/KamranBiglari/ecr-k8s-credentials/aws](https://registry.terraform.io/modules/KamranBiglari/ecr-k8s-credentials/aws/latest)

**Source Code:** [github.com/KamranBiglari/terraform-aws-ecr-k8s-credentials](https://github.com/KamranBiglari/terraform-aws-ecr-k8s-credentials)

---

If you're running a self-hosted Kubernetes cluster and pulling from ECR, this approach can save you from expired credential headaches while maintaining security and simplicity.

Have you solved this problem differently? Found ways to improve this solution? Reach out on [LinkedIn](https://linkedin.com/in/kamran-biglari) or open an issue on [GitHub](https://github.com/KamranBiglari/terraform-aws-ecr-k8s-credentials)!
