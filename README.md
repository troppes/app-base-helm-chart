# app-base

A Helm chart providing shared templates for simple app deployments.

## Usage

**`Chart.yaml`** — declare the dependency:

```yaml
apiVersion: v2
name: myapp
version: 0.1.0
dependencies:
  - name: app-base
    version: "0.1.0"
    repository: "oci://ghcr.io/troppes/helm-charts"
```

**`values.yaml`** — all config goes under the `app-base:` key:

```yaml
app-base:
  nameOverride: myapp  # prevents resources being named "myapp-app-base"

  image:
    repository: myrepo/myapp
    tag: "1.0.0"

  service:
    port: 8080
```

Then run:

```bash
helm dependency update
```

## Values

### Image

| Key | Default | Description |
|---|---|---|
| `image.repository` | `""` | Container image repository |
| `image.tag` | `""` | Image tag (falls back to `Chart.appVersion`) |
| `image.pullPolicy` | `Always` | Image pull policy |
| `imagePullSecrets` | `[]` | Image pull secrets |

### Deployment

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of replicas |
| `updateStrategy.type` | `RollingUpdate` | Deployment update strategy (`RollingUpdate` or `Recreate`) |
| `command` | `[]` | Override container entrypoint |
| `args` | `[]` | Override container arguments |
| `nameOverride` | `""` | Override chart name |
| `fullnameOverride` | `""` | Override full release name |
| `podAnnotations` | `{}` | Pod annotations |
| `podLabels` | `{}` | Extra pod labels |

### Service Account

| Key | Default | Description |
|---|---|---|
| `serviceAccount.create` | `false` | Create a ServiceAccount |
| `serviceAccount.name` | `""` | Name (defaults to release fullname) |
| `serviceAccount.annotations` | `{}` | ServiceAccount annotations |

### Security

| Key | Default | Description |
|---|---|---|
| `podSecurityContext` | `{}` | Pod-level security context (`fsGroup`, `runAsNonRoot`, etc.) |
| `securityContext.allowPrivilegeEscalation` | `false` | Prevent privilege escalation |

Additional container security context fields (`runAsUser`, `readOnlyRootFilesystem`, `capabilities.drop`) can be set under `securityContext`

### Resources

| Key | Default | Description |
|---|---|---|
| `resources.requests.cpu` | `50m` | CPU request |
| `resources.requests.memory` | `128Mi` | Memory request |
| `resources.limits.cpu` | `500m` | CPU limit |
| `resources.limits.memory` | `512Mi` | Memory limit |

### Service

| Key | Default | Description |
|---|---|---|
| `service.type` | `ClusterIP` | Service type |
| `service.port` | `80` | Service (and container) port |
| `service.containerPort` | `""` | Override if container port differs from service port |
| `service.annotations` | `{}` | Service annotations |

### Ingress

| Key | Default | Description |
|---|---|---|
| `ingress.enabled` | `false` | Enable ingress |
| `ingress.className` | `""` | Ingress class name |
| `ingress.annotations` | `{}` | Ingress annotations |
| `ingress.hosts` | `[]` | Ingress hosts |
| `ingress.tls` | `[]` | TLS configuration |

### Environment & Config

| Key | Default | Description |
|---|---|---|
| `env` | `[]` | Environment variables (`name`/`value` or `valueFrom`) |
| `envFrom` | `[]` | ConfigMap or Secret refs via `envFrom` |

### Volumes

| Key | Default | Description |
|---|---|---|
| `volumes` | `[]` | Pod volumes |
| `volumeMounts` | `[]` | Main container volume mounts |

### Probes

All probe types are supported (`httpGet`, `exec`, `tcpSocket`, `grpc`).

| Key | Default | Description |
|---|---|---|
| `probes.liveness.enabled` | `false` | Enable liveness probe |
| `probes.readiness.enabled` | `false` | Enable readiness probe |
| `probes.startup.enabled` | `false` | Enable startup probe |

### Containers

| Key | Default | Description |
|---|---|---|
| `initContainers` | `[]` | Init containers (run before the main container) |
| `extraContainers` | `[]` | Sidecar containers (e.g. VPN, exporters) |

## Examples

All values go under the `app-base:` key in your child chart's `values.yaml`.

### App with environment variables

```yaml
app-base:
  nameOverride: myapp

  image:
    repository: myrepo/myapp
    tag: "1.0.0"

  service:
    port: 8080

  env:
    - name: TZ
      value: "Europe/London"
    - name: MY_SECRET
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: value
```

### App with separate container and service ports

```yaml
app-base:
  service:
    port: 80
    containerPort: 8080
```

### App with a VPN sidecar

```yaml
app-base:
  extraContainers:
    - name: vpn
      image: ghcr.io/qdm12/gluetun:latest
      securityContext:
        capabilities:
          add: [NET_ADMIN]
      envFrom:
        - secretRef:
            name: vpn-credentials

  volumes:
    - name: vpn-config
      secret:
        secretName: vpn-config
```

### App with init container

```yaml
app-base:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
```

### App with stricter security

```yaml
app-base:
  podSecurityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000

  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop: [ALL]
```

### App with ingress and TLS

```yaml
app-base:
  ingress:
    enabled: true
    className: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt
    hosts:
      - host: myapp.example.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: myapp-tls
        hosts:
          - myapp.example.com
```

### Stateful single-pod app

```yaml
app-base:
  updateStrategy:
    type: Recreate

  podSecurityContext:
    fsGroup: 1000

  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: myapp-data

  volumeMounts:
    - name: data
      mountPath: /data
```
