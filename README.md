# muster

cluster bootstrapping


## k3s examples

Write the following manifests to the k3s provisioning directory:
```
/var/lib/rancher/k3s/server/manifests/
```

This can be automated via `cloud-init` or Rancher-Desktop, as detailed in the [Methods](#methods) section.

### Manifests

#### Crossplane

```yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: crossplane
  namespace: kube-system
spec:
  repo: https://charts.crossplane.io/stable
  chart: crossplane
  targetNamespace: crossplane-system
  createNamespace: true
```

#### ArgoCD

```yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: argocd
  namespace: kube-system
spec:
  repo: https://argoproj.github.io/argo-helm
  chart: argo-cd
  targetNamespace: argocd
  createNamespace: true
  valuesContent: |-
    global:
      domain: argocd.local
    crds:
      install: true
    configs:
      cm:
        users.anonymous.enabled: true
      rbac:
        policy.default: role:readonly
      params:
        server.insecure: true
```

#### Private Configuration

Alter this example as required

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: observability
---
apiVersion: v1
kind: Secret
metadata:
  name: telegraf-env
  namespace: observability
type: Opaque
stringData:
  LOCATION_1_NAME: "New York"
  LOCATION_1_LATITUDE: "40.79"
  LOCATION_1_LONGITUDE: "-73.96"
  LOCATION_1_TIMEZONE: "America/New_York"

  LOCATION_2_NAME: "London"
  LOCATION_2_LATITUDE: "51.50"
  LOCATION_2_LONGITUDE: "0.00"
  LOCATION_2_TIMEZONE: "Europe/London"

  UTILITIES: "https://docs.google.com/spreadsheets/d/e/2PACX-asdf/pub?gid=0&single=true&output=csv"
  COMPOSITION: "https://docs.google.com/spreadsheets/d/e/2PACX-qwer/pub?gid=0&single=true&output=csv"

---
apiVersion: v1
kind: Namespace
metadata:
  name: netsec
```

Then append the Secret to the file:
```bash
openssl req \
  -quiet \
  -x509 \
  -new \
  -nodes \
  -newkey rsa:4096 \
  -keyout /tmp/tls.key \
  -out /tmp/tls.crt \
  -days 3650 \
  -subj "/O=Muster/OU=Tech/CN=Muster-CA" \
  -addext "basicConstraints = critical, CA:TRUE" \
  -addext "keyUsage = critical, digitalSignature, cRLSign, keyCertSign"
```
```bash
k \
  -n netsec \
  create secret tls muster-ca \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key \
  --dry-run=client -o yaml
```
Optionally install the CA Cert wherever required:
```bash
$ cat /tmp/tls.crt 
```

#### Application

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: all
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  source:
    path: projects
    repoURL: https://github.com/josephchapman/muster
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Methods

Write the above bootstrapping manifests to the provisioning directories (k3s examples below).

#### `cloud-init`

```yaml
write_files:
  - path: /var/lib/rancher/k3s/server/manifests/muster/crossplane.yaml
    owner: root:root
    permissions: '0600'
    content: |
      ---
      manifest
      ...
  - path: /var/lib/rancher/k3s/server/manifests/muster/argocd.yaml
    owner: root:root
    permissions: '0600'
    content: |
      ---
      manifest
      ...
  - path: /var/lib/rancher/k3s/server/manifests/muster/secret.yaml
    owner: root:root
    permissions: '0600'
    content: |
      ---
      manifest
      ...
  - path: /var/lib/rancher/k3s/server/manifests/muster/application.yaml
    owner: root:root
    permissions: '0600'
    content: |
      ---
      manifest
      ...
```

#### Rancher Desktop (WSL)

Create a `muster.start` file in `C:\Users\username\AppData\Local\rancher-desktop\provisioning>`:
```bash
set -eux
mkdir -p /var/lib/rancher/k3s/server/manifests/muster/

tee /var/lib/rancher/k3s/server/manifests/muster/crossplane.yaml > /dev/null \
<<'EOF'
---
manifest
...
EOF

tee /var/lib/rancher/k3s/server/manifests/muster/argocd.yaml > /dev/null \
<<'EOF'
---
manifest
...
EOF

tee /var/lib/rancher/k3s/server/manifests/muster/secret.yaml > /dev/null \
<<'EOF'
---
manifest
...
EOF

tee /var/lib/rancher/k3s/server/manifests/muster/application.yaml > /dev/null \
<<'EOF'
---
manifest
...
EOF
```
Ensure that the files are LF formatted (not CRLF)
