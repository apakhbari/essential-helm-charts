# RabbitMQ Helm Charts

Good news/bad news on your plan: the good news is your architecture idea (new namespace, exporter, permanent NodePort, ServiceMonitor for the existing Prometheus Operator) is exactly right. The bad news is **Bitnami's free RabbitMQ image is gone** — as of late 2025 Bitnami stopped publishing versioned images on `docker.io/bitnami`; they moved to a paid "Bitnami Secure Images" subscription (`bitnamisecure`) or an unsupported, frozen `bitnamilegacy` mirror. So don't build this on the Bitnami helm chart — it'll break on pull or need a paid subscription, and it's a bad foundation for an air-gapped cluster anyway (extra layers of bitnami-shell/init-container images).

Instead, for a single self-contained RabbitMQ+UI+exporter in a new namespace, the cleanest and most air-gap-friendly approach is:

- **Official RabbitMQ image** `rabbitmq:<version>-management` (one image, UI plugin already baked in)
- **`kbudde/rabbitmq_exporter`** as a sidecar — same exporter you already run in `cattle-monitoring-system`, so your existing Grafana dashboard/queries work unchanged against the new instance too
- A small hand-rolled Helm chart (2 images total, no operator, no CRDs beyond the `ServiceMonitor` your Prometheus Operator already understands)

If you'd rather use the official upstream path going forward, the RabbitMQ team's own recommended k8s deployment is the **RabbitMQ Cluster Operator** (`rabbitmqoperator/cluster-operator`) — more robust for HA, but heavier (extra operator + CRDs) for "just give me one more instance." I'll give you the simple chart route below; I'll flag where the Operator differs.

---

## 0. Decide naming (adjust to your convention)

```
NAMESPACE=rabbitmq-billing          # new namespace, different from your first instance
RELEASE=rabbitmq-billing
REGISTRY=registry.eniac-tech.com
RABBITMQ_TAG=3.13.7-management      # pick current stable at pull time
EXPORTER_TAG=v1.0.0-RC24            # check kbudde/rabbitmq_exporter tags for latest
```

---

## 1. On an internet-connected machine: mirror the two images

```bash
docker pull rabbitmq:${RABBITMQ_TAG}
docker pull kbudde/rabbitmq_exporter:${EXPORTER_TAG}

docker tag rabbitmq:${RABBITMQ_TAG} ${REGISTRY}/library/rabbitmq:${RABBITMQ_TAG}
docker tag kbudde/rabbitmq_exporter:${EXPORTER_TAG} ${REGISTRY}/kbudde/rabbitmq_exporter:${EXPORTER_TAG}

docker login ${REGISTRY}
docker push ${REGISTRY}/library/rabbitmq:${RABBITMQ_TAG}
docker push ${REGISTRY}/kbudde/rabbitmq_exporter:${EXPORTER_TAG}
```

Transfer nothing else — that's the entire image list for this design (no bitnami-shell, no volume-permissions init container, no operator images).

---

## 2. Build the Helm chart

Create this structure on the internet-connected machine (or directly on the airgapped bastion, it's plain YAML):

```
rabbitmq-billing/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── secret.yaml
    ├── configmap.yaml
    ├── service-headless.yaml
    ├── service-nodeport.yaml
    ├── servicemonitor.yaml
    └── statefulset.yaml
```

**Chart.yaml**
```yaml
apiVersion: v2
name: rabbitmq-billing
description: Standalone RabbitMQ with management UI and Prometheus exporter
version: 0.1.0
appVersion: "3.13.7"
```

**values.yaml**
```yaml
image:
  repository: registry.eniac-tech.com/library/rabbitmq
  tag: 3.13.7-management
  pullPolicy: IfNotPresent

exporterImage:
  repository: registry.eniac-tech.com/kbudde/rabbitmq_exporter
  tag: v1.0.0-RC24
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: eniac-registry

auth:
  username: admin
  password: "CHANGE-ME-STRONG-PASSWORD"
  erlangCookie: "CHANGE-ME-RANDOM-COOKIE-STRING"

persistence:
  enabled: true
  size: 8Gi
  storageClassName: ""     # set to your cluster's StorageClass, or "" for default

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

service:
  nodePort:
    amqp: 30672
    management: 30673
  metricsNodePort: 30674   # permanent, fixed NodePort for the exporter's /metrics

serviceMonitor:
  enabled: true
  interval: 30s
  # only needed if your Prometheus CR restricts serviceMonitorSelector by label
  additionalLabels: {}
```

**templates/secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-auth
type: Opaque
stringData:
  username: {{ .Values.auth.username | quote }}
  password: {{ .Values.auth.password | quote }}
  erlang-cookie: {{ .Values.auth.erlangCookie | quote }}
```

**templates/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  rabbitmq.conf: |
    loopback_users.guest = false
    management.tcp.port = 15672
  enabled_plugins: |
    [rabbitmq_management,rabbitmq_peer_discovery_k8s,rabbitmq_prometheus].
```

**templates/service-headless.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  clusterIP: None
  selector:
    app: {{ .Release.Name }}
  ports:
    - name: amqp
      port: 5672
    - name: management
      port: 15672
    - name: metrics
      port: 9419
```

**templates/service-nodeport.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-nodeport
  labels:
    app: {{ .Release.Name }}
spec:
  type: NodePort
  selector:
    app: {{ .Release.Name }}
  ports:
    - name: amqp
      port: 5672
      nodePort: {{ .Values.service.nodePort.amqp }}
    - name: management
      port: 15672
      nodePort: {{ .Values.service.nodePort.management }}
    - name: metrics
      port: 9419
      nodePort: {{ .Values.service.metricsNodePort }}
```

**templates/servicemonitor.yaml**
```yaml
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- toYaml .Values.serviceMonitor.additionalLabels | nindent 4 }}
spec:
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  namespaceSelector:
    matchNames:
      - {{ .Release.Namespace }}
  endpoints:
    - port: metrics
      interval: {{ .Values.serviceMonitor.interval }}
{{- end }}
```

**templates/statefulset.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ .Release.Name }}
spec:
  serviceName: {{ .Release.Name }}
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      imagePullSecrets:
        {{- toYaml .Values.imagePullSecrets | nindent 8 }}
      containers:
        - name: rabbitmq
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: RABBITMQ_DEFAULT_USER
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-auth
                  key: username
            - name: RABBITMQ_DEFAULT_PASS
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-auth
                  key: password
            - name: RABBITMQ_ERLANG_COOKIE
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-auth
                  key: erlang-cookie
          ports:
            - containerPort: 5672
              name: amqp
            - containerPort: 15672
              name: management
            - containerPort: 15692
              name: prom-native
          volumeMounts:
            - name: data
              mountPath: /var/lib/rabbitmq
            - name: config
              mountPath: /etc/rabbitmq/rabbitmq.conf
              subPath: rabbitmq.conf
            - name: config
              mountPath: /etc/rabbitmq/enabled_plugins
              subPath: enabled_plugins
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
        - name: exporter
          image: "{{ .Values.exporterImage.repository }}:{{ .Values.exporterImage.tag }}"
          imagePullPolicy: {{ .Values.exporterImage.pullPolicy }}
          env:
            - name: RABBIT_URL
              value: "http://localhost:15672"
            - name: RABBIT_USER
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-auth
                  key: username
            - name: RABBIT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-auth
                  key: password
            - name: PUBLISH_PORT
              value: "9419"
          ports:
            - containerPort: 9419
              name: metrics
      volumes:
        - name: config
          configMap:
            name: {{ .Release.Name }}-config
  {{- if .Values.persistence.enabled }}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        {{- if .Values.persistence.storageClassName }}
        storageClassName: {{ .Values.persistence.storageClassName }}
        {{- end }}
        resources:
          requests:
            storage: {{ .Values.persistence.size }}
  {{- end }}
```

Note: I kept the exporter (port 9419, same as your existing one) as the primary metrics source for dashboard consistency, but also enabled RabbitMQ's own native Prometheus plugin (`rabbitmq_prometheus`, port 15692) in `enabled_plugins` — free extra metrics if you ever want them, costs nothing since it's part of the official image.

---

## 3. Transfer & load on the air-gapped cluster

Copy the `rabbitmq-billing/` folder to your bastion via your usual offline transfer method, then:

```bash
kubectl create namespace rabbitmq-billing

kubectl create secret docker-registry eniac-registry \
  --docker-server=registry.eniac-tech.com \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n rabbitmq-billing
```

Edit `values.yaml` (password, erlang cookie, storageClassName, NodePort numbers if they conflict with anything already in use — check with `kubectl get svc -A -o wide | grep -E '30672|30673|30674'` first).

---

## 4. Install

```bash
helm install rabbitmq-billing ./rabbitmq-billing -n rabbitmq-billing
```

Verify:
```bash
kubectl get pods,svc,pvc -n rabbitmq-billing
kubectl get servicemonitor -n rabbitmq-billing
```

---

## 5. Confirm Prometheus (rancher-monitoring) will actually scrape it

Before assuming the ServiceMonitor is picked up, check what your Prometheus CR is configured to watch:

```bash
kubectl get prometheus rancher-monitoring-prometheus -n cattle-monitoring-system \
  -o jsonpath='{.spec.serviceMonitorSelector}{"\n"}{.spec.serviceMonitorNamespaceSelector}{"\n"}'
```

- If both come back as `{}` (empty), it watches **all** ServiceMonitors in **all** namespaces regardless of labels — your new one in `rabbitmq-billing` will be picked up automatically.
- If either has `matchLabels`, add those same labels under `serviceMonitor.additionalLabels` in `values.yaml` and `helm upgrade`.

Then confirm the target is actually being scraped:
```bash
kubectl port-forward -n cattle-monitoring-system svc/rancher-monitoring-prometheus 9090:9090
```
and check `http://localhost:9090/targets` for a job matching `rabbitmq-billing`.

---

## 6. Access

- **Management UI:** `http://<any-node-ip>:30673` (login with the `auth.username`/`auth.password` you set)
- **AMQP:** `<node-ip>:30672`
- **Metrics (verify manually):** `curl http://<node-ip>:30674/metrics`

## 7. Grafana

Since you're using the same `kbudde/rabbitmq_exporter`, your existing RabbitMQ dashboard (metric names like `rabbitmq_queue_messages`, `rabbitmq_channels`, etc.) will work as-is against the new instance — just make sure the dashboard's variable/query for `instance` or `job` lets you pick between the two exporters (e.g. a Grafana template variable on `job` or `namespace` label) rather than being hardcoded to the old one.

---

A few things worth flagging:
- **NodePorts are cluster-wide unique** — double check 30672/30673/30674 aren't already claimed by anything else before installing.
- If you eventually want RabbitMQ clustering/HA rather than a single pod, look at the **RabbitMQ Cluster Operator** instead of scaling this StatefulSet's replicas — a single StatefulSet with `replicas > 1` won't auto-cluster correctly without the peer-discovery plugin wired to a proper k8s RBAC service account, which I skipped here since you're mirroring a "different namespace, same simple pattern" setup.
- Rotate `auth.password` and `erlangCookie` out of `values.yaml` into a sealed-secret or vault-managed secret before this goes anywhere near production, if it isn't already.

Want me to also add the peer-discovery RBAC (ServiceAccount/Role/RoleBinding) so this chart is ready to scale to a real 3-node cluster later, or is single-node fine for this instance?