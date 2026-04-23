# Task2: динамическое масштабирование контейнеров

## Артефакты

- `01-deployment.yaml` — тестовое приложение `ghcr.io/yandex-practicum/scaletestapp:latest`.
- `02-service.yaml` — `NodePort`-сервис для доступа к приложению и Prometheus scrape.
- `03-hpa-memory.yaml` — HPA по memory utilization `80%`, `1..10` реплик.
- `04-service-monitor.yaml` — сбор `/metrics` через Prometheus Operator.
- `05-prometheus-adapter-values.yaml` — mapping `http_requests_total` → `http_requests_per_second`.
- `06-hpa-rps.yaml` — HPA по RPS на один pod, target `10`.
- `locustfile.py` — нагрузочный сценарий для `/`.

## Evidence

- `evidence/01-app-running.png` — приложение, pod и NodePort service запущены.
- `evidence/02-memory-hpa-before.png` — HPA по памяти до нагрузки.
- `evidence/03-memory-hpa-scaled.png` — HPA по памяти увеличил replicas.
- `evidence/04-memory-hpa-details.png` — детали HPA по памяти и `ValidMetricFound`.
- `evidence/05-prometheus-targets.png` — Prometheus target `scaletestapp` в состоянии `UP`.
- `evidence/06-prometheus-http-requests-total.png` — Prometheus видит `http_requests_total`.
- `evidence/07-prometheus-rate.png` — Prometheus считает `sum(rate(http_requests_total[1m]))`.
- `evidence/08-custom-metric-rps.png` — Custom Metrics API отдаёт `http_requests_per_second`.
- `evidence/09-rps-hpa-before.png` — HPA по RPS до нагрузки.
- `evidence/10-rps-hpa-watch.png` — HPA по RPS масштабируется под Locust-нагрузкой.
- `evidence/11-rps-hpa-scaled.png` — deployment масштабирован под RPS-нагрузкой.
- `evidence/12-locust-ui-running.png` — Locust UI с нагрузкой на `/`.

## Часть 1: HPA по памяти

```bash
minikube start --addons=metrics-server
kubectl apply -f Task2/01-deployment.yaml
kubectl apply -f Task2/02-service.yaml
kubectl apply -f Task2/03-hpa-memory.yaml
kubectl rollout status deployment/scaletestapp
kubectl get hpa scaletestapp-hpa
minikube service scaletestapp --url
```

Нагрузку запускать из директории `Task2`:

```bash
locust --host <minikube-service-url>
```

Для evidence сохранить скриншот Kubernetes dashboard или лог:

```bash
kubectl get hpa scaletestapp-hpa -w
kubectl get deployment scaletestapp
```

## Часть 2: HPA по RPS

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

kubectl apply -f Task2/04-service-monitor.yaml
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  -f Task2/05-prometheus-adapter-values.yaml

kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl apply -f Task2/06-hpa-rps.yaml
kubectl get hpa scaletestapp-hpa
```

Prometheus UI:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Для evidence сделать скриншоты:

- Prometheus `Status → Targets`: target `scaletestapp` в состоянии `UP`.
- Prometheus `Graph`: `http_requests_total` или `rate(http_requests_total[1m])`.
- Kubernetes dashboard или `kubectl get hpa scaletestapp-hpa -w`, где видно увеличение replicas.
- Locust UI, где видно нагрузку на `/`, RPS и процент ошибок.

## Статическая проверка

```bash
kubectl apply --dry-run=client -f Task2/01-deployment.yaml
kubectl apply --dry-run=client -f Task2/02-service.yaml
kubectl apply --dry-run=client -f Task2/03-hpa-memory.yaml
kubectl apply --dry-run=client -f Task2/06-hpa-rps.yaml
kubectl explain hpa.spec.metrics
```

`04-service-monitor.yaml` проверяется после установки `kube-prometheus-stack`, потому что `ServiceMonitor` является CRD Prometheus Operator.
