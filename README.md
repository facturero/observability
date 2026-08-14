# Observabilidad (APM) — facturero

Monitoreo de tiempos de respuesta, errores y throughput de los 7 servicios
(Node + Hono + Sequelize + MySQL) usando OpenTelemetry + SigNoz. Sin logs en
esta iteración (solo traces + métricas RED).

Es un proyecto del monorepo independiente, como `mysql-basic` y
`rabbitmq-service`: solo manifiestos de K8s, desplegado con un workflow de
GitHub Actions.

## Arquitectura

```
App (Node) ──OTLP HTTP 4318──▶ OTel Collector (DaemonSet, ns observability)
                                   │ k8sattributes + batch + spanmetrics
                                   ▼ OTLP gRPC 4317
                             SigNoz (ns signoz)
                             clickhouse + query-service + frontend + collector
```

- Cada servicio inicia el SDK de OpenTelemetry (solo si `OTEL_EXPORTER_OTLP_ENDPOINT`
  está definida) e instrumenta HTTP (span por request), Sequelize y mysql2
  (span por query SQL).
- El gateway además instrumenta `undici` (fetch) para propagar el `traceparent`
  a los servicios downstream y así tener trazas extremo a extremo.
- El collector deriva métricas RED (rate / errors / duration) de los spans con
  el conector `spanmetrics`; no hace falta escribir código de métricas.

## Estructura

```
k8s/
  00-namespaces.yaml           # namespaces observability + signoz
  01-signoz-crds.yaml          # CRDs del clickhouse-operator
  02-signoz.yaml               # stack SigNoz completo (helm template)
  03-otel-collector.yaml       # DaemonSet OTel intermedio + RBAC + Service
.github/workflows/deploy.yaml  # apply en orden en el runner self-hosted
```

Los manifests de SigNoz se generaron del chart oficial con:

```bash
helm repo add signoz https://charts.signoz.io
helm template signoz signoz/signoz -n signoz --output-dir ./tmp-signoz
```

y se consolidaron en `02-signoz.yaml` (orden lógico: clickhouse-operator →
zookeeper → clickhouse → otel-collector → query-service → job de migración).
Para actualizar SigNoz a una versión nueva: regenera el template y consolida
igual que antes.

## Despliegue

### Automático (preferido)

Push a `master` que toque `k8s/**` (o `workflow_dispatch`) dispara el workflow
`Deploy observability to k3s` (runner self-hosted), que aplica en orden:
namespaces → CRDs → SigNoz → OTel Collector y espera los rollouts.

### Manual

```bash
kubectl apply -f k8s/00-namespaces.yaml
kubectl apply -f k8s/01-signoz-crds.yaml
kubectl apply -f k8s/02-signoz.yaml
kubectl apply -f k8s/03-otel-collector.yaml

kubectl rollout status deployment/signoz-clickhouse-operator -n signoz
kubectl rollout status statefulset/signoz-zookeeper -n signoz
kubectl rollout status statefulset/signoz -n signoz
kubectl rollout status daemonset/otel-collector -n observability
kubectl wait --for=condition=complete job/signoz-telemetrystore-migrator -n signoz --timeout=300s
```

### UI de SigNoz

```bash
kubectl port-forward -n signoz svc/signoz 3301:3301
# → http://localhost:3301
```

### Env vars por servicio (ya agregadas en cada `k8s/deployment.yaml`)

- `OTEL_SERVICE_NAME=<nombre-servicio>`
- `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability.svc.cluster.local:4318`

## Verificación

```bash
# 1. Collector sin errores de export
kubectl logs -n observability -l app=otel-collector --tail=50

# 2. Una petición real por el gateway
curl -i https://<gateway>/health

# 3. En SigNoz → Traces → buscar por el trace-id del response (header traceparent)
#    o filtrar por servicio. Debería verse: gateway → servicio → mysql SELECT.

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
