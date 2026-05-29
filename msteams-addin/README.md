# Sendent MS Teams Add-in Helm Chart

Deploy the Sendent Microsoft Teams Add-in on Kubernetes.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- A running Nextcloud instance
- A domain name pointing to your cluster (e.g., `teams.example.com`)

## Installation

Create a file called `my-values.yaml` with your configuration:

```yaml
config:
  BASE_URL: "https://teams.example.com"
  MSAPP_TYPE: "your-app-type"
  MSAPP_ID: "your-azure-app-id"
  MSAPP_TENANT_ID: "your-azure-tenant-id"
  DEFAULT_NEXTCLOUD_URL: "https://nextcloud.example.com"

secret:
  MSAPP_PASSWORD: "your-secure-password"

```

Install:

```bash
helm install sendent-msteams ./msteams-addin -f my-values.yaml

```

Verify the deployment is running:

```bash
kubectl get pods -l app.kubernetes.io/name=sendent-msteams

```

You should see your pod(s) in `Running` status.

The chart creates a `ClusterIP` service on port 4200. See [Exposing the add-in](#exposing-the-add-in) for how to make it accessible externally.

## Configuration

### Application Parameters

| Parameter | Description | Required | Default |
| --- | --- | --- | --- |
| `config.BASE_URL` | Public URL where the add-in will be accessible | Yes | `"https://teams.yourdomain.com"` |
| `config.DEFAULT_NEXTCLOUD_URL` | Default Nextcloud server URL returned to the add-in | No | `"https://nextcloud.yourdomain.com"` |
| `config.MSAPP_TYPE` | Azure App type: `MultiTenant`, `SingleTenant`, or `UserAssignedMSI`. Leave empty to default to `MultiTenant` | No | `""` |
| `config.MSAPP_ID` | Azure App (client) ID used for bot authentication and the on-behalf-of token exchange | Yes | `"your-app-id"` |
| `config.MSAPP_TENANT_ID` | Azure Tenant ID (required for `SingleTenant` apps; ignored otherwise) | Conditional | `"your-tenant-id"` |
| `config.PROXY_PLACEHOLDER_URL` | Fallback target for the `/proxy/*` request proxy (the per-request `sendent-apiurl` header normally overrides this) | No | `"https://placeholder.sendent.dev"` |
| `secret.MSAPP_PASSWORD` | Azure App client secret. Ignored when `secret.existingSecret` is set | Conditional | `""` |
| `secret.existingSecret` | Name of an externally-managed `Secret` to read the client secret from instead of creating one. See [Managing the client secret externally](#managing-the-client-secret-externally) | No | `""` |
| `secret.existingSecretKey` | Key within `secret.existingSecret` holding the client secret | No | `"MSAPP_PASSWORD"` |

### Deployment Parameters

| Parameter | Description | Default |
| --- | --- | --- |
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Container image repository | `rg.nl-ams.scw.cloud/sendent-public/sendent-msteams` |
| `image.tag` | Container image tag | `"latest"` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `4200` |
| `ingress.enabled` | Enable ingress resource | `false` |
| `resources` | CPU/memory resource requests and limits | `{}` |

The add-in is stateless, so it can be scaled horizontally by increasing `replicaCount`. For production deployments, it is recommended to set resource requests and limits:

```yaml
replicaCount: 3

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

```

## Managing the client secret externally

By default, the chart creates a Kubernetes `Secret` from the value of `secret.MSAPP_PASSWORD`. This is convenient for getting started but ties the secret to your values file. For production, you'll usually want to manage the secret out-of-band and point the chart at it via `secret.existingSecret`.

Create the Secret yourself, by whatever means you prefer. For example:

```bash
kubectl create secret generic msteams-azure \
  --from-literal=MSAPP_PASSWORD='your-real-client-secret'
```

Then reference it from your values file and leave `secret.MSAPP_PASSWORD` unset:

```yaml
secret:
  existingSecret: msteams-azure
  # existingSecretKey defaults to MSAPP_PASSWORD; override if your Secret uses a different key
  # existingSecretKey: client-secret
```

The same pattern works with any tool that produces a regular `Secret` resource:

- **[External Secrets Operator](https://external-secrets.io/)**: sync the value from Vault, AWS Secrets Manager, 1Password, etc. via an `ExternalSecret` CR that targets a Secret named `msteams-azure`.
- **[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)**: encrypt a `Secret` with `kubeseal` and commit the resulting `SealedSecret` to git; the in-cluster controller decrypts it into the regular Secret the chart will read.

## Exposing the add-in

### Option A: Ingress (recommended)

The chart can create an Ingress resource to expose the add-in externally. You will need:

* An ingress controller (e.g. [ingress-nginx](https://kubernetes.github.io/ingress-nginx/))
* [cert-manager](https://cert-manager.io/) (for automatic TLS certificates)

Install them if they are not already present on your cluster:

```bash
# Ingress controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

```

Create a ClusterIssuer for Let's Encrypt:

```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx

```

```bash
kubectl apply -f cluster-issuer.yaml

```

Point your domain's A record at the ingress controller's external IP:

```bash
kubectl get svc -n ingress-nginx

```

Then add the following to your values file:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: teams.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: msteams-addin-tls
      hosts:
        - teams.example.com

```

Adjust `className`, `annotations`, `pathType`, and `tls` to match your cluster's ingress setup.

### Option B: Reverse proxy

If you prefer to manage routing outside of Kubernetes, keep the default `ClusterIP` service and point your existing reverse proxy (e.g., Nginx, Caddy) at the service. You can use `kubectl port-forward` to expose the service locally:

```bash
kubectl port-forward svc/sendent-msteams 4200:4200

```

Then configure your reverse proxy to forward traffic from your domain to `localhost:4200`.

## Upgrading

```bash
helm upgrade sendent-msteams ./msteams-addin -f my-values.yaml

```

## Uninstalling

```bash
helm uninstall sendent-msteams

```

## Troubleshooting

```bash
# Pod status
kubectl get pods -l app.kubernetes.io/name=sendent-msteams

# Pod events and details
kubectl describe pod -l app.kubernetes.io/name=sendent-msteams

# Application logs
kubectl logs -l app.kubernetes.io/name=sendent-msteams

```

If your pods are in `CrashLoopBackOff`, check the logs for configuration errors. The most common issues are missing `BASE_URL` or authentication configuration parameters.