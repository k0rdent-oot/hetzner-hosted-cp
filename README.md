# Hetzner Hosted Control Plane

Kubernetes management cluster with K0rdent (KCM) and Hetzner hosted control plane template.

## Quick Start

### 1. Install k0s

```bash
sudo wget https://github.com/k0sproject/k0s/releases/download/v1.34.4%2Bk0s.0/k0s-v1.34.4+k0s.0-amd64 -O /usr/local/bin/k0s
sudo chmod +x /usr/local/bin/k0s

k0s install controller \
    --enable-dynamic-config \
    --disable-components=konnectivity-server \
    --enable-worker \
    --no-taints \
    --kubelet-root-dir=/var/lib/kubelet \
    --kubelet-extra-args="--cloud-provider=external" \
    --verbose

sudo systemctl enable --now k0scontroller
ln -s /usr/local/bin/k0s /usr/local/bin/kubectl
```

### 2. Install Helm

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### 3. Setup Environment

```bash
export KUBECONFIG=/var/lib/k0s/pki/admin.conf
```

### 4. Install K0rdent (KCM)

```bash
helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 1.7.0 -n kcm-system --create-namespace \
  --set regional.telemetry.mode=disabled \
  --set regional.velero.enabled=false

kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: oot-repo
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/managed: "true"
spec:
  interval: 10m0s
  provider: generic
  type: oci
  url: oci://ghcr.io/k0rdent-oot/oot/charts
---
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ProviderTemplate
metadata:
  name: cluster-api-provider-k0sproject-k0smotron-9999-42-2
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: kcm
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-k0sproject-k0smotron
      version: "9999.42.2"
      interval: 10m0s
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: oot-repo
EOF

k0s kubectl patch mgmt kcm --type=merge -p '{"spec":{"providers":[{"name":"cluster-api-provider-k0sproject-k0smotron","template":"cluster-api-provider-k0sproject-k0smotron-9999-42-2","config":{"version":"v1.10.3"}},{"name":"projectsveltos"}]}}'
```

### 5. Install Hetzner CCM

The Hetzner Cloud Controller Manager runs in the management cluster and allocates a Hetzner load balancer for the HAProxy ingress service. This LB IP is used as the DNS target for all child cluster ingress hostnames.

Replace `YOUR_HETZNER_API_TOKEN` with your actual token:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: hcloud
  namespace: kube-system
stringData:
  token: "YOUR_HETZNER_API_TOKEN"
EOF

helm repo add hcloud https://charts.hetzner.cloud
helm install hcloud-ccm hcloud/hcloud-cloud-controller-manager \
  --namespace kube-system \
  --version 1.30.1
```

### 6. Install HAProxy Ingress

The `load-balancer.hetzner.cloud/location` annotation is required — the CCM refuses to create a LB without knowing the datacenter. Replace `nbg1` with your server's location (`fsn1`, `nbg1`, `hel1`, etc.):

```bash
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
  --namespace ingress-controller \
  --create-namespace \
  --set controller.ingressClassResource.enabled=true \
  --set controller.service.type=LoadBalancer \
  --set-json 'controller.service.annotations={"load-balancer.hetzner.cloud/location":"nbg1"}'
```

Wait for the LoadBalancer IP to be assigned by the CCM:

```bash
kubectl get svc -n ingress-controller haproxy-ingress -w
```

### 7. Install Hetzner Provider

Install the provider chart (creates ProviderInterface and ClusterRole for KCM), create the ProviderTemplate so KCM knows which chart version to deploy, and register the provider with the management cluster:

```bash
helm install cluster-api-provider-hetzner \
  oci://ghcr.io/k0rdent-oot/oot/charts/cluster-api-provider-hetzner \
  --version 1.2.1 \
  --namespace kcm-system

kubectl apply -f - <<EOF
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ProviderTemplate
metadata:
  name: cluster-api-provider-hetzner
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: kcm
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-hetzner
      version: "1.2.1"
      interval: 10m0s
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: oot-repo
EOF

k0s kubectl patch mgmt kcm --type=merge -p '{"spec":{"providers":[{"name":"cluster-api-provider-k0sproject-k0smotron","template":"cluster-api-provider-k0sproject-k0smotron-9999-42-2","config":{"version":"v1.10.3"}},{"name":"projectsveltos"},{"name":"cluster-api-provider-hetzner","template":"cluster-api-provider-hetzner"}]}}'
```

### 8. Register ClusterTemplate

```bash
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: k0rdent-oot-charts
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/managed: "true"
spec:
  interval: 10m0s
  provider: generic
  type: oci
  url: oci://ghcr.io/k0rdent-oot/charts
EOF

kubectl apply -f - <<EOF
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterTemplate
metadata:
  name: hetzner-hosted-cp
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: kcm
spec:
  helm:
    chartSpec:
      chart: hetzner-hosted-cp
      version: "0.0.1"
      interval: 10m0s
      sourceRef:
        kind: HelmRepository
        name: k0rdent-oot-charts
EOF
```

### 9. Configure Credentials

Replace `YOUR_HETZNER_API_TOKEN` with your actual token:

```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: hetzner-config
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
stringData:
  hcloud: "YOUR_HETZNER_API_TOKEN"
---
apiVersion: k0rdent.mirantis.com/v1beta1
kind: Credential
metadata:
  name: hetzner-cluster-identity-cred
  namespace: kcm-system
spec:
  description: Hetzner credentials
  identityRef:
    apiVersion: v1
    kind: Secret
    name: hetzner-config
    namespace: kcm-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hetzner-config-resource-template
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
  annotations:
    projectsveltos.io/template: "true"
EOF
```

#### Verify

```bash
kubectl get clustertemplate -n kcm-system
```

Should show `VALID: true`

### 10. Install OpenEBS (Local Path Provisioner)

k0smotron requires persistent storage for control plane pods. Install OpenEBS with only the LocalPath provisioner enabled and set it as the default storage class:

```bash
helm repo add openebs https://openebs.github.io/charts
helm install openebs openebs/openebs \
  --namespace openebs \
  --create-namespace \
  --set mayastor.enabled=false \
  --set jiva.enabled=false \
  --set cstor.enabled=false \
  --set zfs-localpv.enabled=false \
  --set lvm-localpv.enabled=false \
  --set nfs-provisioner.enabled=false \
  --set localprovisioner.enabled=true \
  --set localprovisioner.hostpathClass.isDefaultClass=true
```

### 11. Deploy a Cluster

Get the external IP of the HAProxy ingress load balancer allocated in step 6:

```bash
LB_IP=$(kubectl get svc haproxy-ingress -n ingress-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $LB_IP
```

Hostnames use [sslip.io](https://sslip.io) — any name with an embedded IP resolves to that IP, so no DNS setup is required. Multiple clusters share the same LB and are differentiated by SNI hostname via the cluster-name subdomain (e.g. `api.my-cluster.<LB_IP>.sslip.io`).

```bash
kubectl apply -f - <<EOF
---
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: my-cluster
  namespace: kcm-system
spec:
  template: hetzner-hosted-cp
  credential: hetzner-cluster-identity-cred
  config:
    region: "nbg1"
    workersNumber: 1
    sshKeyNames:
      - my-ssh-key
    tokenRef:
      name: hetzner-config
      key: hcloud
    controlPlane:
      etcd:
        persistence:
          autoDeletePVCs: true
      ingress:
        apiHost: "api.my-cluster.${LB_IP}.sslip.io"
        konnectivityHost: "konnectivity.my-cluster.${LB_IP}.sslip.io"
    worker:
      type: cpx32
      image: ubuntu-24.04
    k0s:
      version: v1.34.4+k0s.0
      network:
        provider: calico
EOF
```

### 12. Monitor and Access

```bash
# Watch provisioning progress
kubectl get clusterdeployment my-cluster -n kcm-system -w

# Get kubeconfig once the cluster is Ready
kubectl get secret my-cluster-kubeconfig -n kcm-system -o jsonpath='{.data.value}' | base64 -d > my-cluster.kubeconfig

# Verify nodes
kubectl --kubeconfig=my-cluster.kubeconfig get nodes -o wide
```
