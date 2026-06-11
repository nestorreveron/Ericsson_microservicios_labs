# Advanced OpenTelemetry and Jaeger on AKS
## Istio Installed, but Security Enforcement Disabled

**Course:** Advanced Microservices, Kubernetes and Observability  
**Environment:** Azure Kubernetes Service (AKS), Istio already installed, OpenTelemetry Collector and Jaeger  
**Level:** Advanced — Senior Engineer / Architect / SRE  
**Estimated duration:** 4–6 hours  
**Lab type:** Complementary hands-on laboratory  
**Core tracing path:** Application or telemetry generator → OTLP/Zipkin → OpenTelemetry Collector → Jaeger

---

## 1. Lab purpose

The cluster already contains Istio and the existing e-commerce microservices. For this laboratory, Istio remains installed, but its security enforcement must not block the tracing exercises.

The lab therefore uses the following operating model:

- Istio remains installed in AKS.
- Existing Istio sidecars may remain attached to application pods.
- JWT enforcement is disabled for the lab application namespace.
- Istio authorization rules are disabled for the lab application namespace.
- Namespace-level mTLS is configured as `PERMISSIVE`.
- OpenTelemetry does not depend on Istio security policies.
- Application telemetry is sent directly to an OpenTelemetry Collector.
- Jaeger is used to store, search and visualize traces.
- Zipkin-format spans are accepted by the Collector to demonstrate protocol interoperability.
- Istio-generated Envoy spans are optional and are not required to complete the core lab.

The objective is not merely to deploy tools. The objective is to understand how distributed telemetry is generated, propagated, processed, sampled, exported and diagnosed.

---

## 2. Key message

The wrong question is:

> Do we need OpenTelemetry or Jaeger?

The better question is:

> What operational problem are we trying to solve?

In a distributed system, a single business request may cross several services and dependencies:

```text
Customer
   |
   v
Store Front
   |
   +--> Product Service --> Redis / MongoDB
   |
   +--> Order Service --> RabbitMQ
                           |
                           v
                    Makeline Service
```

Logs may show individual errors. Metrics may show system health. Distributed traces show the complete request path and the time spent in each operation.

---

## 3. Learning objectives

By the end of the lab, students will be able to:

1. Explain the OpenTelemetry architecture.
2. Distinguish API, SDK, instrumentation, Collector and backend responsibilities.
3. Configure an OpenTelemetry Collector gateway in AKS.
4. Configure OTLP and Zipkin receivers.
5. Use processors to control memory, enrich data and batch telemetry.
6. Export traces from the Collector to Jaeger.
7. Validate the telemetry pipeline independently from the application.
8. Build and inspect a multi-service parent/child trace.
9. Demonstrate broken context propagation.
10. Explain W3C Trace Context and Baggage.
11. Compare head sampling and tail sampling.
12. Troubleshoot receiver, processor and exporter failures.
13. Explain why setting OpenTelemetry environment variables does not automatically instrument an application.
14. Describe production deployment patterns for OpenTelemetry and Jaeger.
15. Operate Istio in the cluster without allowing its security policies to interfere with the lab.

---

## 4. Target architecture

### 4.1 Core architecture used in this lab

```text
Existing application pods or telemetry generators
              |
              | OTLP/gRPC :4317
              | OTLP/HTTP :4318
              | Zipkin    :9411
              v
+------------------------------------+
| OpenTelemetry Collector Gateway    |
|                                    |
| Receivers                          |
| - OTLP                             |
| - Zipkin                           |
|                                    |
| Processors                         |
| - memory_limiter                   |
| - k8sattributes                    |
| - resource                         |
| - batch                            |
| - tail_sampling (advanced section) |
|                                    |
| Exporters                          |
| - OTLP to Jaeger                   |
| - debug                            |
+------------------+-----------------+
                   |
                   | OTLP/gRPC
                   v
             +-----------+
             |  Jaeger   |
             |  UI/API   |
             +-----------+
```

### 4.2 Role of Istio

```text
Istio remains installed
        |
        +--> Ingress Gateway can remain available
        +--> Sidecars can remain injected
        +--> Authorization enforcement is removed for the lab namespace
        +--> JWT validation is removed for the lab namespace
        +--> mTLS is PERMISSIVE for the lab namespace
        +--> Envoy tracing is optional, not part of the required path
```

### 4.3 Why the observability namespace is outside the mesh

The OpenTelemetry Collector and Jaeger will run in a dedicated namespace with automatic Istio injection disabled.

This avoids accidental dependencies on:

- sidecar injection,
- mesh mTLS,
- JWT policies,
- authorization policies,
- Envoy routing,
- mesh tracing configuration.

The tracing backend remains reachable as a normal Kubernetes service.

---

## 5. OpenTelemetry conceptual architecture

### 5.1 OpenTelemetry API

The API defines how application code creates telemetry.

For tracing, the main abstractions are:

```text
TracerProvider
   |
   v
Tracer
   |
   v
Span
```

### 5.2 OpenTelemetry SDK

The SDK implements the API and controls:

- span creation,
- sampling,
- span processors,
- resource attributes,
- exporters,
- context propagation.

### 5.3 Instrumentation

Instrumentation creates telemetry around operations such as:

```text
HTTP request
Database query
RabbitMQ publish
Redis lookup
Business operation
```

Instrumentation may be:

- automatic,
- library-based,
- manual,
- generated by infrastructure such as Envoy.

### 5.4 OpenTelemetry Collector

The Collector receives, processes and exports telemetry.

```text
Receivers → Processors → Exporters
```

The Collector does **not** automatically create application spans. It only processes telemetry that another component generates.

### 5.5 Jaeger

Jaeger is the tracing backend used in this lab. It provides:

- trace ingestion,
- trace storage,
- query APIs,
- service search,
- trace visualization,
- latency analysis.

---

## 6. Prerequisites

The environment must already contain:

- an AKS cluster,
- `kubectl` access,
- Istio installed,
- an Istio ingress gateway,
- the existing e-commerce application,
- application services such as product, order and makeline services,
- Azure Cloud Shell Bash or another Bash terminal,
- `curl`, `openssl` and `jq`.

Validate access:

```bash
az account show -o table
kubectl get nodes
kubectl get pods -n istio-system
```

Validate required local commands:

```bash
for cmd in kubectl curl openssl jq; do
  command -v "$cmd" >/dev/null 2>&1 || echo "Missing command: $cmd"
done
```

---

## 7. Version pins used by the lab

The lab uses explicit image versions for reproducibility:

```text
OpenTelemetry Collector Contrib: 0.154.0
Telemetrygen:                     v0.154.0
Jaeger all-in-one:                1.76.0
```

Jaeger `all-in-one` is used because it is predictable and simple for classroom use. It is not the recommended topology for a high-volume production environment.

> Production note: Current Jaeger releases use the Jaeger v2 architecture. The lab keeps the stable v1 all-in-one image only to reduce deployment complexity and avoid distracting students with backend storage configuration.

---

## 8. Prepare the working environment

Create a working directory:

```bash
mkdir -p ~/otel-jaeger-aks-lab/{manifests,backup,notes}
cd ~/otel-jaeger-aks-lab
```

Set variables:

```bash
export OTEL_NS="otel-advanced"

export APP_NS="aks-store-persistence-lab"

export ISTIO_NS="aks-istio-system"

echo "Application namespace:   $APP_NS"
echo "Observability namespace: $OTEL_NS"
echo "Istio namespace:         $ISTIO_NS"
```

Validate application resources:

```bash
kubectl get deploy,statefulset,pod,svc -n "$APP_NS"
```

If the application is in another namespace:

```bash
export APP_NS="<application-namespace>"
```

---

# Part 1 — Keep Istio Installed but Disable Security Enforcement

## 9. Understand what must be disabled

Three Istio security resources are relevant:

| Resource | Purpose | Lab action |
|---|---|---|
| `AuthorizationPolicy` | Allows or denies requests | Remove lab namespace policies |
| `RequestAuthentication` | Validates JWTs presented to workloads | Remove lab namespace policies |
| `PeerAuthentication` | Controls inbound mTLS requirements | Replace namespace policies with `PERMISSIVE` |

Important nuances:

- `RequestAuthentication` alone does not require every request to contain a JWT.
- An invalid JWT can still be rejected when a `RequestAuthentication` resource applies.
- If no applicable Istio `ALLOW` policy exists, requests are allowed by default.
- A matching `DENY` policy always takes precedence.
- A mesh-wide policy in the Istio root namespace may still affect the application.
- Do not delete all security policies from `istio-system` blindly.

---

## 10. Inventory current Istio policies

List security resources in the application namespace:

```bash
kubectl get authorizationpolicy -n "$APP_NS"
kubectl get requestauthentication -n "$APP_NS"
kubectl get peerauthentication -n "$APP_NS"
```

List security resources across the cluster:

```bash
kubectl get authorizationpolicy -A
kubectl get requestauthentication -A
kubectl get peerauthentication -A
```

Check policies in the Istio root namespace:

```bash
kubectl get authorizationpolicy -n "$ISTIO_NS"
kubectl get requestauthentication -n "$ISTIO_NS"
kubectl get peerauthentication -n "$ISTIO_NS"
```

### Architect checkpoint

If you see a mesh-wide `DENY` or an `ALLOW` policy without a workload selector in `istio-system`, stop and review it with the instructor.

A namespace-level permissive policy cannot override a matching mesh-wide `DENY` policy.

---

## 11. Back up application namespace security policies

The following function exports restorable JSON files without server-generated metadata:

```bash
backup_istio_kind() {
  local kind="$1"
  local namespace="$2"

  for name in $(kubectl get "$kind" -n "$namespace" \
      -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
    echo "Backing up $kind/$name"
    kubectl get "$kind" "$name" -n "$namespace" -o json \
      | jq 'del(
          .metadata.uid,
          .metadata.resourceVersion,
          .metadata.creationTimestamp,
          .metadata.generation,
          .metadata.managedFields,
          .status
        )' \
      > "backup/${kind}-${name}.json"
  done
}

backup_istio_kind authorizationpolicy "$APP_NS"
backup_istio_kind requestauthentication "$APP_NS"
backup_istio_kind peerauthentication "$APP_NS"

ls -la backup
```

### What is happening?

The files can later be applied again with `kubectl apply`. Removing server-generated metadata avoids errors such as:

```text
resourceVersion should not be set on objects to be created
```

---

## 12. Remove namespace authorization and JWT enforcement

Delete all `AuthorizationPolicy` resources from the application namespace:

```bash
kubectl delete authorizationpolicy --all \
  -n "$APP_NS" \
  --ignore-not-found
```

Delete all `RequestAuthentication` resources from the application namespace:

```bash
kubectl delete requestauthentication --all \
  -n "$APP_NS" \
  --ignore-not-found
```

Delete namespace-level or workload-specific `PeerAuthentication` resources so they do not override the lab policy:

```bash
kubectl delete peerauthentication --all \
  -n "$APP_NS" \
  --ignore-not-found
```

Validate:

```bash
kubectl get authorizationpolicy -n "$APP_NS"
kubectl get requestauthentication -n "$APP_NS"
kubectl get peerauthentication -n "$APP_NS"
```

Expected result:

```text
No resources found
```

---

## 13. Apply namespace-level permissive mTLS

Create the policy:

```bash
cat <<EOF > manifests/istio-lab-permissive.yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: otel-lab-permissive
  namespace: ${APP_NS}
spec:
  mtls:
    mode: PERMISSIVE
EOF
```

Apply:

```bash
kubectl apply -f manifests/istio-lab-permissive.yaml
```

Validate:

```bash
kubectl get peerauthentication -n "$APP_NS"
kubectl describe peerauthentication otel-lab-permissive -n "$APP_NS"
```

Expected configuration:

```text
Mode: PERMISSIVE
```

### What is happening?

`PERMISSIVE` allows workloads to accept both:

- Istio mTLS traffic,
- plaintext traffic.

This is useful for the lab because the OpenTelemetry Collector will run outside the mesh.

### Production reality

This configuration is for training. Production environments normally prefer `STRICT` mTLS after all workloads and dependencies are compatible.

---

## 14. Check DestinationRules that might still force mTLS

List destination rules:

```bash
kubectl get destinationrule -A
```

Inspect suspicious global rules:

```bash
kubectl get destinationrule -A -o yaml | grep -n -E "ISTIO_MUTUAL|MUTUAL|DISABLE" -B5 -A8 || true
```

A global `ISTIO_MUTUAL` rule that applies to every Kubernetes service could cause the sidecar to send mTLS to the Collector even though the Collector has no sidecar.

Do not delete DestinationRules unless you confirm they are lab-specific.

---

## 15. Validate access without JWT

Get the ingress gateway address:

```bash
export GATEWAY_IP=$(kubectl get svc istio-ingressgateway \
  -n "$ISTIO_NS" \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "$GATEWAY_IP"
```

Test without a token:

```bash
curl -i "http://${GATEWAY_IP}"
```

```
cat <<EOF > store-front-istio-ingress.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: store-front-gateway
  namespace: ${APP_NS}
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: store-front-vs
  namespace: ${APP_NS}
spec:
  hosts:
  - "*"
  gateways:
  - store-front-gateway
  http:
  - route:
    - destination:
        host: store-front.${APP_NS}.svc.cluster.local
        port:
          number: 80
EOF
```


Interpretation:

| Result | Meaning |
|---|---|
| `200` | Application route works |
| `301/302` | Application redirected the request |
| `404` | Gateway route or host/path mismatch; not necessarily security |
| `401` | Authentication is still being enforced somewhere |
| `403` | Authorization policy or application security may still be blocking |
| `503` | Route exists but the backend may be unavailable |

The goal is not necessarily to obtain `200`. The goal is to confirm that Istio JWT/authorization policy is no longer producing `401/403` for the lab request.

---

# Part 2 — Deploy a Dedicated OpenTelemetry and Jaeger Stack

## 16. Create a namespace outside the Istio mesh

Create the namespace:

```bash
kubectl create namespace "$OTEL_NS" \
  --dry-run=client -o yaml \
  | kubectl apply -f -
```

Disable sidecar injection:

```bash
kubectl label namespace "$OTEL_NS" \
  istio-injection=disabled \
  --overwrite
```

Also exclude the namespace from ambient mode if that label is used in the cluster:

```bash
kubectl label namespace "$OTEL_NS" \
  istio.io/dataplane-mode=none \
  --overwrite
```

Validate labels:

```bash
kubectl get namespace "$OTEL_NS" --show-labels
```

### What is happening?

The observability stack will operate as normal Kubernetes workloads. It will not require Envoy sidecars, mesh certificates or Istio authorization rules.

---

## 17. Deploy Jaeger

Create the manifest:

```bash
cat <<'EOF' > manifests/jaeger.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: otel-advanced
  labels:
    app.kubernetes.io/name: jaeger
    app.kubernetes.io/component: all-in-one
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: jaeger
  template:
    metadata:
      labels:
        app.kubernetes.io/name: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.76.0
        imagePullPolicy: IfNotPresent
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        ports:
        - name: ui
          containerPort: 16686
        - name: otlp-grpc
          containerPort: 4317
        - name: otlp-http
          containerPort: 4318
        readinessProbe:
          tcpSocket:
            port: ui
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: ui
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: otel-advanced
  labels:
    app.kubernetes.io/name: jaeger
spec:
  selector:
    app.kubernetes.io/name: jaeger
  ports:
  - name: ui
    port: 16686
    targetPort: ui
  - name: otlp-grpc
    port: 4317
    targetPort: otlp-grpc
  - name: otlp-http
    port: 4318
    targetPort: otlp-http
EOF
```

Apply and validate:

```bash
kubectl apply -f manifests/jaeger.yaml
kubectl rollout status deployment/jaeger \
  -n "$OTEL_NS" \
  --timeout=180s
kubectl get pod,svc -n "$OTEL_NS"
```

Inspect logs:

```bash
kubectl logs deployment/jaeger \
  -n "$OTEL_NS" \
  --tail=100
```

### What is happening?

Jaeger all-in-one combines:

```text
Collector
Query service
Web UI
In-memory trace storage
```

All trace data is lost if the Jaeger pod is recreated.

---

## 18. Open the Jaeger UI

Run in one terminal:

```bash
kubectl port-forward service/jaeger \
  -n "$OTEL_NS" \
  16686:16686
```

Open:

```text
http://localhost:16686
```

If using Azure Cloud Shell, use the Cloud Shell web preview for port `16686`.

Keep the port-forward running while using the UI.

---

## 19. Create Collector RBAC

The Kubernetes attributes processor needs read-only access to workload metadata.

```bash
cat <<'EOF' > manifests/otel-collector-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector
  namespace: otel-advanced
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-k8sattributes
rules:
- apiGroups: [""]
  resources:
  - pods
  - namespaces
  - nodes
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources:
  - replicasets
  - deployments
  - statefulsets
  - daemonsets
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-k8sattributes
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-k8sattributes
subjects:
- kind: ServiceAccount
  name: otel-collector
  namespace: otel-advanced
EOF
```

Apply:

```bash
kubectl apply -f manifests/otel-collector-rbac.yaml
```

Validate permissions:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:${OTEL_NS}:otel-collector \
  --all-namespaces

kubectl auth can-i list deployments.apps \
  --as=system:serviceaccount:${OTEL_NS}:otel-collector \
  --all-namespaces
```

Expected result:

```text
yes
```

---

## 20. Create the Collector configuration

```bash
cat <<'EOF' > manifests/otel-collector-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: otel-advanced
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      zipkin:
        endpoint: 0.0.0.0:9411

    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
        spike_limit_mib: 80

      k8sattributes:
        auth_type: serviceAccount
        passthrough: false
        pod_association:
        - sources:
          - from: resource_attribute
            name: k8s.pod.ip
        - sources:
          - from: connection
        extract:
          metadata:
          - k8s.namespace.name
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.node.name
          - k8s.deployment.name
          - k8s.statefulset.name

      resource/lab:
        attributes:
        - key: deployment.environment.name
          value: ericsson-training
          action: upsert
        - key: cloud.provider
          value: azure
          action: upsert
        - key: k8s.cluster.name
          value: aks-ericsson-observability
          action: upsert

      batch:
        timeout: 5s
        send_batch_size: 512
        send_batch_max_size: 1024

    exporters:
      otlp/jaeger:
        endpoint: jaeger.otel-advanced.svc.cluster.local:4317
        tls:
          insecure: true

      debug:
        verbosity: basic

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      zpages:
        endpoint: 0.0.0.0:55679
      pprof:
        endpoint: 0.0.0.0:1777

    service:
      extensions: [health_check, zpages, pprof]
      telemetry:
        logs:
          level: info
      pipelines:
        traces:
          receivers: [otlp, zipkin]
          processors: [memory_limiter, k8sattributes, resource/lab, batch]
          exporters: [otlp/jaeger, debug]
EOF
```

### Component interpretation

```text
OTLP receiver
  Accepts native OpenTelemetry traffic.

Zipkin receiver
  Accepts legacy or Zipkin-compatible spans.

memory_limiter
  Protects the Collector from excessive memory consumption.

k8sattributes
  Enriches spans with Kubernetes metadata.

resource/lab
  Adds environment, provider and cluster attributes.

batch
  Groups telemetry before export for better efficiency.

otlp/jaeger
  Sends traces to Jaeger using OTLP/gRPC.

debug
  Prints summary telemetry to Collector logs.
```

---

## 21. Deploy the Collector

```bash
cat <<'EOF' > manifests/otel-collector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: otel-advanced
  labels:
    app.kubernetes.io/name: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: otel-collector
  template:
    metadata:
      labels:
        app.kubernetes.io/name: otel-collector
    spec:
      serviceAccountName: otel-collector
      containers:
      - name: otel-collector
        image: ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-contrib:0.154.0
        imagePullPolicy: IfNotPresent
        args:
        - --config=/conf/otel-collector-config.yaml
        env:
        - name: GOMEMLIMIT
          value: 400MiB
        ports:
        - name: otlp-grpc
          containerPort: 4317
        - name: otlp-http
          containerPort: 4318
        - name: zipkin
          containerPort: 9411
        - name: health
          containerPort: 13133
        - name: zpages
          containerPort: 55679
        - name: pprof
          containerPort: 1777
        - name: metrics
          containerPort: 8888
        readinessProbe:
          httpGet:
            path: /
            port: health
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: health
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: collector-config
          mountPath: /conf
          readOnly: true
      volumes:
      - name: collector-config
        configMap:
          name: otel-collector-config
          items:
          - key: otel-collector-config.yaml
            path: otel-collector-config.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: otel-advanced
  labels:
    app.kubernetes.io/name: otel-collector
spec:
  selector:
    app.kubernetes.io/name: otel-collector
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: otlp-grpc
  - name: otlp-http
    port: 4318
    targetPort: otlp-http
  - name: zipkin
    port: 9411
    targetPort: zipkin
  - name: health
    port: 13133
    targetPort: health
  - name: zpages
    port: 55679
    targetPort: zpages
  - name: pprof
    port: 1777
    targetPort: pprof
  - name: metrics
    port: 8888
    targetPort: metrics
EOF
```

Apply the ConfigMap and deployment:

```bash
kubectl apply -f manifests/otel-collector-config.yaml
kubectl apply -f manifests/otel-collector.yaml
```

Validate rollout:

```bash
kubectl rollout status deployment/otel-collector \
  -n "$OTEL_NS" \
  --timeout=180s

kubectl get pods,svc -n "$OTEL_NS"
```

Inspect startup logs:

```bash
kubectl logs deployment/otel-collector \
  -n "$OTEL_NS" \
  --tail=150
```

Expected indicators:

```text
Everything is ready
Starting GRPC server
Starting HTTP server
```

Exact log messages may vary by Collector version.

---

## 22. Verify that the observability stack has no Istio sidecars

```bash
kubectl get pods -n "$OTEL_NS" \
  -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```

Expected containers:

```text
jaeger
otel-collector
```

You should not see:

```text
istio-proxy
```

---

# Part 3 — Validate the Collector Independently from the Application

## 23. Validate the Collector health endpoint

Run:

```bash
kubectl port-forward service/otel-collector \
  -n "$OTEL_NS" \
  13133:13133
```

From another terminal:

```bash
curl -i http://localhost:13133/
```

Expected result:

```text
HTTP/1.1 200 OK
```

Stop the port-forward with `Ctrl+C`.

---

## 24. Inspect Collector internal metrics

Run:

```bash
kubectl port-forward service/otel-collector \
  -n "$OTEL_NS" \
  8888:8888
```

From another terminal:

```bash
curl -s http://localhost:8888/metrics \
  | grep -E "otelcol_receiver|otelcol_exporter|otelcol_processor" \
  | head -40
```

Important metrics include:

```text
otelcol_receiver_accepted_spans
otelcol_receiver_refused_spans
otelcol_exporter_sent_spans
otelcol_exporter_send_failed_spans
otelcol_process_memory_rss
```

Stop the port-forward with `Ctrl+C`.

---

## 25. Inspect zPages

Run:

```bash
kubectl port-forward service/otel-collector \
  -n "$OTEL_NS" \
  55679:55679
```

Open:

```text
http://localhost:55679/debug/servicez
```

Useful endpoints:

```text
/debug/servicez
/debug/pipelinez
/debug/tracez
```

### What is happening?

zPages gives runtime diagnostic information about the Collector. It is useful for training and troubleshooting but should not normally be exposed publicly.

---

## 26. Generate native OTLP traces with telemetrygen

Create a Job:

```bash
cat <<'EOF' > manifests/telemetrygen.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: telemetrygen-traces
  namespace: otel-advanced
spec:
  ttlSecondsAfterFinished: 300
  backoffLimit: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: telemetrygen
    spec:
      restartPolicy: Never
      containers:
      - name: telemetrygen
        image: ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:v0.154.0
        args:
        - traces
        - --otlp-endpoint=otel-collector.otel-advanced.svc.cluster.local:4317
        - --otlp-insecure
        - --traces=10
EOF
```

Run:

```bash
kubectl delete job telemetrygen-traces \
  -n "$OTEL_NS" \
  --ignore-not-found

kubectl apply -f manifests/telemetrygen.yaml
kubectl wait --for=condition=complete \
  job/telemetrygen-traces \
  -n "$OTEL_NS" \
  --timeout=120s
```

Inspect generator output:

```bash
kubectl logs job/telemetrygen-traces -n "$OTEL_NS"
```

Inspect Collector output:

```bash
kubectl logs deployment/otel-collector \
  -n "$OTEL_NS" \
  --since=5m
```

In Jaeger:

1. Open the Jaeger UI.
2. Find the service generated by telemetrygen.
3. Select **Find Traces**.
4. Open one trace.

### Pipeline validated

```text
Telemetrygen
   |
   | OTLP/gRPC
   v
OpenTelemetry Collector
   |
   | OTLP/gRPC
   v
Jaeger
```

---

# Part 4 — Create a Multi-Service Distributed Trace

## 27. Why use the Zipkin receiver for this exercise?

A manually submitted Zipkin trace lets the class create parent/child relationships without rebuilding the existing application.

This tests:

- Collector protocol interoperability,
- trace ID reuse,
- parent span IDs,
- service names,
- timing relationships,
- attributes,
- Jaeger visualization.

---

## 28. Port-forward the Zipkin receiver

Run in one terminal:

```bash
kubectl port-forward service/otel-collector \
  -n "$OTEL_NS" \
  9411:9411
```

Keep it running.

---

## 29. Generate a complete parent/child trace

Run in another terminal:

```bash
export TRACE_ID=$(openssl rand -hex 16)
export ROOT_SPAN=$(openssl rand -hex 8)
export PRODUCT_SPAN=$(openssl rand -hex 8)
export INVENTORY_SPAN=$(openssl rand -hex 8)
export PAYMENT_SPAN=$(openssl rand -hex 8)
export TS=$(($(date +%s%N)/1000))

cat > /tmp/complete-trace.json <<EOF
[
  {
    "traceId": "${TRACE_ID}",
    "id": "${ROOT_SPAN}",
    "kind": "SERVER",
    "name": "GET /checkout",
    "timestamp": ${TS},
    "duration": 1300000,
    "localEndpoint": {"serviceName": "store-front"},
    "tags": {
      "http.request.method": "GET",
      "url.path": "/checkout",
      "order.id": "ORD-1001",
      "lab.outcome": "success"
    }
  },
  {
    "traceId": "${TRACE_ID}",
    "parentId": "${ROOT_SPAN}",
    "id": "${PRODUCT_SPAN}",
    "kind": "CLIENT",
    "name": "GET product dog-food",
    "timestamp": $((TS + 20000)),
    "duration": 160000,
    "localEndpoint": {"serviceName": "product-service"},
    "tags": {
      "db.system": "mongodb",
      "product.id": "dog-food",
      "cache.system": "redis"
    }
  },
  {
    "traceId": "${TRACE_ID}",
    "parentId": "${ROOT_SPAN}",
    "id": "${INVENTORY_SPAN}",
    "kind": "CLIENT",
    "name": "reserve inventory",
    "timestamp": $((TS + 220000)),
    "duration": 720000,
    "localEndpoint": {"serviceName": "inventory-service"},
    "tags": {
      "inventory.product.id": "dog-food",
      "inventory.quantity": "1"
    }
  },
  {
    "traceId": "${TRACE_ID}",
    "parentId": "${ROOT_SPAN}",
    "id": "${PAYMENT_SPAN}",
    "kind": "CLIENT",
    "name": "authorize payment",
    "timestamp": $((TS + 970000)),
    "duration": 240000,
    "localEndpoint": {"serviceName": "payment-service"},
    "tags": {
      "payment.method": "card",
      "payment.currency": "USD"
    }
  }
]
EOF

curl -i -X POST \
  http://localhost:9411/api/v2/spans \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/complete-trace.json

echo "Trace ID: $TRACE_ID"
```

In Jaeger:

1. Select `store-front`.
2. Click **Find Traces**.
3. Open `GET /checkout`.
4. Confirm that the child spans share the same trace ID.

Expected hierarchy:

```text
GET /checkout — store-front
├── GET product dog-food — product-service
├── reserve inventory — inventory-service
└── authorize payment — payment-service
```

### Discussion

- Which operation used the most time?
- Are these spans sequential or parallel?
- Is the root duration consistent with the children?
- Which attributes are technical?
- Which attributes represent business context?

---

## 30. Demonstrate broken context propagation

Create two operations that should belong to the same request but use different trace IDs:

```bash
export TRACE_FRONT=$(openssl rand -hex 16)
export TRACE_INVENTORY=$(openssl rand -hex 16)
export SPAN_FRONT=$(openssl rand -hex 8)
export SPAN_INVENTORY=$(openssl rand -hex 8)
export TS=$(($(date +%s%N)/1000))

cat > /tmp/broken-context.json <<EOF
[
  {
    "traceId": "${TRACE_FRONT}",
    "id": "${SPAN_FRONT}",
    "kind": "SERVER",
    "name": "POST /orders",
    "timestamp": ${TS},
    "duration": 800000,
    "localEndpoint": {"serviceName": "order-service"},
    "tags": {
      "order.id": "ORD-BROKEN-1",
      "context.propagation": "broken"
    }
  },
  {
    "traceId": "${TRACE_INVENTORY}",
    "id": "${SPAN_INVENTORY}",
    "kind": "SERVER",
    "name": "reserve inventory",
    "timestamp": $((TS + 100000)),
    "duration": 500000,
    "localEndpoint": {"serviceName": "inventory-service"},
    "tags": {
      "order.id": "ORD-BROKEN-1",
      "context.propagation": "broken"
    }
  }
]
EOF

curl -i -X POST \
  http://localhost:9411/api/v2/spans \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/broken-context.json

echo "Frontend trace ID:  $TRACE_FRONT"
echo "Inventory trace ID: $TRACE_INVENTORY"
```

Expected result:

Jaeger displays two separate traces even though both spans contain the same `order.id`.

### Key lesson

Correlation attributes do not replace trace context.

The same business identifier can help investigators search, but distributed tracing requires propagation of the same trace context across service boundaries.

---

# Part 5 — Trace Context and Baggage

## 31. W3C Trace Context

The standard HTTP header is:

```text
traceparent: 00-<trace-id>-<parent-span-id>-<trace-flags>
```

Example:

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Components:

```text
00                                Version
4bf92f3577b34da6a3ce929d0e0e4736  Trace ID
00f067aa0ba902b7                  Parent span ID
01                                Sampled flag
```

When Service A calls Service B:

```text
Service A creates or receives a trace context
        |
        | inject traceparent
        v
HTTP request to Service B
        |
        | extract traceparent
        v
Service B creates a child span
```

If injection or extraction fails, the trace becomes fragmented.

---

## 32. Baggage

Baggage carries application-defined key/value pairs across service boundaries.

Example:

```text
baggage: tenant.id=training,order.id=ORD-1001
```

Possible uses:

- tenant identifier,
- transaction classification,
- experiment cohort,
- routing context.

Risks:

- baggage is transmitted across service boundaries,
- excessive baggage increases request size,
- sensitive data can leak,
- high-cardinality values can increase observability costs.

Important nuance:

> Baggage is not automatically added to every span as an attribute. Instrumentation must explicitly read it and record the values that are appropriate.

---

# Part 6 — Integrate Existing Application Workloads

## 33. First identify application languages and images

```bash
kubectl get deployments -n "$APP_NS" \
  -o custom-columns='DEPLOYMENT:.metadata.name,CONTAINERS:.spec.template.spec.containers[*].name,IMAGES:.spec.template.spec.containers[*].image'
```

Inspect one workload:

```bash
export TARGET_DEPLOYMENT="product-service"

kubectl get deployment "$TARGET_DEPLOYMENT" \
  -n "$APP_NS" \
  -o yaml > "notes/${TARGET_DEPLOYMENT}-before-otel.yaml"
```

Search for existing OpenTelemetry configuration:

```bash
kubectl get deployment "$TARGET_DEPLOYMENT" \
  -n "$APP_NS" \
  -o yaml \
  | grep -i -E "OTEL_|opentelemetry|javaagent|CORECLR|NODE_OPTIONS" || true
```

---

## 34. Critical distinction: configuration is not instrumentation

The following environment variables configure an OpenTelemetry SDK or agent:

```text
OTEL_EXPORTER_OTLP_ENDPOINT
OTEL_SERVICE_NAME
OTEL_RESOURCE_ATTRIBUTES
OTEL_PROPAGATORS
OTEL_TRACES_SAMPLER
```

They do not install an SDK and they do not modify application code.

This command alone does **not** guarantee traces:

```bash
kubectl set env deployment/<name> \
  OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.otel-advanced.svc.cluster.local:4318
```

Traces appear only if the workload already includes one of the following:

- an OpenTelemetry SDK,
- an OpenTelemetry automatic instrumentation agent,
- a framework integration that emits OpenTelemetry,
- manual tracing code.

---

## 35. Configure an application that is already instrumented

Only use this section when the application already contains OpenTelemetry instrumentation.

```bash
export TARGET_DEPLOYMENT="product-service"
export OTEL_SERVICE_NAME="product-service"

kubectl set env deployment/"$TARGET_DEPLOYMENT" \
  -n "$APP_NS" \
  OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector.${OTEL_NS}.svc.cluster.local:4318" \
  OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf" \
  OTEL_SERVICE_NAME="$OTEL_SERVICE_NAME" \
  OTEL_RESOURCE_ATTRIBUTES="service.namespace=${APP_NS},deployment.environment.name=ericsson-training" \
  OTEL_PROPAGATORS="tracecontext,baggage" \
  OTEL_TRACES_SAMPLER="parentbased_always_on"
```

Wait for rollout:

```bash
kubectl rollout status deployment/"$TARGET_DEPLOYMENT" \
  -n "$APP_NS" \
  --timeout=180s
```

Check the new pod environment:

```bash
POD=$(kubectl get pod -n "$APP_NS" \
  -l app="$TARGET_DEPLOYMENT" \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)

echo "$POD"
```

Because labels vary, locate the pod manually if the variable is empty:

```bash
kubectl get pods -n "$APP_NS" --show-labels
```

Generate application traffic and inspect Jaeger.

---

## 36. Language-specific instrumentation choices

### Java

Typical automatic instrumentation requires the Java agent inside the image or mounted into the pod:

```text
JAVA_TOOL_OPTIONS=-javaagent:/otel/opentelemetry-javaagent.jar
```

### .NET

Automatic instrumentation generally requires the .NET profiler and startup hook files to exist inside the container.

### Node.js

The image must include the relevant OpenTelemetry packages, and startup configuration normally uses `NODE_OPTIONS` or an instrumentation bootstrap file.

### Python

The image must include the OpenTelemetry distribution and instrumentation packages. Startup commonly uses:

```text
opentelemetry-instrument python app.py
```

### Go

Go typically uses compile-time/manual instrumentation or supported zero-code approaches. Environment variables alone cannot inject the SDK into a compiled binary.

### Architect discussion

Why is automatic instrumentation convenient but incomplete?

It normally observes framework operations such as HTTP, database and messaging calls. It may not understand business operations such as:

```text
ValidateOrderPolicy
CalculateDiscount
ReserveLimitedInventory
ApproveHighRiskPayment
```

Those usually require manual spans.

---

## 37. Manual instrumentation model

Language-neutral pseudocode:

```text
receive HTTP request
extract parent context
start span "CreateOrder"

  start child span "ValidateCustomer"
  validate customer
  end child span

  start child span "ReserveInventory"
  reserve product
  end child span

  start child span "PublishOrderCreated"
  inject trace context into message headers
  publish message
  end child span

end root span
```

For messaging, context must be injected into message headers by the producer and extracted by the consumer.

This is essential for RabbitMQ flows because the consumer may process the message later and on another pod.

---

# Part 7 — Sampling

## 38. Head sampling

Head sampling decides at or near the beginning of a trace whether it should be recorded.

Typical SDK configuration:

```bash
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.10
```

Meaning:

```text
Keep approximately 10% of new root traces.
Child services respect the parent sampling decision.
```

Advantages:

- simple,
- low overhead,
- decision made early.

Risk:

- a trace may be discarded before the system knows that it became slow or failed.

---

## 39. Tail sampling

Tail sampling waits for spans to arrive and evaluates the completed or partially completed trace.

Possible rules:

```text
Keep errors
Keep slow traces
Keep VIP customer transactions
Keep 10% of normal traces
```

Tradeoffs:

- more memory,
- delayed decisions,
- requires trace affinity when scaled,
- more operational complexity.

---

## 40. Enable tail sampling in the Collector

Create an advanced Collector configuration:

```bash
cat <<'EOF' > manifests/otel-collector-config-tail-sampling.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: otel-advanced
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      zipkin:
        endpoint: 0.0.0.0:9411

    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
        spike_limit_mib: 80

      k8sattributes:
        auth_type: serviceAccount
        passthrough: false
        pod_association:
        - sources:
          - from: resource_attribute
            name: k8s.pod.ip
        - sources:
          - from: connection
        extract:
          metadata:
          - k8s.namespace.name
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.node.name
          - k8s.deployment.name
          - k8s.statefulset.name

      resource/lab:
        attributes:
        - key: deployment.environment.name
          value: ericsson-training
          action: upsert
        - key: cloud.provider
          value: azure
          action: upsert
        - key: k8s.cluster.name
          value: aks-ericsson-observability
          action: upsert

      tail_sampling:
        decision_wait: 10s
        num_traces: 50000
        expected_new_traces_per_sec: 100
        policies:
        - name: keep-business-errors
          type: string_attribute
          string_attribute:
            key: lab.outcome
            values: [error]
        - name: keep-slow-traces
          type: latency
          latency:
            threshold_ms: 500
        - name: keep-normal-baseline
          type: probabilistic
          probabilistic:
            sampling_percentage: 10

      batch:
        timeout: 5s
        send_batch_size: 512
        send_batch_max_size: 1024

    exporters:
      otlp/jaeger:
        endpoint: jaeger.otel-advanced.svc.cluster.local:4317
        tls:
          insecure: true
      debug:
        verbosity: basic

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      zpages:
        endpoint: 0.0.0.0:55679
      pprof:
        endpoint: 0.0.0.0:1777

    service:
      extensions: [health_check, zpages, pprof]
      telemetry:
        logs:
          level: info
      pipelines:
        traces:
          receivers: [otlp, zipkin]
          processors: [memory_limiter, k8sattributes, resource/lab, tail_sampling, batch]
          exporters: [otlp/jaeger, debug]
EOF
```

Apply and restart:

```bash
kubectl apply -f manifests/otel-collector-config-tail-sampling.yaml
kubectl rollout restart deployment/otel-collector -n "$OTEL_NS"
kubectl rollout status deployment/otel-collector \
  -n "$OTEL_NS" \
  --timeout=180s
```

Validate logs:

```bash
kubectl logs deployment/otel-collector \
  -n "$OTEL_NS" \
  --tail=150
```

### Important processor ordering

The `k8sattributes` processor runs before `tail_sampling` because tail sampling rebuilds batches and may lose the original receiver context used for Kubernetes metadata association.

---

## 41. Generate traces for tail-sampling analysis

With the Zipkin port-forward still active, create an error-labelled trace:

```bash
export TRACE_ID=$(openssl rand -hex 16)
export SPAN_ID=$(openssl rand -hex 8)
export TS=$(($(date +%s%N)/1000))

cat > /tmp/error-trace.json <<EOF
[
  {
    "traceId": "${TRACE_ID}",
    "id": "${SPAN_ID}",
    "kind": "SERVER",
    "name": "POST /orders error",
    "timestamp": ${TS},
    "duration": 120000,
    "localEndpoint": {"serviceName": "order-service"},
    "tags": {
      "lab.outcome": "error",
      "error.type": "RabbitMQUnavailable",
      "order.id": "ORD-ERROR-1"
    }
  }
]
EOF

curl -s -X POST \
  http://localhost:9411/api/v2/spans \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/error-trace.json
```

Create a slow trace:

```bash
export TRACE_ID=$(openssl rand -hex 16)
export SPAN_ID=$(openssl rand -hex 8)
export TS=$(($(date +%s%N)/1000))

cat > /tmp/slow-trace.json <<EOF
[
  {
    "traceId": "${TRACE_ID}",
    "id": "${SPAN_ID}",
    "kind": "SERVER",
    "name": "GET /products slow",
    "timestamp": ${TS},
    "duration": 1800000,
    "localEndpoint": {"serviceName": "product-service"},
    "tags": {
      "lab.outcome": "success",
      "product.id": "dog-food"
    }
  }
]
EOF

curl -s -X POST \
  http://localhost:9411/api/v2/spans \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/slow-trace.json
```

Wait at least the configured decision period:

```bash
sleep 12
```

Search Jaeger for:

```text
order-service
product-service
```

Expected behavior:

- the error-labelled trace is retained,
- the slow trace is retained,
- only approximately 10% of normal fast traces are retained.

---

## 42. Tail-sampling scaling warning

Tail sampling requires all spans from the same trace to reach the same Collector instance.

This is safe in the lab because:

```text
Collector replicas = 1
```

If the deployment is scaled directly:

```bash
kubectl scale deployment otel-collector \
  -n "$OTEL_NS" \
  --replicas=3
```

different spans from the same trace may reach different replicas.

A production design normally uses two layers:

```text
Agent/Gateway Collectors
       |
       | load-balancing exporter using trace ID
       v
Tail-sampling Collector tier
       |
       v
Tracing backend
```

Return the lab to one replica:

```bash
kubectl scale deployment otel-collector \
  -n "$OTEL_NS" \
  --replicas=1
```

---

# Part 8 — Troubleshooting Scenarios

## 43. Troubleshooting model

Always follow the telemetry path:

```text
Instrumentation
   |
   v
Network / DNS
   |
   v
Receiver
   |
   v
Processor
   |
   v
Exporter
   |
   v
Backend / UI
```

Do not start by assuming Jaeger is the problem.

---

## 44. Scenario 1 — No traces in Jaeger

Validate components:

```bash
kubectl get pods -n "$OTEL_NS"
kubectl get svc -n "$OTEL_NS"
```

Check Collector logs:

```bash
kubectl logs deployment/otel-collector \
  -n "$OTEL_NS" \
  --since=10m
```

Check Jaeger logs:

```bash
kubectl logs deployment/jaeger \
  -n "$OTEL_NS" \
  --since=10m
```

Check Collector endpoint DNS from the application namespace:

```bash
kubectl run dns-check \
  -n "$APP_NS" \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- nslookup otel-collector.${OTEL_NS}.svc.cluster.local
```

Test TCP connectivity:

```bash
kubectl run tcp-check \
  -n "$APP_NS" \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- sh -c 'nc -vz otel-collector.otel-advanced.svc.cluster.local 4317'
```

Potential causes:

- application has no instrumentation,
- wrong endpoint,
- wrong protocol,
- invalid DNS name,
- sidecar or DestinationRule mTLS conflict,
- Collector receiver not enabled,
- Collector exporter failure,
- trace sampled out,
- wrong Jaeger time range.

---

## 45. Scenario 2 — Collector exporter failure

Save the working configuration:

```bash
cp manifests/otel-collector-config-tail-sampling.yaml \
   backup/otel-collector-config-working.yaml
```

Create a temporary broken exporter endpoint:

```bash
sed 's/jaeger\.otel-advanced\.svc\.cluster\.local:4317/jaeger-does-not-exist.otel-advanced.svc.cluster.local:4317/' \
  manifests/otel-collector-config-tail-sampling.yaml \
  > manifests/otel-collector-config-broken-exporter.yaml
```

Apply:

```bash
kubectl apply -f manifests/otel-collector-config-broken-exporter.yaml
kubectl rollout restart deployment/otel-collector -n "$OTEL_NS"
kubectl rollout status deployment/otel-collector \
  -n "$OTEL_NS" \
  --timeout=180s
```

Generate telemetry again:

```bash
kubectl delete job telemetrygen-traces \
  -n "$OTEL_NS" \
  --ignore-not-found
kubectl apply -f manifests/telemetrygen.yaml
```

Inspect errors:

```bash
kubectl logs deployment/otel-collector \
  -n "$OTEL_NS" \
  --since=5m \
  | grep -i -E "error|failed|export|dns|lookup" || true
```

Inspect internal metric:

```bash
kubectl port-forward service/otel-collector \
  -n "$OTEL_NS" \
  8888:8888
```

In another terminal:

```bash
curl -s http://localhost:8888/metrics \
  | grep otelcol_exporter_send_failed_spans
```

Restore:

```bash
kubectl apply -f backup/otel-collector-config-working.yaml
kubectl rollout restart deployment/otel-collector -n "$OTEL_NS"
kubectl rollout status deployment/otel-collector \
  -n "$OTEL_NS" \
  --timeout=180s
```

### Discussion

- Did the receiver continue accepting spans?
- Did the backend receive them?
- Where should retry and queue behavior be configured?
- How long can an in-memory queue protect against backend failure?

---

## 46. Scenario 3 — Collector unavailable

Scale the Collector to zero:

```bash
kubectl scale deployment otel-collector \
  -n "$OTEL_NS" \
  --replicas=0
```

Validate:

```bash
kubectl get pods -n "$OTEL_NS"
kubectl get endpoints otel-collector -n "$OTEL_NS"
```

Run telemetrygen:

```bash
kubectl delete job telemetrygen-traces \
  -n "$OTEL_NS" \
  --ignore-not-found
kubectl apply -f manifests/telemetrygen.yaml
kubectl logs job/telemetrygen-traces -n "$OTEL_NS" || true
```

Restore:

```bash
kubectl scale deployment otel-collector \
  -n "$OTEL_NS" \
  --replicas=1

kubectl rollout status deployment/otel-collector \
  -n "$OTEL_NS" \
  --timeout=180s
```

### Discussion

- Should application requests fail when telemetry export fails?
- Should tracing be synchronous or asynchronous?
- What happens to spans buffered only in application memory?
- Would a local agent Collector improve resilience?

Correct principle:

> Observability should not normally become a hard dependency for completing the business transaction.

---

## 47. Scenario 4 — Invalid Collector configuration

Back up the active ConfigMap:

```bash
kubectl get configmap otel-collector-config \
  -n "$OTEL_NS" \
  -o yaml \
  > backup/otel-collector-configmap-active.yaml
```

Do not intentionally break the shared class environment unless the instructor approves it.

Typical symptoms of invalid configuration:

```text
CrashLoopBackOff
unknown component type
invalid keys
pipeline references a missing processor
cannot decode configuration
```

Useful commands:

```bash
kubectl get pods -n "$OTEL_NS"
kubectl describe pod -n "$OTEL_NS" -l app.kubernetes.io/name=otel-collector
kubectl logs deployment/otel-collector -n "$OTEL_NS" --previous
```

---

## 48. Scenario 5 — Istio is still returning 403

Check namespace policies:

```bash
kubectl get authorizationpolicy -n "$APP_NS"
kubectl get requestauthentication -n "$APP_NS"
```

Check root namespace policies:

```bash
kubectl get authorizationpolicy -n "$ISTIO_NS"
kubectl get requestauthentication -n "$ISTIO_NS"
```

Check Envoy response details:

```bash
kubectl logs -n "$APP_NS" \
  -l app=<application-label> \
  -c istio-proxy \
  --tail=100
```

Common Envoy indicators:

```text
RBAC: access denied
Jwt is not in the form of Header.Payload.Signature
```

Also determine whether the application itself returns the 403.

A 403 is not automatically an Istio error.

---

## 49. Scenario 6 — Collector connectivity fails only from sidecar-enabled pods

Symptoms:

- telemetrygen in `otel-advanced` works,
- application pods cannot export,
- the Collector has no Istio sidecar,
- application pods do have sidecars.

Investigate:

```bash
kubectl get destinationrule -A -o yaml \
  | grep -n -E "ISTIO_MUTUAL|MUTUAL" -B8 -A10 || true
```

Check application sidecar logs:

```bash
kubectl logs <application-pod> \
  -n "$APP_NS" \
  -c istio-proxy \
  --tail=150
```

Likely cause:

A DestinationRule forces mTLS to the Collector even though the Collector is outside the mesh.

Potential remediation:

Create a narrow DestinationRule for the Collector that disables TLS only for this destination, after instructor review:

```bash
cat <<EOF > manifests/otel-collector-disable-istio-tls.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: otel-collector-plaintext
  namespace: ${APP_NS}
spec:
  host: otel-collector.${OTEL_NS}.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
EOF
```

Apply only when the conflict is confirmed:

```bash
kubectl apply -f manifests/otel-collector-disable-istio-tls.yaml
```

---

# Part 9 — Optional Istio Trace Comparison

## 50. Optional objective

The core lab sends traces from application instrumentation or test generators directly to the Collector.

Because Istio remains installed, an optional comparison can show the difference between:

```text
Envoy network spans
Application spans
```

Example:

```text
Envoy span:
POST /orders — 450 ms

Application spans:
CreateOrder — 450 ms
├── ValidateCustomer — 15 ms
├── ReserveInventory — 90 ms
├── SaveOrder — 110 ms
└── PublishOrderCreated — 180 ms
```

Envoy sees network operations. Application instrumentation sees business operations.

---

## 51. Check whether an Istio OpenTelemetry provider already exists

```bash
kubectl get configmap istio \
  -n "$ISTIO_NS" \
  -o jsonpath='{.data.mesh}' \
  | grep -A12 -B3 -i "opentelemetry" || true
```

Only continue if an extension provider already points to the lab Collector or the instructor has approved changing the Istio installation profile.

Do not run `istioctl install` blindly on an existing shared cluster.

---

## 52. Optional Telemetry resource

If a provider named `otel-tracing` already exists:

```bash
cat <<EOF > manifests/istio-optional-tracing.yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: optional-envoy-tracing
  namespace: ${APP_NS}
spec:
  tracing:
  - providers:
    - name: otel-tracing
    randomSamplingPercentage: 100
EOF
```

Apply:

```bash
kubectl apply -f manifests/istio-optional-tracing.yaml
```

Restart only the workloads used for the exercise:

```bash
kubectl rollout restart deployment/<deployment-name> -n "$APP_NS"
kubectl rollout status deployment/<deployment-name> \
  -n "$APP_NS" \
  --timeout=180s
```

This section does not re-enable security enforcement.

---

# Part 10 — Production Architecture

## 53. Collector deployment patterns

### Gateway pattern

```text
Applications
    |
    v
Central Collector Deployment
    |
    v
Backend
```

Benefits:

- centralized policy,
- simpler endpoints,
- shared processing,
- lower operational footprint.

Risks:

- shared bottleneck,
- network dependency,
- larger blast radius.

### Agent plus gateway pattern

```text
Application pods
      |
      v
Node/sidecar agents
      |
      v
Gateway Collectors
      |
      v
Backends
```

Benefits:

- local buffering,
- local enrichment,
- reduced application complexity,
- centralized routing at gateway tier.

Tradeoff:

- more components,
- higher resource consumption,
- more operational complexity.

---

## 54. High availability

A production Collector design should consider:

- multiple replicas,
- PodDisruptionBudget,
- topology spread constraints,
- anti-affinity,
- horizontal autoscaling,
- exporter retry queues,
- persistent queues where required,
- overload protection,
- Collector self-monitoring.

Example conceptual flow:

```text
SDKs
 |
 v
Collector Gateway Service
 |
 +--> Collector 1
 +--> Collector 2
 +--> Collector 3
 |
 v
Jaeger collectors / managed backend
```

Tail sampling requires trace-aware routing.

---

## 55. Jaeger production considerations

Jaeger all-in-one with in-memory storage is appropriate for:

- demonstrations,
- local development,
- short workshops,
- small temporary environments.

Production normally requires:

- separate ingestion and query roles,
- persistent storage,
- retention policies,
- backup strategy,
- horizontal scaling,
- authentication for the UI,
- TLS,
- network isolation,
- capacity planning.

---

## 56. Security and governance

Although Istio security enforcement is disabled for this lab, production telemetry pipelines should protect:

- OTLP endpoints,
- Jaeger query access,
- tenant data,
- trace attributes,
- credentials,
- customer identifiers,
- payload content.

Controls may include:

- TLS or mTLS,
- NetworkPolicy,
- private endpoints,
- authenticated ingestion,
- attribute filtering,
- redaction processors,
- least-privilege RBAC,
- retention controls,
- audit logging.

### Governance question

Should these attributes be stored in a trace?

```text
customer.email
credit.card.number
access.token
personal.address
```

Correct answer:

Usually not. Observability data is still organizational data and must follow security, privacy and retention requirements.

---

## 57. Cost and cardinality

Tracing cost depends on:

```text
request volume
× spans per request
× bytes per span
× retention period
× indexing overhead
```

High-cardinality attributes include:

```text
user.id
order.id
session.id
request.id
```

These attributes may be valuable for investigation but expensive for indexing and aggregation.

Architects must balance:

- diagnostic value,
- storage cost,
- privacy,
- search performance,
- retention.

---

# Part 11 — Final War Room Exercise

## 58. Instructor-triggered scenarios

The instructor selects one scenario without telling the students:

1. Collector scaled to zero.
2. Incorrect OTLP endpoint in one deployment.
3. Jaeger scaled to zero.
4. Broken trace propagation.
5. Tail sampling hides most normal traces.
6. Global Istio policy still returns 403.
7. DestinationRule forces mTLS to the non-mesh Collector.
8. Application contains OTEL environment variables but no SDK or agent.

---

## 59. Team diagnosis template

Create the file:

```bash
cat <<'EOF' > notes/final-otel-war-room.md
# OpenTelemetry and Jaeger War Room Diagnosis

## Team
-

## Reported symptom
-

## Business impact
-

## Is the application failing or only observability?
-

## Instrumentation evidence
-

## Network and DNS evidence
-

## Collector receiver evidence
-

## Processor evidence
-

## Exporter evidence
-

## Jaeger evidence
-

## Istio security evidence
-

## Root cause
-

## Recovery action
-

## Preventive control
-

## Recommended alert
-

## Recommended production architecture improvement
-
EOF
```

---

## 60. Guiding questions

- Is telemetry being generated?
- Is the application instrumented or only configured?
- Is the OTLP protocol correct?
- Can the pod resolve the Collector service?
- Can the pod reach port `4317` or `4318`?
- Does the receiver accept spans?
- Does a processor drop the trace?
- Does the exporter send spans successfully?
- Is Jaeger healthy?
- Is the Jaeger time range correct?
- Is the trace sampled?
- Did trace context propagate?
- Is an Istio policy still blocking traffic?
- Is Istio forcing mTLS to a destination outside the mesh?

---

# Part 12 — Cleanup and Restoration

## 61. Remove optional Istio tracing resources

```bash
kubectl delete -f manifests/istio-optional-tracing.yaml \
  --ignore-not-found
```

Remove the optional DestinationRule if it was created:

```bash
kubectl delete -f manifests/otel-collector-disable-istio-tls.yaml \
  --ignore-not-found
```

---

## 62. Restore original application deployment

If you changed an application deployment and saved the original manifest:

```bash
kubectl apply -f "notes/${TARGET_DEPLOYMENT}-before-otel.yaml"
```

Alternatively, remove OpenTelemetry environment variables:

```bash
kubectl set env deployment/"$TARGET_DEPLOYMENT" \
  -n "$APP_NS" \
  OTEL_EXPORTER_OTLP_ENDPOINT- \
  OTEL_EXPORTER_OTLP_PROTOCOL- \
  OTEL_SERVICE_NAME- \
  OTEL_RESOURCE_ATTRIBUTES- \
  OTEL_PROPAGATORS- \
  OTEL_TRACES_SAMPLER-
```

---

## 63. Restore original Istio security policies

Delete the temporary permissive policy:

```bash
kubectl delete peerauthentication otel-lab-permissive \
  -n "$APP_NS" \
  --ignore-not-found
```

Restore backed-up policies:

```bash
for file in backup/authorizationpolicy-*.json \
            backup/requestauthentication-*.json \
            backup/peerauthentication-*.json; do
  [ -f "$file" ] && kubectl apply -f "$file"
done
```

Validate:

```bash
kubectl get authorizationpolicy -n "$APP_NS"
kubectl get requestauthentication -n "$APP_NS"
kubectl get peerauthentication -n "$APP_NS"
```

Test the original security behavior again:

```bash
curl -i "http://${GATEWAY_IP}"
```

If the original lab required JWT, the expected unauthenticated response may again be `403`.

---

## 64. Remove the tracing stack

```bash
kubectl delete namespace "$OTEL_NS"
```

Validate:

```bash
kubectl get namespace "$OTEL_NS" 2>/dev/null || echo "Namespace deleted"
```

---

# Part 13 — Technical Summary

## 65. Responsibilities by component

| Component | Responsibility |
|---|---|
| Application instrumentation | Creates application and business spans |
| Context propagator | Transfers trace context across boundaries |
| OpenTelemetry SDK | Controls span creation, sampling and export |
| OpenTelemetry Collector | Receives, processes and routes telemetry |
| OTLP | Standard telemetry transport protocol |
| Zipkin receiver | Accepts Zipkin-compatible trace data |
| Jaeger | Stores, searches and visualizes traces |
| Istio | Remains installed; security enforcement is disabled for the lab |
| Envoy tracing | Optional network-level tracing source |

---

## 66. What students should remember

1. The Collector does not automatically instrument an application.
2. Environment variables configure an existing SDK or agent; they do not install one.
3. A complete distributed trace requires context propagation.
4. The same business ID does not guarantee the same trace.
5. OTLP is the preferred vendor-neutral transport.
6. The Collector can receive multiple protocols and normalize telemetry.
7. `memory_limiter` should be placed early in the pipeline.
8. Kubernetes enrichment should occur before tail sampling.
9. Tail sampling is powerful but requires trace affinity when scaled.
10. Jaeger all-in-one is suitable for labs, not high-volume production.
11. Istio security and OpenTelemetry are separate concerns.
12. Disabling lab security does not mean production telemetry should be unsecured.
13. Observability failure should not normally stop the business transaction.
14. Traces must be correlated with logs, metrics and business context.

---

## 67. Final architecture statement

```text
Istio remains installed in AKS,
but it does not enforce JWT or authorization for the lab namespace.

Applications and test generators send telemetry directly to OpenTelemetry.

OpenTelemetry Collector receives, enriches, samples and routes traces.

Jaeger provides investigation and visualization.
```

Final message for students:

> Building a distributed system is only half the work. The other half is preserving enough context to understand it when a request becomes slow, fragmented or fails.

---

# Appendix A — Quick validation checklist

```bash
# Istio installed
kubectl get pods -n istio-system

# No namespace authorization/JWT enforcement
kubectl get authorizationpolicy -n "$APP_NS"
kubectl get requestauthentication -n "$APP_NS"

# Permissive mTLS
kubectl get peerauthentication -n "$APP_NS"

# Collector and Jaeger healthy
kubectl get pods -n "$OTEL_NS"

# Collector logs
kubectl logs deployment/otel-collector -n "$OTEL_NS" --tail=100

# Jaeger logs
kubectl logs deployment/jaeger -n "$OTEL_NS" --tail=100

# OTLP service ports
kubectl get svc otel-collector -n "$OTEL_NS"

# Generate traces
kubectl delete job telemetrygen-traces -n "$OTEL_NS" --ignore-not-found
kubectl apply -f manifests/telemetrygen.yaml
kubectl logs job/telemetrygen-traces -n "$OTEL_NS"
```

---

# Appendix B — Minimal troubleshooting runbook

## No traces from application

```text
1. Confirm SDK or agent exists.
2. Confirm OTEL endpoint and protocol.
3. Confirm DNS resolution.
4. Confirm TCP connectivity.
5. Confirm Collector accepted spans.
6. Confirm processors did not drop the trace.
7. Confirm exporter sent spans.
8. Confirm Jaeger time range and service name.
```

## Collector CrashLoopBackOff

```bash
kubectl describe pod -n "$OTEL_NS" -l app.kubernetes.io/name=otel-collector
kubectl logs deployment/otel-collector -n "$OTEL_NS" --previous
```

## Jaeger unavailable

```bash
kubectl get pods,svc,endpoints -n "$OTEL_NS"
kubectl logs deployment/jaeger -n "$OTEL_NS" --tail=150
```

## 403 from ingress

```bash
kubectl get authorizationpolicy -A
kubectl get requestauthentication -A
kubectl logs <pod> -n "$APP_NS" -c istio-proxy --tail=100
```

## Sidecar cannot reach Collector

```bash
kubectl get destinationrule -A -o yaml | grep -n "ISTIO_MUTUAL" -B8 -A10
kubectl logs <pod> -n "$APP_NS" -c istio-proxy --tail=150
```

---

# Appendix C — Official references

- OpenTelemetry Collector overview: https://opentelemetry.io/docs/collector/
- Collector configuration: https://opentelemetry.io/docs/collector/configuration/
- Collector troubleshooting: https://opentelemetry.io/docs/collector/troubleshooting/
- Collector internal telemetry: https://opentelemetry.io/docs/collector/internal-telemetry/
- OpenTelemetry Kubernetes documentation: https://opentelemetry.io/docs/platforms/kubernetes/
- OpenTelemetry automatic instrumentation: https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- OTLP specification: https://opentelemetry.io/docs/specs/otlp/
- Tail sampling processor: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor
- Telemetrygen: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/telemetrygen
- Jaeger architecture: https://www.jaegertracing.io/docs/latest/architecture/
- Istio AuthorizationPolicy: https://istio.io/latest/docs/reference/config/security/authorization-policy/
- Istio PeerAuthentication: https://istio.io/latest/docs/reference/config/security/peer_authentication/
- Istio distributed tracing overview: https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/
