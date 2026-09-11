# Observabilidad (APM) — facturero

Monitoreo de tiempos de respuesta, errores y throughput de los servicios
(Node + Hono + Sequelize + MySQL) usando OpenTelemetry + SigNoz. Sin logs en
esta iteración (solo traces + métricas RED).

Es un repo independiente, como `mysql-basic` y `rabbitmq-service`: solo
manifiestos de K8s, desplegado con un workflow de GitHub Actions sobre el nodo
de observabilidad.

## Arquitectura

```
App (Node) ──OTLP HTTP 4318──▶ OTel Collector (DaemonSet en el cluster del CRM, ns observability)
                                   │ k8sattributes + batch + spanmetrics
                                   ▼ OTLP gRPC 4317
                            192.168.100.183:30017 (NodePort → collector de SigNoz)
                                   │
                              SigNoz (k3s standalone .183, ns signoz)
                              clickhouse + query-service + frontend + collector
```

- **SigNoz vive en el nodo de observabilidad `192.168.100.183`** (k3s
  standalone), desplegado por CI/CD con un runner **repo-level** (label `obs`).
  No usar el cluster del CRM para el backend de SigNoz: allí (`.149`) solo corre
  el **DaemonSet intermedio** `03-otel-collector.yaml`, que reenvía por LAN a
  `192.168.100.183:30017`.
- Cada servicio inicia el SDK de OpenTelemetry (solo si `OTEL_EXPORTER_OTLP_ENDPOINT`
  está definida) e instrumenta HTTP (span por request), Sequelize y mysql2
  (span por query SQL). El endpoint de los servicios sigue siendo el DaemonSet
  local: `http://otel-collector.observability.svc.cluster.local:4318`.
- El gateway además instrumenta `undici` (fetch) para propagar el `traceparent`
  a los servicios downstream y así tener trazas extremo a extremo.
- El collector intermedio deriva métricas RED (rate / errors / duration) de los
  spans con el conector `spanmetrics`; no hace falta escribir código de métricas.

## Estructura

```
k8s/
  00-namespaces.yaml           # namespaces observability + signoz
  01-signoz-crds.yaml          # CRDs del clickhouse-operator
  02-signoz.yaml               # stack SigNoz completo (helm template)
  03-otel-collector.yaml       # DaemonSet OTel intermedio PARA EL CLUSTER .149
.github/workflows/deploy.yaml  # apply 00→01→02 en el runner obs (.183)
```

Los manifests de SigNoz se generaron del chart oficial con:

```bash
helm repo add signoz https://charts.signoz.io
helm template signoz signoz/signoz -n signoz --output-dir ./tmp-signoz
```

y se consolidaron en `02-signoz.yaml` (orden lógico: clickhouse-operator →
zookeeper → clickhouse → otel-collector → query-service → job de migración).
En la consolidación se corrigieron los defectos del chart crudo: namespace
`signoz` en todos los recursos, **los 5 ConfigMaps del clickhouse-operator** y
`config.d/listen.xml` en el CHI. Para actualizar SigNoz a una versión nueva:
regenera el template y consolida igual que antes.

## Despliegue

### Automático (preferido)

Push a `master` que toque `k8s/**` (o `workflow_dispatch`) dispara el workflow
`Deploy observability (SigNoz) to observability node`, que corre en el runner
repo-level `obs` de `.183` (los labels de los runners están disjuntos: `.149`
sirve servicios con `self-hosted`, `.183` sirve solo observability con `obs`).
Aplica en orden `00 → 01 → 02` y espera los rollouts + migración. **NO aplica
`03`**: ese DaemonSet es del cluster del CRM y se aplica a mano allí.

**Nota:** el job exporta `KUBECONFIG=/home/obs/.kube/config`. El `kubectl`
embebido de k3s ignora `~/.kube/config` y sin la variable intenta
`/etc/rancher/k3s/k3s.yaml` (root-only) → `permission denied`. No quites ese
`env` del workflow.

### Manual (.183)

```bash
export KUBECONFIG=/home/obs/.kube/config
kubectl apply -f k8s/00-namespaces.yaml
kubectl apply -f k8s/01-signoz-crds.yaml
kubectl apply -f k8s/02-signoz.yaml
kubectl rollout status deployment/signoz-clickhouse-operator -n signoz
kubectl rollout status statefulset/signoz-zookeeper -n signoz
kubectl rollout status statefulset/signoz -n signoz
kubectl wait --for=condition=Ready pod/chi-signoz-clickhouse-cluster-0-0-0 -n signoz --timeout=300s
kubectl wait --for=condition=complete job/signoz-telemetrystore-migrator -n signoz --timeout=300s
```

### Manual (cluster del CRM, solo DaemonSet intermedio)

```bash
kubectl apply -f k8s/03-otel-collector.yaml
kubectl rollout status daemonset/otel-collector -n observability
```

## UI de SigNoz y puertos

- UI: `http://192.168.100.183:30012`
- OTLP gRPC: `192.168.100.183:30017` (NodePort fijo del Service `signoz-otel-collector`)
- OTLP HTTP: `192.168.100.183:30018`

Si algún día cambian los `nodePort`:

```bash
kubectl get svc signoz-otel-collector -n signoz -o jsonpath='{.spec.ports[?(@.port==4317)].nodePort}'
```

Admin de SigNoz: `admin@facturero.local` (org `facturero`).

### Env vars por servicio (ya agregadas en cada `k8s/deployment.yaml`)

- `OTEL_SERVICE_NAME=<nombre-servicio>`
- `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability.svc.cluster.local:4318`

## Verificación

```bash
# 1. Collector intermedio (.149) sin errores de export
kubectl logs -n observability -l app=otel-collector --tail=50

# 2. Una petición real por el gateway
curl -i https://<gateway>/health

# 3. En SigNoz (http://192.168.100.183:30012) → Traces → buscar por el
#    trace-id del response o filtrar por servicio. Debería verse:
#    gateway → servicio → mysql SELECT.

# 4. En SigNoz → Dashboard → "Overview Metrics" (RED por servicio/ruta).
```

## Traces end-to-end

El gateway inyecta `traceparent` en las llamadas downstream vía la
instrumentación de `undici`. Los servicios reciben ese contexto (default
propagator W3C) y continúan la traza. Cada petición produce una cadena:
`HTTP <método> <ruta>` (gateway) → `HTTP GET ...` (servicio) → `SELECT ...` (mysql2).

## Desactivar

Quita las env vars `OTEL_*` de los deployments y redepliega; el SDK no inicia
si falta `OTEL_EXPORTER_OTLP_ENDPOINT`. Para apagar el backend por completo:
`kubectl delete -f k8s/02-signoz.yaml` (conserva los PVCs de clickhouse).