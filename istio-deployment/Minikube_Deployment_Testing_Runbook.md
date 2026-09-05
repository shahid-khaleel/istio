# Minikube Deployment & Testing Runbook
### Istio + OpenTelemetry + Jaeger + Kiali — full stack, one sample app, end-to-end verification

This mirrors Part 7 of the book, adapted specifically for minikube's networking model (no cloud LoadBalancer, so `minikube tunnel` replaces it) and its resource constraints (sized explicitly so the whole stack actually fits).

Run every command as-is, in order. Where a step needs its own terminal (long-running), that's called out.

**Diagram:** [Trace Join Pipeline](https://claude.ai/code/artifact/e8cad4e4-bdd9-473c-aa51-f365dce876c8) — visualizes the full request/telemetry path (Step 8.5) and the trace-propagation bug and fix (Step 8.6).

---

## 0. Prerequisites

```bash
minikube version
kubectl version --client
helm version
istioctl version --remote=false
```

If `istioctl` isn't installed:
```bash
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PWD/istio-*/bin:$PATH"
```

---

## 1. Start minikube sized for this stack

Istio + OTel Collector + Jaeger + Kiali + Prometheus + a sample app is real weight for a laptop. Undersizing this is the #1 cause of pods stuck `Pending`.

```bash
minikube start \
  --driver=docker \
  --cpus=6 \
  --memory=12288 \
  --disk-size=40g \
  --kubernetes-version=stable

minikube addons enable metrics-server
```

If your machine can't spare 6 CPUs/12GB, drop Grafana and use Jaeger's `allInOne` (already the plan below) — this is the minimum realistic floor for all four tools running together without constant evictions.

**Windows + Docker Desktop:** `minikube start` can fail immediately with `MK_USAGE: Docker Desktop has only <N>MB memory but you specified 12288MB`, even on a machine with plenty of RAM. Docker Desktop's WSL2 backend defaults to ~50% of host RAM unless told otherwise. Raise it via `%USERPROFILE%\.wslconfig` (create it if missing):
```
[wsl2]
memory=14GB
```
Then `wsl --shutdown` (stops all WSL distros/containers momentarily) and restart Docker Desktop. Re-check with `docker info --format '{{.MemTotal}}'` before retrying `minikube start`.

Check node capacity before moving on:
```bash
kubectl describe node minikube | grep -A5 "Allocated resources"
```

---

## 2. Install Istio

```bash
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
kubectl get pods -n istio-system
```
Expect `istiod`, `istio-ingressgateway`, `istio-egressgateway` all `Running`.

### 2.1 Open the tunnel (separate terminal — leave running for the whole session)

Minikube can't provision real cloud LoadBalancers, so `istio-ingressgateway`'s `EXTERNAL-IP` stays `<pending>` without this:

```bash
minikube tunnel
```
Keep this terminal open for every later step that hits the ingress gateway. Verify:
```bash
kubectl get svc istio-ingressgateway -n istio-system
# EXTERNAL-IP should now show 127.0.0.1
```

---

## 3. Prometheus + Grafana (Istio's bundled addons — fine for this lab)

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/addons/grafana.yaml
kubectl rollout status deployment/prometheus -n istio-system
```

---

## 4. OpenTelemetry Operator + Collector

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm install otel-operator open-telemetry/opentelemetry-operator \
  --namespace observability --create-namespace \
  --set manager.collectorImage.repository=otel/opentelemetry-collector-k8s \
  --set admissionWebhooks.certManager.enabled=false

kubectl rollout status deployment/otel-operator-opentelemetry-operator -n observability
```
The chart defaults to using cert-manager to issue the webhook's TLS cert. Since cert-manager isn't installed as part of this lab, `--set admissionWebhooks.certManager.enabled=false` falls back to the chart's built-in Helm-generated self-signed cert (fine for a lab; without this flag the install fails with `no matches for kind "Certificate" in version "cert-manager.io/v1"`).

Save as `otel-collector.yaml`:
```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
  namespace: observability
spec:
  mode: deployment
  replicas: 1
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch: {}
      memory_limiter:
        check_interval: 5s
        limit_mib: 256
    exporters:
      otlp/jaeger:
        endpoint: jaeger-lab-collector.observability.svc.cluster.local:4317
        tls:
          insecure: true
      debug:
        verbosity: basic
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/jaeger, debug]
```
```bash
kubectl apply -f otel-collector.yaml
kubectl get otelcol -n observability
```

---

## 5. Jaeger (Operator + all-in-one — right-sized for a lab)

The Jaeger operator's webhook needs cert-manager to issue its TLS secret (it isn't installed by default in this lab). Install it first and wait for it to be ready:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl rollout status deployment/cert-manager -n cert-manager
kubectl rollout status deployment/cert-manager-webhook -n cert-manager
kubectl rollout status deployment/cert-manager-cainjector -n cert-manager
```

```bash
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml -n observability
kubectl rollout status deployment/jaeger-operator -n observability
```
If this manifest was applied *before* cert-manager existed, the `Certificate`/`Issuer` resources fail with `no matches for kind "Certificate"` and the operator pod hangs in `FailedMount` (missing secret `jaeger-operator-service-cert`). Re-run the `kubectl apply -f .../jaeger-operator.yaml` line above after cert-manager is ready — it's idempotent and will create just the missing pieces.

Separately, the release manifest currently pins its `kube-rbac-proxy` sidecar to `gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1`, a tag that registry no longer serves (`ErrImagePull`/`manifest unknown`). That sidecar only guards the metrics port and isn't needed for the CR to reconcile, so repoint it to the maintained fork:
```bash
kubectl set image deployment/jaeger-operator -n observability kube-rbac-proxy=quay.io/brancz/kube-rbac-proxy:v0.18.0
kubectl rollout status deployment/jaeger-operator -n observability
```
Note: after this image patch (or any deployment update) the old pod's leader-election lease can take up to its `leaseDurationSeconds` (~137s by default) to expire before the new pod takes over and starts reconciling `Jaeger` CRs — if `kubectl get jaeger -n observability` shows a blank `STATUS` right after applying, that's normal; give it ~2 minutes.

Save as `jaeger-lab.yaml`:
```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-lab
  namespace: observability
spec:
  strategy: allInOne
  allInOne:
    options:
      memory:
        max-traces: 50000
```
```bash
kubectl apply -f jaeger-lab.yaml
kubectl get pods -n observability | grep jaeger
```

---

## 6. Kiali

```bash
helm repo add kiali https://kiali.org/helm-charts
helm repo update
helm install kiali-server kiali/kiali-server \
  --namespace istio-system \
  --set auth.strategy=anonymous \
  --set external_services.prometheus.url=http://prometheus.istio-system:9090 \
  --set external_services.tracing.enabled=true \
  --set external_services.tracing.in_cluster_url=http://jaeger-lab-query.observability:16685 \
  --set external_services.tracing.use_grpc=true

kubectl rollout status deployment/kiali -n istio-system
```
`auth.strategy=anonymous` is lab-only — never in production (see Part 5.5 for OIDC).

---

## 7. Deploy the sample app (Istio's `bookinfo` — multi-language on purpose)

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get pods
```
Wait for all `productpage`, `details`, `reviews-v1/v2/v3`, `ratings` pods `Running` with 2/2 containers (app + Envoy sidecar).

### 7.1 Auto-instrument `productpage` (Python) with OTel

```yaml
# instrumentation.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation
  namespace: default
spec:
  exporter:
    endpoint: http://otel-gateway-collector.observability.svc.cluster.local:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_always_on
```
```bash
kubectl apply -f instrumentation.yaml
kubectl patch deployment productpage-v1 -p \
  '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":"true"}}}}}'
kubectl rollout status deployment/productpage-v1
```
**Port note:** use **4318** (the collector's HTTP receiver), not 4317 (gRPC). Python's OTel auto-instrumentation defaults to the OTLP/HTTP protobuf exporter — pointing it at the gRPC port causes every span export to fail with `415 Unsupported Media Type` (visible as 100% error rate on `otel-gateway-collector` in Kiali/Prometheus, even though the app itself keeps serving `200`s to real users — don't mistake that for an app-level outage). After pointing at 4318 you may still see `POST /v1/logs` return `404` in the productpage container logs — that's benign: the SDK also auto-exports logs via OTLP, but the collector config in this runbook only defines a `traces` pipeline, so it correctly declines the signal it isn't configured for.

**Windows/PowerShell note:** PowerShell mangles embedded double quotes when passing a `-p '<json>'` argument to a native exe like `kubectl` (even from a here-string) — it fails with `invalid character 's' looking for beginning of object key string`. Sidestep it by writing the patch to a file and using `--patch-file` instead:
```powershell
$patch = '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":"true"}}}}}'
Set-Content -Path patch-inject-python.json -Value $patch
kubectl patch deployment productpage-v1 --patch-file patch-inject-python.json
```
This applies to every `kubectl patch -p '<json>'` call later in this runbook (Step 10) too.

---

## 8. Generate traffic through the actual ingress path

```bash
curl -s http://localhost/productpage | grep -o "<title>.*</title>"
for i in $(seq 1 50); do curl -s http://localhost/productpage > /dev/null; done
```
(`localhost` works because `minikube tunnel` is mapping the ingress gateway's `EXTERNAL-IP` to `127.0.0.1`.)

---

## 8.5 Enable Envoy-level distributed tracing (gap in Istio's `demo` profile)

The `demo` profile installed in Step 2 registers an `otel-tracing` extension provider *template* in `meshConfig.extensionProviders`, but two things stop it from actually working out of the box:
1. Its `service` field defaults to `opentelemetry-collector.observability.svc.cluster.local` — the OTel Helm chart's operator name, not our actual collector Service (`otel-gateway-collector`, from the `OpenTelemetryCollector` CR in Step 4).
2. No `Telemetry` resource ever *enables* the provider — registering it in meshConfig only makes it available, it doesn't turn on trace export.

Without this, Envoy never emits spans anywhere, and Step 9's "Jaeger has a complete trace" check will only ever show a single app-level span — Step 10's break/restore exercise depends on Envoy-hop spans existing independently of the app, so this step is required, not optional polish.

Fix the provider's service name via a full overlay (do **not** use `istioctl install --set meshConfig.extensionProviders[N].field=...` — indexed `--set` on this array replaces the *entire* array with a single bare element, silently deleting the other providers and even the modified entry's own `name`/`port` fields):
```yaml
# istio-tracing-overlay.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  meshConfig:
    extensionProviders:
    - name: otel
      envoyOtelAls:
        port: 4317
        service: opentelemetry-collector.observability.svc.cluster.local
    - name: skywalking
      skywalking:
        port: 11800
        service: tracing.istio-system.svc.cluster.local
    - name: otel-tracing
      opentelemetry:
        port: 4317
        service: otel-gateway-collector.observability.svc.cluster.local
```
```bash
istioctl install -f istio-tracing-overlay.yaml -y
```
Then enable it mesh-wide (100% sampling is lab-only — never in production):
```yaml
# telemetry-tracing.yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default-tracing
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: otel-tracing
    randomSamplingPercentage: 100.0
```
```bash
kubectl apply -f telemetry-tracing.yaml
```
Envoy sidecars pick this up via xDS within seconds — no pod restarts needed. Re-run Step 8's traffic loop afterward; `http://localhost:16686/api/services` should then list `productpage.default`, `details.default`, `reviews.default`, `ratings.default`, `istio-ingressgateway.istio-system` (Envoy-reported, dotted `service.namespace` names) alongside `productpage-v1` (the app-level OTel SDK name).

---

## 8.6 Fix `productpage`'s app-level spans joining the Envoy trace (source-level bug in the sample app)

Even after 8.5, the app-level `productpage-v1` spans and the Envoy-hop spans landed in Jaeger under *separate* trace IDs — the Envoy-to-Envoy chain was correctly linked end-to-end (verified via `/api/traces/<id>`), but the app's own Flask span never joined it.

**Root cause**, confirmed by temporarily adding a diagnostic Envoy access log format (`accessLogFormat` in `meshConfig`, capturing `%REQ(TRACEPARENT)%`/`%REQ(X-B3-TRACEID)%`) and looking up the exact resulting trace ID directly in Jaeger's `/api/traces/<id>`: Envoy genuinely sends a valid **W3C `traceparent`** header on the inbound request (`x-b3-traceid` is always empty) — so the header format isn't the problem. The actual bug is in bookinfo's own `productpage.py` (baked into the `istio/examples-bookinfo-productpage-v1:1.20.1` image), which at **module import time** unconditionally runs:
```python
set_global_textmap(B3MultiFormat())
```
This silently overrides whatever propagator the `Instrumentation` CR configures via `OTEL_PROPAGATORS` — with no guard against being called after the auto-instrumentation agent already configured its own (unlike `trace.set_tracer_provider()` two lines below it, which *is* guarded and correctly logs `WARNING: Overriding of current TracerProvider is not allowed` when bookinfo's own call is rejected — that warning in the pod logs is expected/harmless, not the bug). Since Envoy sends `traceparent`, not `x-b3-*`, Flask's own context extraction always comes up empty and starts a new root trace, every time, regardless of any `Instrumentation` CR setting.

Confirmed via the installed package list (`kubectl exec ... -- pip list`) that `opentelemetry-instrumentation-flask` and a full auto-instrumentation set are present and correctly loaded — this is purely the propagator getting overridden after the fact, not a missing/misconfigured instrumentor.

**Fix** (patch the image; this cannot be fixed via `Instrumentation` CR or env vars, since the override happens in application code that runs after them):
```bash
curl -s https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/src/productpage/productpage.py -o productpage_patched.py
# Edit productpage_patched.py: comment out the line `set_global_textmap(B3MultiFormat())`
# (leave everything else, including the local `propagator = B3MultiFormat()` object used
# later in getForwardHeaders() for explicit b3 header handling, untouched)
```
```dockerfile
# Dockerfile.productpage-patched
FROM docker.io/istio/examples-bookinfo-productpage-v1:1.20.1
COPY productpage_patched.py /opt/microservices/productpage.py
```
```bash
minikube image build -t productpage-v1-patched:1.20.1 -f Dockerfile.productpage-patched .
kubectl set image deployment/productpage-v1 productpage=docker.io/library/productpage-v1-patched:1.20.1
kubectl rollout status deployment/productpage-v1
```
Verified fix: after patching, `curl`-generated traces consistently show 11-13 spans spanning `productpage-v1`, `istio-ingressgateway.istio-system`, `productpage.default`, `details.default`, `reviews.default` (and `ratings.default` when the reviews version called happens to use it) — all under one shared `TraceId`, across repeated requests. (An occasional 1-span `productpage-v1`-only trace is expected noise from the collector's own internal OTLP export calls under 100% sampling, not a regression.)

---

## 9. Verify every layer independently

**Envoy/Istio metrics** (confirms the mesh data plane is working):
```bash
kubectl exec -it deploy/productpage-v1 -c istio-proxy -- \
  curl -s localhost:15000/stats/prometheus | grep istio_requests_total | head -5
```

**Prometheus has the data**:
```bash
kubectl port-forward -n istio-system svc/prometheus 9090:9090 &
# open http://localhost:9090, query: istio_requests_total{destination_service=~"productpage.*"}
```

**Kiali graph shows traffic**:
```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001 &
# open http://localhost:20001 -> Graph -> namespace: default
```
Confirm: nodes for `productpage`, `details`, `reviews`, `ratings`; animated edges; no red error borders.

**Jaeger has a complete trace**:
```bash
kubectl port-forward -n observability svc/jaeger-lab-query 16686:16686 &
# open http://localhost:16686 -> Service: productpage.default -> Find Traces
```
Open one trace. You should see **both** app-level spans (from the OTel Python auto-instrumentation) **and** Envoy-hop spans, all under one `TraceId`.

---

## 10. Break trace propagation on purpose (do this — it's the highest-value exercise in the whole lab)

```bash
kubectl patch deployment productpage-v1 -p \
  '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":null}}}}}'
kubectl rollout status deployment/productpage-v1
for i in $(seq 1 10); do curl -s http://localhost/productpage > /dev/null; done
```
**Windows/PowerShell:** as noted in Step 7.1, the inline `-p '<json>'` form fails under PowerShell. Use `--patch-file` instead:
```powershell
'{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":null}}}}}' | Set-Content patch-remove-inject-python.json
kubectl patch deployment productpage-v1 --patch-file patch-remove-inject-python.json
kubectl rollout status deployment/productpage-v1
```

Search Jaeger again. Traces from `productpage` now either vanish or arrive fragmented — you're looking at exactly the failure mode described in Part 6.2. Re-apply the annotation to restore it:
```bash
kubectl patch deployment productpage-v1 -p \
  '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":"true"}}}}}'
```
```powershell
# Windows/PowerShell: reuse patch-inject-python.json from Step 7.1
kubectl patch deployment productpage-v1 --patch-file patch-inject-python.json
```

**Observed instead, on this stack:** the Envoy-hop trace chain stayed **fully linked** across 5 fresh traces (ingress → productpage → details/reviews → ratings, one shared `TraceId` each) even with the Python instrumentation removed — the described break didn't reproduce here. Cause, confirmed via `getForwardHeaders()` in `productpage.py` (see also 8.6): bookinfo explicitly copies the **raw `traceparent`/`tracestate` header values by name** from each incoming request onto its outgoing calls — unconditional application code, entirely independent of whatever OTel auto-instrumentation is or isn't injected via the annotation. That explicit copy is what's actually sustaining Envoy-to-Envoy correlation here, not the OTel Python SDK. (Earlier notes in this runbook attributed this to B3 header forwarding — that was a misreading before 8.6's investigation pinned down the actual mechanism; Envoy here sends W3C `traceparent`, not B3, and bookinfo's copy is a plain string passthrough, not routed through any OTel propagator.) Reproducing this exercise's intended effect would mean removing that hardcoded copy from `productpage.py` (the same file patched in 8.6) — out of scope for this exercise; left as an observation rather than a further fix.

---

## 11. Canary rollout — confirm it visually in Kiali

```yaml
# reviews-canary.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels: {version: v1}
  - name: v2
    labels: {version: v2}
  - name: v3
    labels: {version: v3}
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
  - route:
    - destination: {host: reviews, subset: v1}
      weight: 90
    - destination: {host: reviews, subset: v3}
      weight: 10
```
```bash
kubectl apply -f reviews-canary.yaml
for i in $(seq 1 100); do curl -s http://localhost/productpage > /dev/null; done
```
In Kiali's graph, select the `reviews` node's edges — the observed split should track your 90/10 weights over enough requests. Also check the `v3` subset's error rate (bookinfo's `v3` reviews calls `ratings`, so this exercises a real downstream dependency).

---

## Troubleshooting specific to minikube

| Symptom | Cause | Fix |
|---|---|---|
| `istio-ingressgateway` `EXTERNAL-IP` stuck `<pending>` | No `minikube tunnel` running | Start it in its own terminal, keep it open |
| Pods `Pending` with `Insufficient cpu`/`memory` | minikube under-sized | Re-run `minikube start` with higher `--cpus`/`--memory`, or drop Grafana |
| `curl http://localhost/productpage` connection refused | Tunnel not yet mapped, or gateway not ready | `kubectl get svc istio-ingressgateway -n istio-system` — wait for `EXTERNAL-IP` to show `127.0.0.1` |
| Kiali graph empty despite traffic | Prometheus URL wrong, or traffic sent before sidecar injection was labeled on `default` namespace | Re-check `kubectl get ns default --show-labels`; redeploy bookinfo after labeling |
| No traces in Jaeger at all | OTel Collector `otlp/jaeger` exporter endpoint typo, or Jaeger not `Running` yet | Check Collector pod logs: `kubectl logs -n observability deploy/otel-gateway-collector` |
| `minikube start` fails with `MK_USAGE: Docker Desktop has only <N>MB memory` | Docker Desktop's WSL2 backend defaults to ~50% of host RAM | Set `memory=` in `%USERPROFILE%\.wslconfig`, `wsl --shutdown`, restart Docker Desktop (see Step 1) |
| `helm install otel-operator`/`kubectl apply jaeger-operator.yaml` fails: `no matches for kind "Certificate" in version "cert-manager.io/v1"` | cert-manager isn't installed; both operators' webhooks expect it | For the OTel chart: `--set admissionWebhooks.certManager.enabled=false` (Step 4). For Jaeger: install cert-manager first, then re-apply (Step 5) |
| `jaeger-operator` pod stuck `FailedMount`: secret `jaeger-operator-service-cert` not found | Its `Certificate`/`Issuer` never got created (cert-manager missing or applied out of order) | Install cert-manager, then re-run `kubectl apply -f .../jaeger-operator.yaml` — idempotent (Step 5) |
| `jaeger-operator` pod `ErrImagePull` on its `kube-rbac-proxy` container | Manifest pins `gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1`, a tag that registry no longer serves | `kubectl set image deployment/jaeger-operator -n observability kube-rbac-proxy=quay.io/brancz/kube-rbac-proxy:v0.18.0` (Step 5) |
| `otel-gateway-collector` returns `415` on every span export; `productpage` shows a high error ratio in Kiali despite serving real users `200`s | `Instrumentation` CR's exporter endpoint uses port 4317 (gRPC) but Python's OTel SDK defaults to OTLP/HTTP | Point the endpoint at port 4318 instead (Step 7.1) |
| `productpage` app logs show `POST /v1/logs ... 404` repeatedly | SDK also auto-exports logs via OTLP; the collector config here only defines a `traces` pipeline | Benign — collector correctly declines a signal it isn't configured for (Step 7.1) |
| Jaeger has traces, but no Envoy-hop spans at all (only `productpage-v1`) | Istio's `demo` profile registers an `otel-tracing` extension provider but never enables it, and its default `service` name doesn't match this lab's collector | Fix the provider's service name via a full `istioctl install -f` overlay, then apply a `Telemetry` resource to enable it (Step 8.5) |
| `kubectl patch -p '<json>'` fails: `invalid character 's' looking for beginning of object key string` | PowerShell mangles embedded double quotes passed to a native exe, even from a here-string | Write the patch JSON to a file and use `--patch-file` instead (Steps 7.1, 10) |
| App-level `productpage-v1` spans never join the Envoy-hop trace, even with `Instrumentation` correctly configured | `productpage.py` (baked into the sample image) unconditionally calls `set_global_textmap(B3MultiFormat())` at import time, silently overriding whatever propagator `OTEL_PROPAGATORS` set — Envoy sends `traceparent`, not B3, so Flask's context extraction always comes up empty | Patch `productpage.py` to drop that call, rebuild the image with `minikube image build`, repoint the deployment at it (Step 8.6) |
