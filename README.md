# muster

cluster bootrapping


## Installation

Create the ArgoCD 'App of Apps':
```bash
$ kubectl apply -f app-of-apps.yml
```
```
application.argoproj.io/app-of-apps created
```

Confirm all apps are healthy:
```bash
$ kubectl -n argocd get application
```
```
NAME                          SYNC STATUS   HEALTH STATUS
app-of-apps                   Synced        Healthy
chess-analyzer                Synced        Healthy
crossplane-provider-configs   Synced        Healthy
crossplane-providers          Synced        Healthy
logger-speedrun               Synced        Healthy
observability                 Synced        Healthy
prometheus-exporter-weather   Synced        Healthy
security                      Synced        Healthy
```


## ArgoCD

Get ArgoCD admin password:
```bash
$ kubectl -n argocd \
    get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" \
| base64 -d
```

Port forward:
```bash
$ kubectl -n argocd \
    port-forward service/argo-cd-argocd-server \
    8080:http
```
```
http://127.0.0.1:8080
```

Username: `admin`
Password: `<as above>`


## Weather Exporter

Port forward:
```bash
$ kubectl -n datasources \
    port-forward service/prometheus-exporter-weather \
    2112:2112
```
```
http://127.0.0.1:2112/metrics
```


## Prometheus

Port forward:
```bash
$ kubectl -n observability \
    port-forward service/prometheus-server \
    9090:http
```
```
http://127.0.0.1:9090
```

Test query:
```
weather_temperature_actual_celcius
```


## Loki

Port forward:
```bash
$ kubectl -n observability \
    port-forward \
    service/loki 3100:3100
```

Test data:
```bash
$ curl -sX POST \
    http://127.0.0.1:3100/loki/api/v1/push \
    -H 'Content-Type: application/json' \
    --data-raw "
      {
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"testjob\"
            },
            \"values\": [
              [
                \"$(date +%s)000000000\",
                \"it works\"
              ]
            ]
          }
        ]
      }"
```

Confirm test data was received:
```bash
$ curl -s \
    http://127.0.0.1:3100/loki/api/v1/query_range \
    --data-urlencode 'query={job="testjob"}' \
| jq '.data.result'
```
```
[
  {
    "stream": {
      "detected_level": "unknown",
      "job": "testjob",
      "service_name": "testjob"
    },
    "values": [
      [
        "1764277360000000000",
        "it works"
      ]
    ]
  }
]
```


## Grafana

Get Grafana admin password:
```bash
$ kubectl -n observability \
    get secrets grafana \
    -o jsonpath='{.data.admin-password}' \
| base64 -d
```

Port forward:
```bash
$ kubectl -n observability \
    port-forward service/grafana \
    3000:80
```

```
http://127.0.0.1:3000
```
