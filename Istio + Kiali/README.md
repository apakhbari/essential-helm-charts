Integrating Istio and Kiali onto your air-gapped RKE2 cluster with Canal follows the exact same logical blueprint you used for the Prometheus stack: mirror the images, extract the charts, and override the default endpoints using your internal Docker registry (`registry.eniac-tech.com`) and your `internal-registry-secret`.

Since your architecture runs solely on 3 master nodes, there are specific configuration tweaks needed to ensure Istio handles Canal's MTU/networking gracefully.

Here is your step-by-step air-gapped implementation guide.

---

### **Phase 1: Download Charts & Images (Internet Connected)**

Run these commands on your internet-connected machine to grab the exact components needed.

**1. Download the Helm Charts**

```bash
# Add repositories
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo add kiali https://kiali.org/helm-charts
helm repo update

# Pull Istio Base (CRDs), Istiod (Control Plane), and Kiali Server
helm pull istio/base --untar
helm pull istio/istiod --untar
helm pull kiali/kiali-server --untar

```

#### Images
```
quay.io/kiali/kiali:v2.27.0
istio/proxyv2:1.30.1
istio/pilot:1.30.1
```

**2. Extract and Sync Container Images**
Identify the images used across these charts. For standard Istio + Kiali deployments, you will need to pull, retag, and push these core images to `registry.eniac-tech.com`:

* `docker.io/istio/pilot:<version>` (The main Istiod control plane)
* `docker.io/istio/proxyv2:<version>` (The Envoy proxy sidecar injected into your pods)
* `quay.io/kiali/kiali:<version>` (The Kiali visualization engine)

*Script example to sync to your repository:*

```bash
REGISTRY="registry.eniac-tech.com"
IMAGES=(
  "docker.io/istio/pilot:1.24.0"   # Replace with matching version from downloaded chart
  "docker.io/istio/proxyv2:1.24.0"
  "quay.io/kiali/kiali:v2.0.0"
)

for img in "${IMAGES[@]}"; do
  docker pull "$img"
  # Strip registry prefix
  NEW_NAME=$(echo "$img" | sed -E 's|^[^/]+/(.*)$|\1|')
  docker tag "$img" "$REGISTRY/$NEW_NAME"
  docker push "$REGISTRY/$NEW_NAME"
done

```

Transfer the three untarred chart directories to your primary master node (`k8s-core-master01`).

---

### **Phase 2: Deploy Istio Base & Istiod**

Log back into `k8s-core-master01`. We will install Istio inside the `istio-system` namespace.

**1. Ensure your Registry Secret exists**
If you haven't already done so in the `istio-system` namespace, make sure the secret is present so Kubernetes can authenticate with your domain:

```bash
kubectl create namespace istio-system
kubectl create secret docker-registry internal-registry-secret \
  --docker-server=registry.eniac-tech.com \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  -n istio-system

```

**2. Install Istio Base (CRDs)**
The base chart sets up the foundational cluster roles and CRDs. No image overrides are needed here.

```bash
helm install istio-base ./base -n istio-system

```

**3. Create `istiod-values.yaml**`
Because Canal uses standard virtual networking overlays (VXLAN), we must configure Istiod to route explicitly using standard parameters while ensuring it targets your internal registry.

Create `istiod-values.yaml`:

```yaml
global:
  hub: registry.eniac-tech.com/istio
  imagePullSecrets:
    - internal-registry-secret

meshConfig:
  # Crucial for Canal troubleshooting: standardizes traffic discovery metrics
  enablePrometheusMerge: true 
  accessLogFile: /dev/stdout

pilot:
  image: pilot
  autoscaleEnabled: false # Keeps it constrained cleanly on your 3 master nodes

sidecarInjectorWebhook:
  objectSelector:
    enabled: true

```

**4. Deploy Istiod**

```bash
helm install istiod ./istiod -n istio-system -f istiod-values.yaml

```

Verify the control plane is healthy before moving on:

```bash
kubectl get pods -n istio-system

```

---

### **Phase 3: Deploy Kiali (The Topology Dashboard)**

Kiali reads topology information directly from your cluster metrics. It works best if it knows how to read data from the Prometheus instance you installed earlier.

**1. Create `kiali-values.yaml**`

```yaml
image:
  repository: registry.eniac-tech.com/kiali/kiali
  tag: v2.0.0 # Match your synced version
  imagePullSecrets:
    - internal-registry-secret

external_services:
  prometheus:
    # Points Kiali directly to your kube-prometheus-stack service instance
    url: "http://prometheus-stack-kube-prom-prometheus.monitoring.svc.cluster.local:9090"
  istio:
    root_namespace: "istio-system"

auth:
  strategy: "anonymous" # Ideal for closed, highly secure air-gapped testing setups

```

**2. Deploy Kiali**

```bash
helm install kiali-server ./kiali-server -n istio-system -f kiali-values.yaml

```

**3. Deploy Kiali's NodePort**
```bash
kubectl apply -f node-port.yaml
```

---

### **Phase 4: Test and Troubleshoot Traffic Interception**

To see the topology map working on your cluster, you must instruct the Istio control plane to actively inject sidecars into a target namespace.

**1. Enable Auto-Injection**
Create a test namespace (or use an existing application namespace) and apply the injection label:

```bash
kubectl create namespace app-test
kubectl label namespace app-test istio-injection=enabled

kubectl label namespace monitoring istio-injection=enabled
```

**2. Verify Pod Sidecars**
Deploy any application deployment into that namespace. When you run `kubectl get pods -n app-test`, you should notice the container count reads **`2/2`** instead of `1/1`.

The second container is your `istio-proxy` running alongside your application. If it fails with an `ImagePullBackOff`, running `kubectl describe pod` will show if it successfully found your `internal-registry-secret`.

**3. View the Dashboard**
Port-forward the Kiali UI dashboard service to your local machine to check the live topology graphs:

```bash
kubectl port-forward svc/kiali 20001:20001 -n istio-system --address 0.0.0.0

```

Open up your local web browser and head to `http://<your-master-node-ip>:20001`. As your applications transfer standard HTTP or TCP packets across Canal, Kiali will map out the node lines and health status metrics instantly on your screen.