# Setup Wizard: OpenTelemetry Astronomy Shop for Splunk

Use this file as source content for a future UI wizard. It keeps the setup flow
short and separates the Docker and minikube paths.

## Inputs

| Input | Required | Used by |
| --- | --- | --- |
| Splunk Observability Cloud realm | Yes | Docker, minikube |
| Splunk Observability Cloud access token | Yes | Docker, minikube |
| Splunk Platform HEC endpoint | Optional | Docker, minikube |
| Splunk Platform HEC token | Optional | Docker, minikube |
| Splunk Platform log index | Optional | Docker, minikube |
| Kubernetes cluster name | minikube only | minikube |

Treat all tokens as secrets. Do not write them to committed files.

## Path A: Docker

Prerequisites: Docker, Docker Compose, `git`, and at least 4 GB available RAM.

Clone the demo.

```bash
git clone https://github.com/chentaow-splunk/opentelemetry-demo.git
cd opentelemetry-demo
git checkout main
```

Set Splunk values.

```bash
export SPLUNK_ACCESS_TOKEN="<splunk-observability-access-token>"
export SPLUNK_REALM="<splunk-observability-realm>"
export SPLUNK_HEC_TOKEN="<splunk-platform-hec-token>"
export SPLUNK_HEC_URL="<splunk-platform-hec-endpoint>"
export SPLUNK_LOG_INDEX="astronomyshop"
export SPLUNK_MEMORY_TOTAL_MIB=1024
```

Use the Splunk Docker Compose file and start the demo.

```bash
cp splunk/docker-compose.yml docker-compose.yml
docker compose up --force-recreate --remove-orphans --detach
```

Open the demo.

- Web store: `http://localhost:8080/`
- Load generator: `http://localhost:8080/loadgen/`
- Feature flags: `http://localhost:8080/feature`

Stop the demo.

```bash
docker compose down
```

## Path B: Minikube

Prerequisites: Docker, `minikube`, `kubectl`, Helm, `git`, and at least 8 GB
available RAM.

Start minikube.

```bash
minikube start --cpus=6 --memory=8192 --disk-size=32g
```

Set Splunk values.

```bash
export SPLUNK_REALM="<splunk-observability-realm>"
export SPLUNK_ACCESS_TOKEN="<splunk-observability-access-token>"
export CLUSTER_NAME="minikube-otel-demo"
export SPLUNK_HEC_ENDPOINT="<splunk-platform-hec-endpoint>"
export SPLUNK_HEC_TOKEN="<splunk-platform-hec-token>"
export SPLUNK_LOG_INDEX="astronomyshop"
export SPLUNK_METRICS_INDEX="k8s-metrics"
```

Install the Splunk OpenTelemetry Collector.

```bash
helm repo add splunk-otel-collector-chart https://signalfx.github.io/splunk-otel-collector-chart
helm repo update

helm upgrade --install splunk-otel-collector \
  splunk-otel-collector-chart/splunk-otel-collector \
  --namespace splunk-otel \
  --create-namespace \
  --set splunkObservability.realm="${SPLUNK_REALM}" \
  --set splunkObservability.accessToken="${SPLUNK_ACCESS_TOKEN}" \
  --set splunkPlatform.endpoint="${SPLUNK_HEC_ENDPOINT}" \
  --set splunkPlatform.token="${SPLUNK_HEC_TOKEN}" \
  --set splunkPlatform.index="${SPLUNK_LOG_INDEX}" \
  --set splunkPlatform.metricsIndex="${SPLUNK_METRICS_INDEX}" \
  --set clusterName="${CLUSTER_NAME}"
```

For Splunk Observability Cloud only, remove the four `splunkPlatform.*` values
from the Helm command.

Deploy the demo.

```bash
git clone https://github.com/chentaow-splunk/opentelemetry-demo.git
cd opentelemetry-demo
git checkout main

kubectl create namespace otel-demo --dry-run=client -o yaml | kubectl apply -f -
kubectl apply --namespace otel-demo -f ./splunk/opentelemetry-demo.yaml
kubectl -n otel-demo get pods
```

Open the demo.

```bash
kubectl -n otel-demo port-forward svc/frontend-proxy 8080:8080
```

- Web store: `http://localhost:8080/`
- Load generator: `http://localhost:8080/loadgen/`
- Feature flags: `http://localhost:8080/feature`

Clean up.

```bash
kubectl delete --namespace otel-demo -f ./splunk/opentelemetry-demo.yaml
helm uninstall splunk-otel-collector --namespace splunk-otel
minikube delete
```

## Verify

Splunk Observability Cloud:

- Kubernetes Infrastructure shows the configured cluster name for minikube.
- APM shows services in the `opentelemetry-demo` namespace.
- Traces include `deployment.environment=development`.

Splunk Platform, when HEC is configured:

```text
index=<splunk-platform-log-index>
```

## Wizard Notes

- Offer Docker and minikube as the first choice.
- Generate one command path from the selected choice.
- Keep `splunk/docker-compose.yml` as the Docker source.
- Keep `splunk/opentelemetry-demo.yaml` as the minikube source.
- Do not ask users to run `splunk/update-demos.sh`; automation refreshes the
  generated Splunk files from upstream.

## References

- Splunk Docker guide:
  `https://lantern.splunk.com/Observability_Use_Cases/Optimize_Costs/Setting_up_the_OpenTelemetry_Demo_in_Docker`
- Splunk Kubernetes guide:
  `https://lantern.splunk.com/Observability_Use_Cases/Optimize_Costs/Setting_up_the_OpenTelemetry_Demo_in_Kubernetes`
- Splunk OpenTelemetry Collector Helm chart:
  `https://github.com/signalfx/splunk-otel-collector-chart`
