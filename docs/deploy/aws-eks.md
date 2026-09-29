---
title: AWS EKS
summary: Deploy Paperclip to AWS using EKS, RDS Postgres, and EFS
---

Deploy Paperclip to AWS with EKS (compute), RDS Postgres 17 (database), and EFS (persistent storage). This guide uses `eksctl`, the AWS CLI, and `kubectl`, and produces a single-replica Deployment behind an ALB with HTTPS. Use this path if you already run Kubernetes; for everyone else, the [ECS Fargate guide](aws-ecs.md) is simpler and cheaper.

## Prerequisites

- AWS CLI v2 configured with a profile that has admin-level permissions
- `eksctl`, `kubectl`, and `helm` installed locally
- Docker installed locally (for building and pushing the image)
- A registered domain with DNS you control (for the TLS certificate)
- The Paperclip repo cloned locally

Set these shell variables for the rest of the guide:

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME=paperclip
export PAPERCLIP_DOMAIN=paperclip.example.com   # your domain
export DB_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
export AUTH_SECRET=$(openssl rand -base64 32)
export MASTER_KEY=$(openssl rand -base64 32)
```

## 1. Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name paperclip-server \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION
```

## 2. Build and Push Docker Image

```bash
cd /path/to/paperclip

# Authenticate Docker to ECR
aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin \
    $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Build
docker build -t paperclip-server .

# Tag and push
docker tag paperclip-server:latest \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:latest

docker push \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:latest
```

## 3. Create EKS Cluster

`eksctl` creates a dedicated VPC with public and private subnets in two AZs, a managed node group in the private subnets, and the OIDC provider needed for IAM roles for service accounts (IRSA).

```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --version 1.33 \
  --managed \
  --nodegroup-name paperclip-nodes \
  --node-type t3.large \
  --nodes 2 --nodes-min 1 --nodes-max 3 \
  --node-private-networking \
  --with-oidc

# Takes 15-20 min. Confirm kubectl is pointed at the new cluster
kubectl get nodes
```

> **Note:** Agents run inside the Paperclip container, so size nodes for the agent workload, not just the server. `t3.large` (2 vCPU, 8 GB) fits one server pod with room to spare.

## 4. Networking (VPC, Subnets, Security Groups)

Look up the VPC that `eksctl` created and the cluster security group that the nodes (and therefore pods) use:

```bash
VPC_ID=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

CLUSTER_SG=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

# Get two private subnets (for RDS and EFS)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[?MapPublicIpOnLaunch==`false`] | [0:2].SubnetId' \
  --output text)
SUBNET_1=$(echo $SUBNET_IDS | awk '{print $1}')
SUBNET_2=$(echo $SUBNET_IDS | awk '{print $2}')
```

Create security groups. The ALB is created by the AWS Load Balancer Controller in step 8, which manages its own security groups, so only RDS and EFS need one here:

```bash
# RDS security group — inbound from the cluster only
RDS_SG=$(aws ec2 create-security-group \
  --group-name paperclip-rds \
  --description "Paperclip RDS" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp --port 5432 \
  --source-group $CLUSTER_SG

# EFS security group — inbound NFS from the cluster only
EFS_SG=$(aws ec2 create-security-group \
  --group-name paperclip-efs \
  --description "Paperclip EFS" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG \
  --protocol tcp --port 2049 \
  --source-group $CLUSTER_SG
```

## 5. Create RDS Postgres Instance

```bash
# The cluster VPC doesn't come with a DB subnet group — create one
# that spans our two private subnets so RDS can place the instance.
aws rds create-db-subnet-group \
  --db-subnet-group-name paperclip-db-subnet \
  --db-subnet-group-description "Paperclip RDS subnets" \
  --subnet-ids $SUBNET_1 $SUBNET_2

aws rds create-db-instance \
  --db-instance-identifier paperclip-db \
  --db-instance-class db.t4g.micro \
  --engine postgres \
  --engine-version 17 \
  --master-username paperclip \
  --master-user-password "$DB_PASSWORD" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --vpc-security-group-ids $RDS_SG \
  --db-subnet-group-name paperclip-db-subnet \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --no-multi-az \
  --db-name paperclip \
  --region $AWS_REGION

# Wait for it to become available (takes 5-10 min)
aws rds wait db-instance-available \
  --db-instance-identifier paperclip-db

# Get the endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier paperclip-db \
  --query 'DBInstances[0].Endpoint.Address' --output text)

DATABASE_URL="postgresql://paperclip:${DB_PASSWORD}@${RDS_ENDPOINT}:5432/paperclip"
```

## 6. Create EFS Filesystem

```bash
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=paperclip-data \
  --query 'FileSystemId' --output text)

# Create mount targets in each subnet
for SUBNET in $SUBNET_1 $SUBNET_2; do
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $SUBNET \
    --security-groups $EFS_SG
done

# Wait for mount targets
aws efs describe-mount-targets --file-system-id $EFS_ID
```

Install the EFS CSI driver as an EKS add-on, with an IAM role for its service account:

```bash
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --namespace kube-system \
  --name efs-csi-controller-sa \
  --role-name paperclip-efs-csi \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --role-only \
  --approve

eksctl create addon \
  --cluster $CLUSTER_NAME \
  --name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/paperclip-efs-csi
```

## 7. Namespace, Storage, and Secrets

```bash
kubectl create namespace paperclip
```

Create a StorageClass and PersistentVolumeClaim. The `efs-ap` provisioning mode creates an EFS access point that forces UID/GID 1000, matching the `node` user in the Paperclip image:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: paperclip-efs
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: $EFS_ID
  directoryPerms: "700"
  uid: "1000"
  gid: "1000"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: paperclip-data
  namespace: paperclip
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: paperclip-efs
  resources:
    requests:
      storage: 10Gi
EOF
```

Store secrets as a Kubernetes Secret:

```bash
kubectl create secret generic paperclip-secrets \
  --namespace paperclip \
  --from-literal=DATABASE_URL="$DATABASE_URL" \
  --from-literal=BETTER_AUTH_SECRET="$AUTH_SECRET" \
  --from-literal=PAPERCLIP_SECRETS_MASTER_KEY="$MASTER_KEY" \
  --from-literal=ANTHROPIC_API_KEY="YOUR_ANTHROPIC_KEY" \
  --from-literal=OPENAI_API_KEY="YOUR_OPENAI_KEY" \
  --from-literal=GITHUB_TOKEN="YOUR_GITHUB_PAT"
```

> **Warning:** Back up `$MASTER_KEY` outside the cluster (for example in a password manager or AWS Secrets Manager). Paperclip encrypts stored secrets with this key. If you restore the RDS database without the original key, every secret stored in Paperclip becomes unreadable. Supplying the key here, instead of letting Paperclip generate one on the EFS volume, keeps it separate from the data it protects.

> **Note:** Kubernetes Secrets are only base64-encoded. Enable [envelope encryption with a KMS key](https://docs.aws.amazon.com/eks/latest/userguide/enable-kms.html) on the cluster, or sync from AWS Secrets Manager with the External Secrets Operator, if that matters for your environment.

## 8. AWS Load Balancer Controller

The controller turns a Kubernetes Ingress into an ALB. Create its IAM policy and service account, then install it with Helm:

```bash
curl -o /tmp/alb-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name PaperclipALBControllerPolicy \
  --policy-document file:///tmp/alb-iam-policy.json

eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/PaperclipALBControllerPolicy \
  --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION \
  --set vpcId=$VPC_ID

kubectl rollout status deployment/aws-load-balancer-controller -n kube-system
```

> **Note:** The policy above tracks the controller's `main` branch. For production, pin both the policy and the Helm chart to the same released version.

## 9. TLS Certificate

Request a certificate (you must validate via DNS):

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name $PAPERCLIP_DOMAIN \
  --validation-method DNS \
  --query 'CertificateArn' --output text)

# Get the CNAME record to add to your DNS
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

Add the CNAME to your DNS provider, then wait for validation:

```bash
aws acm wait certificate-validated --certificate-arn $CERT_ARN
```

## 10. Deploy Paperclip

The Deployment uses the same environment as the ECS task definition. It runs one replica with the `Recreate` strategy: Paperclip is a single-instance control plane (one heartbeat scheduler, one local workspace), so the old pod must stop before the new one starts.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: paperclip-server
  namespace: paperclip
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: paperclip-server
  template:
    metadata:
      labels:
        app: paperclip-server
    spec:
      containers:
        - name: paperclip-server
          image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:latest
          ports:
            - containerPort: 3100
          env:
            - { name: NODE_ENV, value: "production" }
            - { name: HOST, value: "0.0.0.0" }
            - { name: PORT, value: "3100" }
            - { name: SERVE_UI, value: "true" }
            - { name: PAPERCLIP_HOME, value: "/paperclip" }
            - { name: PAPERCLIP_INSTANCE_ID, value: "default" }
            - { name: PAPERCLIP_CONFIG, value: "/paperclip/instances/default/config.json" }
            - { name: PAPERCLIP_DEPLOYMENT_MODE, value: "authenticated" }
            - { name: PAPERCLIP_DEPLOYMENT_EXPOSURE, value: "public" }
            - { name: PAPERCLIP_PUBLIC_URL, value: "https://$PAPERCLIP_DOMAIN" }
            - { name: PAPERCLIP_MIGRATION_AUTO_APPLY, value: "true" }
            - { name: HEARTBEAT_SCHEDULER_ENABLED, value: "true" }
          envFrom:
            - secretRef:
                name: paperclip-secrets
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "2"
              memory: 4Gi
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3100
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /api/health
              port: 3100
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: paperclip-data
              mountPath: /paperclip
      volumes:
        - name: paperclip-data
          persistentVolumeClaim:
            claimName: paperclip-data
---
apiVersion: v1
kind: Service
metadata:
  name: paperclip-server
  namespace: paperclip
spec:
  type: ClusterIP
  selector:
    app: paperclip-server
  ports:
    - port: 80
      targetPort: 3100
EOF
```

## 11. Ingress (ALB)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: paperclip-server
  namespace: paperclip
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
    alb.ingress.kubernetes.io/healthcheck-path: /api/health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
spec:
  ingressClassName: alb
  rules:
    - host: $PAPERCLIP_DOMAIN
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: paperclip-server
                port:
                  number: 80
EOF

# Wait for the ALB address (takes 2-3 min)
ALB_DNS=$(kubectl get ingress paperclip-server -n paperclip \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $ALB_DNS
```

Point your DNS to the ALB:
- Create a CNAME or ALIAS record for `$PAPERCLIP_DOMAIN` -> `$ALB_DNS`

## 12. Verify Deployment

```bash
# Watch the pod come up
kubectl get pods -n paperclip -w

# Check rollout and pod health
kubectl rollout status deployment/paperclip-server -n paperclip
kubectl describe pod -n paperclip -l app=paperclip-server

# Check logs
kubectl logs -n paperclip deployment/paperclip-server --since=10m -f

# Hit the health endpoint
curl -sf https://$PAPERCLIP_DOMAIN/api/health
```

**Healthy indicators:**
- Pod status: `Running`, `1/1` ready
- Logs show `plugin job coordinator started` and `plugin-loader: loadAll complete`
- `/api/health` returns 200

## Create the First Admin

A fresh public instance stays in `bootstrap_pending` until the first admin exists. In `authenticated` + `public` mode, the browser cannot claim admin. You must create a one-time bootstrap invite with the CLI and open it in your browser.

Run the setup wizard in a throwaway pod. It has no volume, so it does not touch the server's `/paperclip` data on EFS:

```bash
kubectl run paperclip-bootstrap -n paperclip --rm -it --restart=Never \
  --image=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:latest \
  --env="DATABASE_URL=$DATABASE_URL" \
  --env="PAPERCLIP_DEPLOYMENT_MODE=authenticated" \
  --env="PAPERCLIP_DEPLOYMENT_EXPOSURE=public" \
  --env="PAPERCLIP_PUBLIC_URL=https://$PAPERCLIP_DOMAIN" \
  -- npx --yes paperclipai onboard
```

Choose **Quickstart**. The wizard reads the environment above, writes a config inside the throwaway pod, and prints a bootstrap invite URL. Answer **No** when it asks to start Paperclip, so it does not run a second server against the same database.

> **Note:** `paperclipai auth bootstrap-ceo` alone does not work here. It needs a config file, and the Deployment is configured through environment variables only.

Open the invite URL, sign up, and accept the invite. That account becomes the first instance admin.

## Post-Deploy Security Hardening

After the first admin has accepted the bootstrap invite, lock down the instance:

```bash
# Disable public sign-up (prevents unauthorized users from creating accounts).
# Setting the env var triggers a new rollout.
kubectl set env deployment/paperclip-server -n paperclip \
  PAPERCLIP_AUTH_DISABLE_SIGN_UP=true
```

Use the invite flow (added in v2026.416.0) to grant access to additional users after sign-up is disabled.

## Deploying Updates

Build, push, and roll out a new image. Tag with a unique version rather than relying on `:latest`, so Kubernetes sees the change and rollbacks have something to return to:

```bash
VERSION=$(git rev-parse --short HEAD)

# Build and push new image
docker build -t paperclip-server .
docker tag paperclip-server:latest \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:$VERSION
docker push \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:$VERSION

# Roll out
kubectl set image deployment/paperclip-server -n paperclip \
  paperclip-server=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/paperclip-server:$VERSION

# Watch the deployment
kubectl rollout status deployment/paperclip-server -n paperclip
```

Because the strategy is `Recreate`, the old pod stops before the new one starts, so expect a short outage during each rollout. Database migrations apply automatically on startup (`PAPERCLIP_MIGRATION_AUTO_APPLY`).

## Rollback

If the new deployment is unhealthy:

```bash
# Kubernetes keeps the previous ReplicaSets. To roll back:

# 1. See the revision history
kubectl rollout history deployment/paperclip-server -n paperclip

# 2. Roll back to the previous revision (or add --to-revision=<N>)
kubectl rollout undo deployment/paperclip-server -n paperclip
```

Unlike ECS, there is no circuit breaker. With `Recreate`, a bad image means downtime until you run the rollback, so watch `kubectl rollout status` after every update.

## Scaling to Zero (Cost Savings)

Scale down when not in use:

```bash
# Stop the pod
kubectl scale deployment/paperclip-server -n paperclip --replicas=0

# Stop paying for nodes too (the EKS control plane keeps billing)
eksctl scale nodegroup \
  --cluster $CLUSTER_NAME \
  --name paperclip-nodes \
  --nodes 0 --nodes-min 0

# Start
eksctl scale nodegroup \
  --cluster $CLUSTER_NAME \
  --name paperclip-nodes \
  --nodes 2 --nodes-min 1
kubectl scale deployment/paperclip-server -n paperclip --replicas=1
```

RDS can also be stopped (auto-restarts after 7 days):

```bash
aws rds stop-db-instance --db-instance-identifier paperclip-db
aws rds start-db-instance --db-instance-identifier paperclip-db
```

## Teardown

Remove all resources in reverse order:

```bash
# 1. Kubernetes resources (deleting the Ingress removes the ALB)
kubectl delete namespace paperclip
kubectl delete storageclass paperclip-efs
helm uninstall aws-load-balancer-controller -n kube-system

# 2. RDS (creates final snapshot)
aws rds delete-db-instance \
  --db-instance-identifier paperclip-db \
  --final-db-snapshot-identifier paperclip-db-final
aws rds wait db-instance-deleted --db-instance-identifier paperclip-db
aws rds delete-db-subnet-group --db-subnet-group-name paperclip-db-subnet

# 3. EFS (mount targets must be deleted first)
for MT in $(aws efs describe-mount-targets --file-system-id $EFS_ID --query 'MountTargets[*].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id $MT
done
# Mount-target deletion is async; poll until none remain before deleting
# the filesystem, otherwise delete-file-system fails with FileSystemInUse.
echo "Waiting for mount targets to delete..."
while aws efs describe-mount-targets \
  --file-system-id $EFS_ID \
  --query 'MountTargets[0].MountTargetId' --output text 2>/dev/null | grep -q 'fsmt-'; do
  sleep 5
done
aws efs delete-file-system --file-system-id $EFS_ID

# 4. Security groups (after all dependents are gone)
for sg in $EFS_SG $RDS_SG; do
  aws ec2 delete-security-group --group-id $sg
done

# 5. EKS cluster (also deletes the VPC, node group, and IAM service accounts)
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION

# 6. ACM cert and IAM policy
aws acm delete-certificate --certificate-arn $CERT_ARN
aws iam delete-policy \
  --policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/PaperclipALBControllerPolicy

# 7. ECR
aws ecr delete-repository --repository-name paperclip-server --force
```

## Cost Reference

| Service | Config | Monthly |
|---------|--------|---------|
| EKS control plane | 1 cluster | ~$73 |
| EC2 nodes | 2x t3.large, 24/7 | ~$120 |
| RDS Postgres | db.t4g.micro, 20 GB | ~$15 |
| ALB | 1 LCU average | ~$22 |
| NAT Gateway | 1 AZ (created by `eksctl`) | ~$35 |
| EFS | 1 GB Standard | ~$0.30 |
| CloudWatch Logs | Control plane logs off by default | ~$0 |
| ECR | ~1 GB | ~$0.10 |
| **Total (2 nodes)** | | **~$265/mo** |
| **Total (1 node)** | | **~$205/mo** |

The EKS control plane and NAT Gateway bill even when the node group is scaled to zero. If you don't already run Kubernetes, ECS Fargate is roughly half the cost.
