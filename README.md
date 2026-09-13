# AWS ElastiCache Redis with Amazon EKS

This repository demonstrates how to connect applications running on an **Amazon EKS cluster** to **AWS ElastiCache for Redis** using a Kubernetes `ExternalName` Service.

Instead of running Redis inside Kubernetes, Redis is managed by AWS ElastiCache. Kubernetes provides a DNS-friendly Service name that applications can use to connect to the ElastiCache endpoint.

## Architecture

```text
                    AWS VPC
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Amazon EKS                                       │
│   ┌─────────────────────────────────────────────┐   │
│   │                                             │   │
│   │  Application Pod                            │   │
│   │       │                                     │   │
│   │       │ redis:6379                          │   │
│   │       ▼                                     │   │
│   │  Kubernetes ExternalName Service            │   │
│   │       │                                     │   │
│   └───────┼─────────────────────────────────────┘   │
│           │                                         │
│           ▼                                         │
│   AWS ElastiCache for Redis                         │
│   ┌─────────────────────────────────────────────┐   │
│   │                                             │   │
│   │  Redis Endpoint                             │   │
│   │  <redis-endpoint>:6379                      │   │
│   │                                             │   │
│   └─────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## How It Works

The Redis database is created and managed by AWS ElastiCache.

The EKS cluster does **not** run a Redis Deployment or StatefulSet.

A Kubernetes `ExternalName` Service provides an internal DNS alias:

```text
redis
```

which resolves to the AWS ElastiCache Redis endpoint.

For example:

```text
redis.default.svc.cluster.local
        |
        v
my-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

Kubernetes `ExternalName` Services work by returning a DNS CNAME rather than creating a normal Kubernetes endpoint. This makes them suitable for creating a Kubernetes DNS alias for an external service.

---

# Prerequisites

Make sure the following are installed:

- AWS CLI
- kubectl
- An existing EKS cluster
- An AWS VPC
- An ElastiCache subnet group
- An ElastiCache security group
- Appropriate AWS IAM permissions

Check your tools:

```bash
aws --version
kubectl version --client
```

Verify that kubectl is connected to EKS:

```bash
kubectl get nodes
```

---

# 1. Configure AWS Credentials

Configure AWS CLI:

```bash
aws configure
```

Verify:

```bash
aws sts get-caller-identity
```

---

# 2. Create ElastiCache Subnet Group

ElastiCache should be deployed into subnets that can communicate with the EKS cluster.

Example:

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name redis-subnet-group \
  --cache-subnet-group-description "Subnet group for Redis" \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy
```

Verify:

```bash
aws elasticache describe-cache-subnet-groups \
  --cache-subnet-group-name redis-subnet-group
```

---

# 3. Create Security Group

Create a security group for ElastiCache:

```bash
aws ec2 create-security-group \
  --group-name redis-elasticache-sg \
  --description "Security group for ElastiCache Redis" \
  --vpc-id vpc-xxxxxxxx
```

Get the security group ID:

```bash
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=redis-elasticache-sg \
  --query 'SecurityGroups[0].GroupId' \
  --output text
```

Example:

```text
sg-0123456789abcdef
```

Allow Redis traffic from the EKS worker-node/pod security group.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef \
  --protocol tcp \
  --port 6379 \
  --source-group sg-EKS-XXXXXXXX
```

> Do not allow `0.0.0.0/0` access to Redis.

Redis should normally remain private inside the VPC.

---

# 4. Create ElastiCache Redis

Example:

```bash
aws elasticache create-replication-group \
  --replication-group-id eks-redis \
  --replication-group-description "Redis for EKS applications" \
  --engine redis \
  --cache-node-type cache.t4g.small \
  --num-cache-clusters 1 \
  --cache-subnet-group-name redis-subnet-group \
  --security-group-ids sg-0123456789abcdef \
  --transit-encryption-enabled
```

Check the status:

```bash
aws elasticache describe-replication-groups \
  --replication-group-id eks-redis
```

Wait until the replication group becomes available.

---

# 5. Get the Redis Endpoint

Run:

```bash
aws elasticache describe-replication-groups \
  --replication-group-id eks-redis \
  --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address' \
  --output text
```

Example output:

```text
eks-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

You will use this hostname in the Kubernetes `ExternalName` Service.

---

# 6. Create Kubernetes ExternalName Service

Create:

```text
redis-externalname.yaml
```

with:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
spec:
  type: ExternalName
  externalName: eks-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

Replace:

```text
eks-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

with your actual ElastiCache endpoint.

Apply it:

```bash
kubectl apply -f redis-externalname.yaml
```

Verify:

```bash
kubectl get svc redis
```

Expected output:

```text
NAME    TYPE           CLUSTER-IP   EXTERNAL-IP
redis   ExternalName   <none>       eks-redis.xxxxxx.ng.0001.use1.cache.amazonaws.com
```

---

# 7. Test DNS Resolution

Run a temporary pod:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- sh
```

Inside the pod:

```bash
nslookup redis.default.svc.cluster.local
```

You should see the ElastiCache hostname.

---

# 8. Test Redis Connection

If TLS is enabled, use a Redis client that supports TLS.

For example, create a temporary Redis client pod:

```bash
kubectl run redis-client \
  --image=redis:latest \
  --restart=Never \
  --rm -it \
  -- bash
```

Then connect:

```bash
redis-cli -h redis -p 6379 --tls
```

If authentication is enabled:

```bash
redis-cli \
  -h redis \
  -p 6379 \
  --tls \
  --user default \
  --pass '<REDIS_PASSWORD>'
```

Test Redis:

```text
PING
```

Expected:

```text
PONG
```

Test SET:

```text
SET test "hello"
```

Expected:

```text
OK
```

Test GET:

```text
GET test
```

Expected:

```text
"hello"
```

---

# 9. Use Redis From an Application

Applications running inside EKS can use the Kubernetes DNS name:

```text
redis:6379
```

For a fully qualified Kubernetes DNS name:

```text
redis.default.svc.cluster.local:6379
```

For example:

```text
REDIS_HOST=redis
REDIS_PORT=6379
```

If TLS is enabled:

```text
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_TLS=true
```

---

# 10. Example Application Deployment

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-test-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis-test-app
  template:
    metadata:
      labels:
        app: redis-test-app
    spec:
      containers:
        - name: app
          image: your-app-image:latest
          env:
            - name: REDIS_HOST
              value: "redis"
            - name: REDIS_PORT
              value: "6379"
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get pods
```

---

# 11. Important Security Configuration

The recommended network flow is:

```text
EKS Pod
   |
   | TCP 6379
   |
   v
ElastiCache Security Group
   |
   v
ElastiCache Redis
```

The ElastiCache security group should allow Redis traffic only from the appropriate EKS security group.

Do **not** expose Redis with:

```yaml
type: LoadBalancer
```

and do not open Redis to:

```text
0.0.0.0/0
```

Redis is a database service and should normally remain private.

---

# 12. ExternalName vs LoadBalancer

`ExternalName` does **not** expose Redis to the public Internet.

It creates a Kubernetes DNS alias.

```text
Application
     |
     v
redis.default.svc.cluster.local
     |
     v
ExternalName
     |
     v
AWS ElastiCache endpoint
```

This is different from:

```yaml
type: LoadBalancer
```

A LoadBalancer Service provisions an external load-balancing path, while an ExternalName Service is primarily a DNS alias.

For ElastiCache, `ExternalName` is useful when applications expect a Kubernetes Service name but the actual database is managed outside Kubernetes.

---

# 13. Troubleshooting

### Check the Service

```bash
kubectl get svc redis
```

### Describe the Service

```bash
kubectl describe svc redis
```

### Check DNS

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- nslookup redis
```

### Check EKS nodes

```bash
kubectl get nodes -o wide
```

### Check AWS ElastiCache

```bash
aws elasticache describe-replication-groups \
  --replication-group-id eks-redis
```

### Check security groups

Make sure:

```text
EKS security group
       |
       | TCP 6379
       v
ElastiCache security group
```

is allowed.

### Check VPC routing

EKS and ElastiCache should have network connectivity through the appropriate VPC/subnets/routes.

---

# 14. Cleanup

Delete the Kubernetes Service:

```bash
kubectl delete -f redis-externalname.yaml
```

Delete the ElastiCache replication group:

```bash
aws elasticache delete-replication-group \
  --replication-group-id eks-redis
```

Delete the subnet group:

```bash
aws elasticache delete-cache-subnet-group \
  --cache-subnet-group-name redis-subnet-group
```

---

# Repository Structure

A recommended repository structure is:

```text
redis-k8s/
├── README.md
├── redis-externalname.yaml
└── deployment.yaml
```

The important point is that Redis itself is **not deployed by Kubernetes** in this architecture. AWS ElastiCache manages Redis, while Kubernetes provides the DNS abstraction through `ExternalName`.

## References

- Kubernetes ConfigMap/Redis documentation
- AWS ElastiCache documentation
- Kubernetes Service documentation

## Summary

This setup provides:

- AWS-managed Redis
- Redis outside the EKS cluster
- Kubernetes DNS name for Redis
- Private VPC connectivity
- Security-group-based access control
- Optional TLS encryption
- No Redis StatefulSet/Deployment required in EKS

The final application connection is simply:

```text
redis:6379
```

with `redis` resolving through Kubernetes to the AWS ElastiCache endpoint.