# install efk-logging (via HELM3)

Reference: https://docs.bitnami.com/tutorials/integrate-logging-kubernetes-kibana-elasticsearch-fluentd/

### create namespace logging
 
> kubectl create namespace logging

### prepare helm repo

> helm repo add bitnami https://charts.bitnami.com/bitnami

### prepare volume 
for es master, es data, kibana

> kubectl apply -f pv0.yaml -n logging

> kubectl apply -f pv1.yaml -n logging

> kubectl apply -f pv2.yaml -n logging

### install es with replicas=1 (for development purpose)

> helm install elasticsearch -n logging --set master.persistence.storageClass=logging,data.persistence.storageClass=logging,master.replicas=1,coordinating.replicas=1,data.replicas=1,ingest.replicas=1,master.persistence.size=10Gi,data.persistence.size=10Gi bitnami/elasticsearch

### install kibana with replicas=1 (for development purpose)

> helm install kibana bitnami/kibana -n logging --set global.storageClass=logging,persistence.size=10Gi,service.port=8888,elasticsearch.enabled=false,elasticsearch.hosts[0]=elasticsearch-elasticsearch-coordinating-only.logging.svc.cluster.local,elasticsearch.port=9200,service.type=LoadBalancer


## install fluent-bit (option 1)

Reference: https://docs.fluentbit.io/manual/installation/kubernetes

> kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml

> kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml

> kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml

### install fluent-bit config (configmap setting for forward all console log to fluent-bit)

> kubectl apply -f fluent-bit-configmap.yaml -n logging

### install fluent-bit on DaemonSet

> kubectl apply -f fluent-bit-ds.yaml -n logging

## install fluentd (option 2)

Reference: https://docs.bitnami.com/tutorials/integrate-logging-kubernetes-kibana-elasticsearch-fluentd/

> kubectl apply -f elasticsearch-output.yaml -n logging

> kubectl apply -f apache-log-parser.yaml -n logging

> helm install fluentd bitnami/fluentd -n logging --set aggregator.replicaCount=1 --set aggregator.configMap=elasticsearch-output --set forwarder.configMap=apache-log-parser --set aggregator.extraEnv[0].name=ELASTICSEARCH_HOST --set aggregator.extraEnv[0].value=elasticsearch-elasticsearch-coordinating-only.logging.svc.cluster.local --set aggregator.extraEnv[1].name=ELASTICSEARCH_PORT --set-string aggregator.extraEnv[1].value=9200 --set forwarder.extraEnv[0].name=FLUENTD_DAEMON_USER --set forwarder.extraEnv[0].value=root --set forwarder.extraEnv[1].name=FLUENTD_DAEMON_GROUP --set forwarder.extraEnv[1].value=root
  
### note
- browser to open http://ip:8888 to access kibana
- tested on kube version 1.17
- suggest to use fluent-bit (option 1), less resources needed
- if you're running kube 1.15 or lower, suggest to use https://github.com/komljen/helm-charts/tree/master/efk directly, less effort