# gitops

ArgoCD GitOps repository for retail-store-sample-app. Implements app-of-apps pattern with environment-specific overlays.

## Directory Structure

```
gitops/
├── argocd/                    # ArgoCD configuration
│   ├── applications/         # Application manifests
│   ├── projects/             # ArgoCD projects
│   └── rbac/                 # RBAC configuration
├── environments/             # Environment-specific configs
│   ├── dev/                  # Development environment
│   ├── stage/                # Staging environment
│   └── prod/                 # Production environment
├── apps/                     # Application configurations
│   ├── platform/             # Platform components
│   ├── ui/                   # UI service
│   ├── cart/                 # Cart service
│   ├── catalog/              # Catalog service
│   ├── checkout/             # Checkout service
│   └── orders/               # Orders service
└── README.md
```

## ArgoCD Setup

### Prerequisites
- ArgoCD installed in cluster
- Git repository accessible by ArgoCD
- Appropriate RBAC permissions

### Installation

1. **Install ArgoCD CLI**
```bash
# Via cluster-ops playbook
ansible-playbook cluster-ops/playbooks/install-tools.yml
```

2. **Bootstrap ArgoCD**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

3. **Initial Admin Password**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

4. **Login to ArgoCD**
```bash
argocd login <argocd-server-url> --username admin --password <password>
```

5. **Create Root Application**
```bash
kubectl apply -f gitops/argocd/applications/root-app.yaml
```

## Application Structure

### Root Application
The root application (`root-app.yaml`) deploys:
- Platform components (ingress, cert-manager, monitoring)
- Environment-specific applications

### Environment Applications
Each environment has its own application that deploys:
- All microservices with environment-specific values
- Shared platform components

### Service Applications
Individual service applications for:
- Independent deployment
- Rollback capabilities
- Service-specific configurations

## Environment Overlays

### Development (dev/)
- Minimal resource allocation
- Self-signed certificates
- Single replica deployments
- Debug logging enabled

### Staging (stage/)
- Medium resource allocation
- Let's Encrypt certificates
- Auto-scaling enabled
- Production-like configuration

### Production (prod/)
- High resource allocation
- Production certificates
- Multi-AZ deployment
- Security hardening

## Deployment Strategy

### App-of-Apps Pattern
```
Root App
├── Platform App (shared infra)
├── Dev Environment App
│   ├── UI App (dev values)
│   ├── Cart App (dev values)
│   ├── Catalog App (dev values)
│   ├── Checkout App (dev values)
│   └── Orders App (dev values)
├── Stage Environment App
│   └── [Same structure as dev]
└── Prod Environment App
    └── [Same structure as dev]
```

### Sync Waves
Applications are deployed in waves:
1. **Wave 0**: Platform components
2. **Wave 1**: Databases and storage
3. **Wave 2**: Core services (catalog, cart)
4. **Wave 3**: Business logic (checkout, orders)
5. **Wave 4**: UI and ingress

## RBAC Configuration

### Projects
- **platform**: Platform components
- **dev**: Development environment
- **stage**: Staging environment
- **prod**: Production environment

### Roles
- **admin**: Full access
- **developer**: Deploy and sync applications
- **viewer**: Read-only access

## Notifications

### Slack Integration
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: retail-store-alerts
    notifications.argoproj.io/subscribe.on-sync-failed.slack: retail-store-alerts
```

### Email Notifications
Configured for:
- Sync failures
- Deployment completions
- Health status changes

## Monitoring

### Application Health
```bash
# Check application status
argocd app get retail-store-dev

# List all applications
argocd app list

# Get application events
argocd app get retail-store-dev --show-events
```

### Sync Status
```bash
# Force sync
argocd app sync retail-store-dev

# Get sync status
argocd app get retail-store-dev --show-operation
```

## Troubleshooting

### Application Not Syncing
1. Check ArgoCD UI for errors
2. Verify repository access: `argocd repo list`
3. Check application manifests: `argocd app manifests retail-store-dev`
4. Review sync logs: `argocd app logs retail-store-dev`

### Certificate Issues
1. Check cert-manager status: `kubectl get certificate -A`
2. Verify issuer configuration: `kubectl get clusterissuer`
3. Check DNS propagation: `nslookup ui.dev.corp.example.internal`

### Resource Constraints
1. Check pod status: `kubectl get pods -A`
2. Review resource usage: `kubectl top pods -A`
3. Check HPA status: `kubectl get hpa -A`

## Security

### Repository Access
- SSH keys for private repositories
- Token-based authentication for GitHub
- Repository allowlists in ArgoCD

### RBAC Best Practices
- Least privilege principle
- Regular role audits
- Service account usage for CI/CD

### Secret Management
- Sealed Secrets for sensitive data
- External secret stores (AWS Secrets Manager)
- No secrets in Git repository

## CI/CD Integration

### GitHub Actions
- Automated testing on PRs
- Deployment on merge to main
- Rollback on failures

### ArgoCD Image Updater
- Automatic image updates
- Semantic versioning support
- Multi-source registry support

## Backup and Recovery

### Application Backups
```bash
# Export applications
argocd app export retail-store-dev > backup-dev.yaml

# Import applications
argocd app create -f backup-dev.yaml
```

### Disaster Recovery
1. Backup ArgoCD configuration
2. Document manual deployment procedures
3. Test recovery procedures regularly
