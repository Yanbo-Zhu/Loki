

# 1 通过 docker 

https://ghazanfaralidevops.medium.com/grafana-loki-promtail-complete-end-to-end-project-d698aaa636d6

![](https://miro.medium.com/v2/resize:fit:700/0*KYC2a7rEsloeKVa2)

![](https://miro.medium.com/v2/resize:fit:700/0*qCBtNDVf9QLVhCyN)

localhost:9080/targets
http://myserver:9080/targets   (https://community.grafana.com/t/2-how-to-check-promtail-agent-status-via-promtail-ui/100267)

# 2 通过 helm 

https://github.com/grafana/helm-charts/tree/main/charts/promtail
https://github.com/grafana/helm-charts/blob/main/charts/promtail/values.yaml

我仔细研究过， 通过helm 安装 promtail 在 kubnetes cluster中。 没办法 expose loki through ingress of cluster 

