EKS Mega Project: Production-Grade Helm Chart
This project demonstrates a professional, reusable, and parameterized Helm Chart designed for Amazon EKS. It abstracts complex Kubernetes manifests into a simplified configuration system, allowing for seamless deployments across Dev, Staging, and Production environments. Built with best practices in mind, this chart emphasizes security, scalability, reliability, and EKS-specific optimizations like ALB integration and IRSA support.
Project Architecture
Core Features
•	Environment Abstraction: Single chart logic with environment-specific overrides (e.g., values-prod.yaml for production settings like higher replicas and resource limits).
•	Zero-Downtime Updates: Configured with RollingUpdate strategies, liveness/readiness/startup probes, and config/secret checksum annotations for automatic pod restarts on changes.
•	AWS Integration: Native support for AWS Application Load Balancer (ALB) via Ingress annotations (e.g., target-type: ip for high performance, ACM for TLS).
•	Auto-Scaling: Horizontal Pod Autoscaler (HPA) with CPU/memory metrics and customizable scaling behavior (e.g., stabilization windows).
•	Security First: Pod and container security contexts (non-root users, capability drops, read-only root filesystem), optional IRSA for AWS service access, and encrypted Secrets management.
•	Reliability Enhancements: Pod Disruption Budgets (PDB) for high availability, affinity/anti-affinity for multi-AZ spreading, and tolerations/node selectors for targeted scheduling.
•	GitOps Ready: Includes GitHub Actions workflow for automated linting, testing, and deployments with --atomic for safe rollbacks.
•	Monitoring Ready: Annotations for Prometheus scraping and integration with EKS metrics.
This chart is suitable for deploying stateless web apps (e.g., nginx-based or custom backends) and can be extended for stateful workloads.

📂 Project Structure
text
eks-mega-chart/
├── templates/              # Kubernetes Manifest Templates
│   ├── _helpers.tpl        # Reusable naming, label, and service account logic
│   ├── configmap.yaml      # Non-sensitive config data
│   ├── deployment.yaml     # Core application deployment with security and probes
│   ├── hpa.yaml            # Horizontal Pod Autoscaler
│   ├── ingress.yaml        # AWS ALB Ingress configuration
│   ├── pdb.yaml            # Pod Disruption Budget for HA
│   ├── secret.yaml         # Sensitive data secrets
│   ├── service.yaml        # Service exposure
│   └── serviceaccount.yaml # ServiceAccount with optional IRSA annotations
├── .github/                # CI/CD workflows
│   └── workflows/          
│       └── deploy.yml      # GitHub Actions for lint, test, deploy
├── .gitignore              # Ignore local files and secrets
├── Chart.yaml              # Chart metadata and versioning
├── LICENSE                 # e.g., MIT License
├── README.md               # This file
├── values.yaml             # Default (Dev) configuration
└── values-prod.yaml        # Production overrides

🚀 Getting Started
Prerequisites
•	Kubernetes 1.22+ (tested on EKS).
•	Helm v3.8.0+.
•	EKS cluster with:
o	AWS Load Balancer Controller installed (for ALB Ingress).
o	Metrics Server (for HPA).
o	Optional: Cluster Autoscaler, Prometheus for monitoring, External Secrets Operator for prod secrets.
•	AWS CLI configured for EKS access (if using CI/CD).
1. Installation (Dev)
Deploy with default dev settings (e.g., minimal resources, no autoscaling):
Bash
helm install my-app ./eks-mega-chart --namespace dev --create-namespace
2. Installation (Production)
Deploy with production overrides (e.g., autoscaling, ALB Ingress, resource limits):
Bash
helm upgrade --install my-app-prod ./eks-mega-chart \
  --namespace production \
  --create-namespace \
  -f ./eks-mega-chart/values-prod.yaml \
  --atomic  # Ensures automatic rollback on failure
3. Upgrades and Rollbacks
Upgrade with new values:
Bash
helm upgrade my-app-prod ./eks-mega-chart -f values-prod.yaml --namespace production
Rollback to a previous revision if issues arise:
Bash
helm rollback my-app-prod 1 --namespace production
4. Testing the Chart
Lint for errors:
Bash
helm lint ./eks-mega-chart
Dry-run template rendering:
Bash
helm template my-app-prod ./eks-mega-chart -f values-prod.yaml | kubectl apply -f - --dry-run=client
Deploy to a test cluster and verify with:
Bash
kubectl get all -n production
kubectl describe ingress my-app-prod -n production  # Check ALB status

## ⚙️ Configuration Reference

You can use `values.yaml` for defaults and override them via `-f` flags or custom files. Key parameters:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of desired pods | `1` |
| `image.repository` | Docker image repository | `nginx` (customize for your app) |
| `image.tag` | Docker image tag (defaults to `Chart.appVersion`) | `""` |
| `service.type` | Service type (e.g., ClusterIP) | `ClusterIP` |
| `service.port` | Service port | `80` |
| `resources.requests` | Resource requests (CPU/memory) | `{ cpu: 100m, memory: 128Mi }` |
| `resources.limits` | Resource limits (CPU/memory) | `{}` (add for production) |
| `livenessProbe / readinessProbe / startupProbe` | Health checks with delays/timeouts | HTTP GET on `/` with sensible defaults |
| `autoscaling.enabled` | Enable Horizontal Pod Autoscaler (HPA) | `false` |
| `autoscaling.minReplicas / maxReplicas` | HPA replica range | `1 / 5` |
| `autoscaling.targetCPUUtilizationPercentage` | CPU scaling threshold | `80` |
| `autoscaling.targetMemoryUtilizationPercentage` | Memory scaling threshold | `80` (optional) |
| `autoscaling.behavior` | Scaling behavior (e.g., stabilization windows) | `{}` |
| `ingress.enabled` | Enable ALB Ingress | `false` |
| `ingress.className` | Ingress class (e.g., alb) | `alb` |
| `ingress.annotations` | ALB annotations (e.g., scheme, target-type, certificate-arn) | EKS-optimized defaults |
| `ingress.hosts` | Host configurations | `chart-example.local` |
| `ingress.tls` | TLS secrets | `[]` |
| `podSecurityContext` | Pod-level security (e.g., `fsGroup: 1000`) | `{}` |
| `securityContext` | Container-level security (e.g., `runAsNonRoot: true`) | Defaults to secure settings |
| `affinity / nodeSelector / tolerations` | Pod placement rules | `{}` |
| `pdb.enabled` | Enable Pod Disruption Budget | `false` |
| `pdb.minAvailable` | Minimum available pods during disruptions | `1` |
| `serviceAccount.create` | Create ServiceAccount | `true` |
| `serviceAccount.annotations` | Annotations (e.g., for IRSA: `eks.amazonaws.com/role-arn`) | `{}` |
| `configData` | Non-sensitive environment variables (e.g., `DB_HOSTNAME`) | Example values |
| `secretData` | Sensitive environment variables (base64-encoded by Helm) | Example placeholders (use external secrets in production) |
| `volumeMounts / volumes` | Additional volumes (e.g., `emptyDir` for `/tmp` with read-only FS) | `[]` |

> For full details, see `values.yaml`. Customize for your app (e.g., update probes to match your health endpoints).

🛠 CI/CD Pipeline
The repository includes a GitHub Actions workflow in .github/workflows/deploy.yml for automated deployments:
1.	Lint: Runs helm lint on changes.
2.	Authenticate: Uses AWS credentials (via secrets) to access EKS.
3.	Deploy: Executes helm upgrade --install with --atomic for safe, atomic deployments.
4.	Test: Optional post-deploy smoke tests (e.g., curl ALB endpoint).
Trigger on pushes to main or via manual dispatch. For secrets, use GitHub Secrets (e.g., AWS_ACCESS_KEY_ID).

🔒 Security and Best Practices
•	Secrets Management: Use placeholders in values; integrate AWS Secrets Manager via IRSA or External Secrets Operator for production.
•	Scanning: Run trivy fs . for vulnerabilities and kube-bench on the cluster.
•	Cost Optimization: ALB target-type: ip reduces latency/costs; HPA prevents over-provisioning.
•	Extensibility: Add initContainers/sidecars in deployment.yaml for complex apps.
•	Monitoring: Enable Prometheus annotations in templates for metrics collection.

