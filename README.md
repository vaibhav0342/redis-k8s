# AWS ElastiCache Redis with EKS and Kubernetes ExternalName

This guide explains how to create an **AWS ElastiCache Redis** cluster and connect applications running in an **Amazon EKS** cluster using a Kubernetes `ExternalName` Service.

## Architecture

```text
                    AWS VPC
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Amazon EKS                                        │
│  ┌───────────────────────────────────────────────┐  │
│  │                                               │  │
│  │ Application Pod                               │  │
│  │                                               │  │
│  │ redis:6379                                    │  │
│  │    │                                          │  │
│  │    ▼                                          │  │
│  │ Kubernetes ExternalName Service              │  │
│  │    │                                          │  │
│  └────┼──────────────────────────────────────────┘  │
│       │                                             │
│       ▼                                             │
│  AWS ElastiCache Redis                              │
│  Private Endpoint                                   │
│       :6379                                         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## Prerequisites

Make sure you have:

- AWS CLI installed and configured
- An existing EKS cluster
- `kubectl` configured for the EKS cluster
- An existing VPC
- EKS and ElastiCache deployed in the same VPC or connected VPCs
- Appropriate security groups
- Redis subnet group

Verify AWS:

```bash
aws sts get-caller-identity
```

Verify EKS:

```bash
kubectl get nodes
```

---

# 1. Create ElastiCache Subnet Group

First, create a subnet group using private subnets.

Example:

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name my-redis-subnet-group \
  --cache-subnet-group-description "Subnet group for Redis" \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy
```

Verify:

```bash
aws elasticache describe-cache-subnet-groups \
  --cache-subnet-group-name my-redis-subnet-group
```

---

# 2. Create Security Group

Create a security group for ElastiCache.

```bash
aws ec2 create-security-group \
  --group-name redis-sg \
  --description "Security group for ElastiCache Redis" \
  --vpc-id vpc-xxxxxxxx
```

Save the returned security group ID:

```text
sg-xxxxxxxx
```

## Allow Redis traffic

The recommended configuration is to allow TCP `6379` only from the EKS worker-node/pod security group.

Example:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp \
  --port 6379 \
  --source-group sg-eks-xxxxxxxx
```

> Do not allow `0.0.0.0/0` for Redis unless you have a very specific security requirement and additional protection.

---

# 3. Create ElastiCache Redis

Example:

```bash
aws elasticache create-replication-group \
  --replication-group-id my-redis \
  --replication-group-description "Redis for EKS applications" \
  --engine redis \
  --cache-node-type cache.t4g.small \
  --num-cache-clusters 1 \
  --cache-subnet-group-name my-redis-subnet-group \
  --security-group-ids sg-xxxxxxxx \
  --transit-encryption-enabled
```

Check the status:

```bash
aws elasticache describe-replication-groups \
  --replication-group-id my-redis
```

Wait until the status becomes:

```text
available
```

---

# 4. Get Redis Endpoint

Get the primary endpoint:

```bash
aws elasticache describe-replication-groups \
  --replication-group-id my-redis \
  --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address' \
  --output text
```

Example output:

```text
my-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

Save this hostname.

---

# 5. Create Kubernetes ExternalName Service

Create a file:

```text
redis-externalname.yaml
```

Add:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
spec:
  type: ExternalName
  externalName: my-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

Replace:

```text
my-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

with your actual ElastiCache endpoint.

Apply it:

```bash
kubectl apply -f redis-externalname.yaml
```

Verify:

```bash
kubectl get service redis
```

Expected:

```text
NAME    TYPE           CLUSTER-IP   EXTERNAL-IP
redis   ExternalName   <none>       my-redis.xxxxxx...
```

---

# 6. Application Connection

Applications running in the same Kubernetes namespace can connect using:

```text
redis:6379
```

Applications in another namespace can use:

```text
redis.default.svc.cluster.local:6379
```

For example:

```text
redis://redis:6379
```

---

# 7. TLS Connection

If ElastiCache was created with:

```text
--transit-encryption-enabled
```

the application should use a TLS connection.

For applications supporting Redis URI syntax:

```text
rediss://redis:6379
```

For example:

```text
rediss://redis.default.svc.cluster.local:6379
```

The exact TLS configuration depends on the Redis client used by your application.

---

# 8. Test Redis Connectivity

You can create a temporary Redis client pod.

```bash
kubectl run redis-client \
  --rm -it \
  --image=redis:7 \
  -- bash
```

Inside the container:

```bash
redis-cli -h redis -p 6379
```

If TLS is enabled:

```bash
redis-cli \
  -h redis \
  -p 6379 \
  --tls
```

If authentication is enabled, provide the appropriate Redis credentials.

---

# 9. Test DNS

From a Kubernetes pod:

```bash
nslookup redis
```

You should see the ExternalName DNS resolution.

You can also check:

```bash
kubectl get svc redis -o yaml
```

Expected:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ExternalName
  externalName: my-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

---

# 10. Important Security Configuration

The recommended network flow is:

```text
EKS Pod
   |
   | TCP 6379
   |
   v
EKS Security Group
   |
   |
   v
Redis Security Group
   |
   |
   v
ElastiCache Redis
```

The ElastiCache security group should allow:

```text
Source: EKS application/node/pod security group
Protocol: TCP
Port: 6379
```

Avoid:

```text
0.0.0.0/0
```

---

# 11. ExternalName Does Not Expose Redis to the Internet

`ExternalName` is only a Kubernetes DNS alias.

It does **not** create:

- AWS Load Balancer
- Public IP
- Internet-facing Redis
- NodePort
- Ingress

The request flow is:

```text
Application
    |
    v
redis:6379
    |
    v
Kubernetes DNS
    |
    v
ElastiCache DNS endpoint
    |
    v
AWS ElastiCache
```

This is the preferred architecture when Redis is intended to be accessed by applications inside EKS.

---

# 12. Kubernetes Application Example

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-test
  template:
    metadata:
      labels:
        app: redis-test
    spec:
      containers:
        - name: app
          image: redis:7
          command:
            - sleep
            - "3600"
```

Apply:

```bash
kubectl apply -f redis-test.yaml
```

Get the pod:

```bash
kubectl get pods
```

Enter the pod:

```bash
kubectl exec -it <pod-name> -- bash
```

Test:

```bash
redis-cli -h redis -p 6379
```

---

# 13. Recommended Production Architecture

For production workloads, consider:

```text
                    AWS
                     |
              ┌──────┴──────┐
              |              |
             EKS        ElastiCache
              |              |
        Application Pods   Redis
              |              |
              └──────┬───────┘
                     |
                Private VPC
```

Recommended settings:

- Private ElastiCache subnets
- Security groups restricted to EKS
- TLS/transit encryption enabled
- Redis authentication/authorization where applicable
- Multi-AZ configuration for production
- Automatic backups
- CloudWatch monitoring
- Encryption at rest
- No public Internet access

---

# 14. Useful Commands

### Check EKS nodes

```bash
kubectl get nodes
```

### Check Redis service

```bash
kubectl get svc redis
```

### Describe Redis service

```bash
kubectl describe svc redis
```

### Check ElastiCache

```bash
aws elasticache describe-replication-groups \
  --replication-group-id my-redis
```

### Get Redis endpoint

```bash
aws elasticache describe-replication-groups \
  --replication-group-id my-redis \
  --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address' \
  --output text
```

---

# Summary

The main components are:

```text
AWS ElastiCache Redis
        |
        | Private DNS endpoint
        |
        v
Kubernetes ExternalName Service
        |
        | redis:6379
        |
        v
EKS Applications
```

The Kubernetes service is:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ExternalName
  externalName: <ELASTICACHE_ENDPOINT>
```

This allows your applications to use a stable Kubernetes hostname:

```text
redis:6379
```

while AWS ElastiCache manages the actual Redis service.