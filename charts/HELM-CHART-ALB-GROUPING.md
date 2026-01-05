# Helm Charts with ALB Grouping and External DNS

This guide explains how to use the updated vote and result Helm charts with ALB grouping and External DNS for cost-effective, automated DNS management.

## Overview

The charts now support:
- ✅ **ALB Grouping** - Share one ALB across all applications
- ✅ **External DNS** - Automatic Route53 DNS record creation
- ✅ **Wildcard Certificates** - Single ACM cert for all apps
- ✅ **Flexible Configuration** - Easy to enable/disable features
- ✅ **Production Ready** - Health checks, SSL redirect, security

## Quick Start

### Prerequisites

1. **EKS Cluster** with AWS Load Balancer Controller installed
2. **External DNS** installed (from terraform: `install_external_dns = true`)
3. **ACM Certificate** (preferably wildcard: `*.example.com`)
4. **Route53 Hosted Zone** for your domain

### 1. Create Shared Configuration File

Create `shared-alb-values.yaml` with common settings:

```yaml
# shared-alb-values.yaml
# Common configuration for all apps using shared ALB

ingress:
  enabled: true
  className: alb
  domain: example.com  # Your domain
  
  alb:
    enabled: true
    groupName: cluster-shared-alb  # All apps use this same name
    scheme: internet-facing
    targetType: ip
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/your-wildcard-cert
    
    healthcheck:
      path: /
      protocol: HTTP
      intervalSeconds: 15
      timeoutSeconds: 5
      successCodes: "200"
  
  externalDns:
    enabled: true
    # hostname set per-app in individual values files
```

### 2. Deploy Vote App

Create `vote-values.yaml`:

```yaml
# vote-values.yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-vote
  tag: latest

ingress:
  alb:
    groupOrder: "10"  # Priority for vote app
  externalDns:
    hostname: vote.example.com
```

Deploy:
```bash
helm install vote ./charts/vote \
  -f shared-alb-values.yaml \
  -f vote-values.yaml \
  --namespace voting-app \
  --create-namespace
```

### 3. Deploy Result App

Create `result-values.yaml`:

```yaml
# result-values.yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-result
  tag: latest

postgresql:
  server: postgres-service
  postgresUsername: postgres
  postgresqlPassword: postgres

ingress:
  alb:
    groupOrder: "20"  # Priority for result app
  externalDns:
    hostname: result.example.com
```

Deploy:
```bash
helm install result ./charts/result \
  -f shared-alb-values.yaml \
  -f result-values.yaml \
  --namespace voting-app
```

### 4. Verify Shared ALB

```bash
# Check ingresses (both should exist)
kubectl get ingress -n voting-app

# Verify both use same ALB
kubectl get ingress -n voting-app -o yaml | grep "group.name"

# Check only ONE ALB was created
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-clustershar`)].LoadBalancerName'

# Verify DNS records
nslookup vote.example.com
nslookup result.example.com
# Both should resolve to the SAME ALB hostname
```

## Configuration Options

### ALB Grouping Settings

```yaml
ingress:
  alb:
    # Group name - MUST be identical across all apps to share ALB
    groupName: "cluster-shared-alb"
    
    # Group order - controls routing priority
    # Lower number = higher priority (1-1000)
    # Use increments of 10 for flexibility
    groupOrder: "10"  # vote: 10, result: 20, api: 30, etc.
```

### External DNS Settings

```yaml
ingress:
  externalDns:
    enabled: true
    
    # Explicit hostname
    hostname: "myapp.example.com"
    
    # Or use default: <release>-<chart>.<domain>
    # hostname: ""  # Results in: vote-vote.example.com
    
    # Custom TTL
    ttl: "300"  # 5 minutes
```

### Certificate Configuration

**Option 1: Wildcard Certificate (Recommended)**
```yaml
ingress:
  alb:
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/wildcard-cert
```

Covers: `*.example.com` and `example.com`

**Option 2: Multiple Certificates**
```yaml
ingress:
  alb:
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/cert1,arn:aws:acm:us-east-1:123456789012:certificate/cert2
```

**Option 3: No HTTPS** (Not recommended)
```yaml
ingress:
  alb:
    certificateArn: ""  # Leave empty for HTTP only
```

### Health Check Configuration

```yaml
ingress:
  alb:
    healthcheck:
      path: /health        # Custom health endpoint
      protocol: HTTP       # or HTTPS
      intervalSeconds: 30  # Check every 30 seconds
      timeoutSeconds: 10   # 10 second timeout
      successCodes: "200,301,302"  # Acceptable status codes
```

## Deployment Scenarios

### Scenario 1: Development Environment

**Simple setup, single domain, shared ALB**

```yaml
# dev-shared-values.yaml
ingress:
  enabled: true
  className: alb
  domain: dev.example.com
  
  alb:
    enabled: true
    groupName: dev-shared-alb
    scheme: internet-facing
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/dev-wildcard
  
  externalDns:
    enabled: true
```

Deploy all apps:
```bash
# Vote
helm install vote ./charts/vote \
  -f dev-shared-values.yaml \
  --set ingress.alb.groupOrder=10 \
  --set ingress.externalDns.hostname=vote.dev.example.com

# Result
helm install result ./charts/result \
  -f dev-shared-values.yaml \
  --set ingress.alb.groupOrder=20 \
  --set ingress.externalDns.hostname=result.dev.example.com
```

### Scenario 2: Production Environment

**HA setup, multiple replicas, strict health checks**

```yaml
# prod-shared-values.yaml
replicaCount: 3

ingress:
  enabled: true
  className: alb
  domain: example.com
  
  alb:
    enabled: true
    groupName: prod-shared-alb
    scheme: internet-facing
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/prod-wildcard
    
    # Strict health checks
    healthcheck:
      path: /health
      intervalSeconds: 10
      timeoutSeconds: 5
      successCodes: "200"
    
    # Tag ALB (only on first app)
    tags: "Environment=Production,Team=Platform,CostCenter=Engineering"
    
    # Custom ALB name (only on first app)
    loadBalancerName: "prod-voting-app-alb"
  
  externalDns:
    enabled: true
    ttl: "60"  # Low TTL for faster failover
```

Deploy:
```bash
# Vote (first app - includes ALB tags and name)
helm install vote ./charts/vote \
  -f prod-shared-values.yaml \
  --set ingress.alb.groupOrder=10 \
  --set ingress.externalDns.hostname=vote.example.com \
  --namespace production

# Result (subsequent app - inherits ALB settings)
helm install result ./charts/result \
  -f prod-shared-values.yaml \
  --set ingress.alb.groupOrder=20 \
  --set ingress.externalDns.hostname=result.example.com \
  --namespace production
```

### Scenario 3: Multi-Environment Setup

**Use same cluster for dev, staging, and prod with separate ALBs**

```yaml
# dev-values.yaml
ingress:
  alb:
    groupName: dev-alb
    groupOrder: "10"
  externalDns:
    hostname: vote.dev.example.com

# staging-values.yaml
ingress:
  alb:
    groupName: staging-alb
    groupOrder: "10"
  externalDns:
    hostname: vote.staging.example.com

# prod-values.yaml
ingress:
  alb:
    groupName: prod-alb
    groupOrder: "10"
  externalDns:
    hostname: vote.example.com
```

**Result**: 3 ALBs (one per environment), but still saves cost vs per-app ALBs

### Scenario 4: Internal + External Split

**Public apps on internet-facing ALB, admin on internal ALB**

```yaml
# public-apps-values.yaml
ingress:
  alb:
    groupName: public-alb
    scheme: internet-facing

# internal-apps-values.yaml
ingress:
  alb:
    groupName: internal-alb
    scheme: internal
```

Deploy:
```bash
# Public vote
helm install vote ./charts/vote \
  -f public-apps-values.yaml \
  --set ingress.externalDns.hostname=vote.example.com

# Internal admin
helm install admin ./charts/result \
  -f internal-apps-values.yaml \
  --set ingress.externalDns.hostname=admin.internal.example.com
```

## Using with ArgoCD

### ArgoCD Application with Shared ALB

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vote
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo
    targetRevision: main
    path: charts/vote
    helm:
      values: |
        image:
          repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-vote
          tag: v1.2.0
        
        ingress:
          enabled: true
          className: alb
          domain: example.com
          
          alb:
            enabled: true
            groupName: cluster-shared-alb
            groupOrder: "10"
            certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/wildcard
          
          externalDns:
            enabled: true
            hostname: vote.example.com
  
  destination:
    server: https://kubernetes.default.svc
    namespace: voting-app
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### ApplicationSet for Multiple Apps

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: voting-app
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - app: vote
        order: "10"
        hostname: vote.example.com
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-vote
      - app: result
        order: "20"
        hostname: result.example.com
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-result
  
  template:
    metadata:
      name: '{{app}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/your-repo
        targetRevision: main
        path: 'charts/{{app}}'
        helm:
          values: |
            image:
              repository: '{{image}}'
              tag: latest
            
            ingress:
              enabled: true
              className: alb
              domain: example.com
              
              alb:
                enabled: true
                groupName: cluster-shared-alb
                groupOrder: "{{order}}"
                certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/wildcard
              
              externalDns:
                enabled: true
                hostname: '{{hostname}}'
      
      destination:
        server: https://kubernetes.default.svc
        namespace: voting-app
      
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Verification

### Check Ingress Configuration

```bash
# View ingress resources
kubectl get ingress -n voting-app

# Check ALB annotations
kubectl get ingress vote-vote -n voting-app -o yaml | grep -A20 "annotations:"

# Verify group.name is set
kubectl get ingress -n voting-app -o jsonpath='{.items[*].metadata.annotations.alb\.ingress\.kubernetes\.io/group\.name}' | tr ' ' '\n' | sort -u
# Should show only: cluster-shared-alb

# Check group orders
kubectl get ingress -n voting-app -o custom-columns=NAME:.metadata.name,ORDER:.metadata.annotations.alb\.ingress\.kubernetes\.io/group\.order
```

### Check ALB Creation

```bash
# List ALBs (should see only ONE for shared-alb group)
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-clustershar`)].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}' \
  --output table

# Get ALB ARN
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-clustershar`)].LoadBalancerArn' \
  --output text)

# Check listener rules (one per ingress)
LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query 'Listeners[?Port==`443`].ListenerArn' \
  --output text)

aws elbv2 describe-rules --listener-arn $LISTENER_ARN \
  --query 'Rules[*].[Priority,Conditions[0].Values[0]]' \
  --output table
```

### Check External DNS Records

```bash
# Watch External DNS create records
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --follow

# Check DNS resolution
nslookup vote.example.com
nslookup result.example.com

# Both should resolve to the same ALB hostname
dig vote.example.com +short
dig result.example.com +short
```

### Test HTTPS Access

```bash
# Test vote app
curl -I https://vote.example.com

# Test result app
curl -I https://result.example.com

# Verify certificate
echo | openssl s_client -connect vote.example.com:443 -servername vote.example.com 2>/dev/null | openssl x509 -noout -subject
```

## Common Patterns

### Pattern 1: Basic Voting App Deployment

**Single shared ALB, wildcard certificate, automatic DNS**

```bash
# Install vote
helm install vote ./charts/vote \
  --set image.repository=123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-vote \
  --set image.tag=v1.0.0 \
  --set ingress.enabled=true \
  --set ingress.className=alb \
  --set ingress.domain=example.com \
  --set ingress.alb.enabled=true \
  --set ingress.alb.groupName=cluster-shared-alb \
  --set ingress.alb.groupOrder=10 \
  --set ingress.alb.certificateArn=arn:aws:acm:us-east-1:123456789012:certificate/wildcard \
  --set ingress.externalDns.enabled=true \
  --set ingress.externalDns.hostname=vote.example.com \
  --namespace voting-app \
  --create-namespace

# Install result
helm install result ./charts/result \
  --set image.repository=123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-result \
  --set image.tag=v1.0.0 \
  --set ingress.enabled=true \
  --set ingress.className=alb \
  --set ingress.domain=example.com \
  --set ingress.alb.enabled=true \
  --set ingress.alb.groupName=cluster-shared-alb \
  --set ingress.alb.groupOrder=20 \
  --set ingress.alb.certificateArn=arn:aws:acm:us-east-1:123456789012:certificate/wildcard \
  --set ingress.externalDns.enabled=true \
  --set ingress.externalDns.hostname=result.example.com \
  --namespace voting-app
```

### Pattern 2: Environment-Specific Deployment

**Use Helm values files per environment**

```yaml
# environments/dev.yaml
image:
  tag: develop

ingress:
  domain: dev.example.com
  alb:
    groupName: dev-alb
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/dev-wildcard
  externalDns:
    hostname: vote.dev.example.com
```

```yaml
# environments/prod.yaml
image:
  tag: v1.0.0

replicaCount: 3

ingress:
  domain: example.com
  alb:
    groupName: prod-alb
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/prod-wildcard
    healthcheck:
      intervalSeconds: 10
  externalDns:
    hostname: vote.example.com
    ttl: "60"
```

Deploy:
```bash
# Dev
helm install vote ./charts/vote -f environments/dev.yaml -n dev

# Prod
helm install vote ./charts/vote -f environments/prod.yaml -n production
```

### Pattern 3: GitOps with Kustomize

**Base configuration + environment overlays**

```
k8s/
├── base/
│   ├── vote-values.yaml
│   └── result-values.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── values-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── values-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── values-patch.yaml
```

## Upgrading Deployments

### Update Image Tag

```bash
# Upgrade vote to new version
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set image.tag=v1.1.0 \
  --namespace voting-app

# Verify rollout
kubectl rollout status rollout/vote-vote -n voting-app
```

### Update Hostname

```bash
# Change DNS record
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.externalDns.hostname=voting.example.com \
  --namespace voting-app

# External DNS will update Route53 automatically
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --tail=20
```

### Migrate to Shared ALB

If you have existing ingresses without grouping:

```bash
# Upgrade to use shared ALB
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.alb.groupName=cluster-shared-alb \
  --set ingress.alb.groupOrder=10 \
  --namespace voting-app

# Old ALB will be deleted automatically
# Verify
aws elbv2 describe-load-balancers
```

## Troubleshooting

### Issue: Ingress Not Created

**Check Helm release**:
```bash
helm get values vote -n voting-app
```

**Verify ingress.enabled = true**:
```bash
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.enabled=true \
  --namespace voting-app
```

### Issue: Multiple ALBs Created

**Check group names match exactly**:
```bash
kubectl get ingress -n voting-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.alb\.ingress\.kubernetes\.io/group\.name}{"\n"}{end}'
```

**Common causes**:
- Typo in `groupName` (case-sensitive!)
- Different `scheme` values
- Missing `alb.groupName` on some ingresses

**Fix**:
```bash
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.alb.groupName=cluster-shared-alb \
  --namespace voting-app
```

### Issue: DNS Record Not Created

**Check External DNS logs**:
```bash
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --tail=50
```

**Verify annotation**:
```bash
kubectl get ingress vote-vote -n voting-app -o jsonpath='{.metadata.annotations.external-dns\.alpha\.kubernetes\.io/hostname}'
```

**Check hostname is set**:
```bash
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.externalDns.enabled=true \
  --set ingress.externalDns.hostname=vote.example.com \
  --namespace voting-app
```

### Issue: Health Checks Failing

**Check target health**:
```bash
# Get target group ARN
TG_ARN=$(kubectl get targetgroupbindings -n voting-app -o jsonpath='{.items[0].status.targetGroupARN}')

# Check target health
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

**Update health check path**:
```bash
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.alb.healthcheck.path=/healthz \
  --namespace voting-app
```

## Complete Example Values

### vote-production.yaml

```yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-vote
  tag: v1.0.0
  pullPolicy: IfNotPresent

redis:
  host: redis-service
  usePassword: true

replicaCount: 3

ingress:
  enabled: true
  className: alb
  domain: example.com
  
  alb:
    enabled: true
    groupName: prod-shared-alb
    groupOrder: "10"
    scheme: internet-facing
    targetType: ip
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/wildcard
    
    healthcheck:
      path: /
      protocol: HTTP
      intervalSeconds: 15
      timeoutSeconds: 5
      successCodes: "200"
    
    # Only set these on the FIRST app in the group
    loadBalancerName: prod-voting-alb
    tags: Environment=Production,Team=Platform
  
  externalDns:
    enabled: true
    hostname: vote.example.com
    ttl: "60"
```

### result-production.yaml

```yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1:123456789012.dkr.ecr.us-east-1.amazonaws.com/octopus-workers-result
  tag: v1.0.0
  pullPolicy: IfNotPresent

postgresql:
  server: postgres-service
  postgresUsername: postgres
  postgresqlPassword: ${POSTGRES_PASSWORD}  # Use secret management

replicaCount: 3

ingress:
  enabled: true
  className: alb
  domain: example.com
  
  alb:
    enabled: true
    groupName: prod-shared-alb  # SAME as vote
    groupOrder: "20"  # Different order
    scheme: internet-facing
    targetType: ip
    certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/wildcard
    
    healthcheck:
      path: /
      protocol: HTTP
      intervalSeconds: 15
      timeoutSeconds: 5
      successCodes: "200"
  
  externalDns:
    enabled: true
    hostname: result.example.com
    ttl: "60"
```

## Best Practices

1. ✅ **Use Shared Values File**: Create `shared-alb-values.yaml` for common config
2. ✅ **Set Group Order**: Use increments of 10 (10, 20, 30, etc.) for flexibility
3. ✅ **Wildcard Certificate**: Use one cert for all apps (`*.example.com`)
4. ✅ **External DNS**: Enable for automatic DNS management
5. ✅ **Health Checks**: Configure appropriate paths and intervals
6. ✅ **Tags**: Set on first app only (ALB-level, not ingress-level)
7. ✅ **Test First**: Use `helm template` to preview generated manifests

## Migration Guide

### From Existing Ingresses

If you have existing ingresses without grouping:

```bash
# Step 1: Get current certificate ARN
CERT_ARN=$(kubectl get ingress vote-vote -n voting-app -o jsonpath='{.metadata.annotations.alb\.ingress\.kubernetes\.io/certificate-arn}')

# Step 2: Upgrade with grouping
helm upgrade vote ./charts/vote \
  --reuse-values \
  --set ingress.alb.groupName=cluster-shared-alb \
  --set ingress.alb.groupOrder=10 \
  --set ingress.alb.certificateArn=$CERT_ARN \
  --namespace voting-app

# Step 3: Wait for new ALB
kubectl get ingress vote-vote -n voting-app -w

# Step 4: Old ALB will be deleted automatically
```

## Cost Savings

**Example: 5 Apps (vote, result, api, admin, docs)**

Without Grouping:
- 5 ALBs × $16/month = **$80/month**

With Grouping:
- 1 ALB × $16/month = **$16/month**
- **Savings: $64/month (80%)**

Plus reduced data transfer costs and simpler management!

## References

- [AWS Load Balancer Controller IngressGroup](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.8/guide/ingress/ingress_group/)
- [External DNS Annotations](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/annotations/annotations.md)
- [Helm Values Documentation](https://helm.sh/docs/chart_template_guide/values_files/)