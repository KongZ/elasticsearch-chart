# Elasticsearch Helm Chart
This Chart is designed to use on any Elasticsearch images. 

## Features
* Separate Elasticsearch nodes to specific roles master, coordinating and data nodes in single chart.
* Master and Data nodes will be deployed using Kubernetes StatefulSets
* Coordinating nodes will be deployed using Kubernetes Deployment
* Data nodes supports scaling up and down without degrading the cluster. The `pre-stop-hook`will automatically reallocate shards to available data node before shutting down. And `post-start-hook` will automatically bring node back to the cluster.
* Automatic discovery new node
* Support affinity and anti-affinity
* Enabled Prometheus Exporter (https://github.com/vvanholl/elasticsearch-prometheus-exporter)
* Automatic join other clusters
* Automatic calculate minimum master nodes using quoram 

## Installing the Chart

To install the chart with the release name `my-es`:

```bash
$ helm install --name ${my-es} .
```

## Deleting the Charts
```bash
$ helm delete ${my-es} --purge
```

Deletion of the StatefulSet doesn't cascade to deleting associated PVCs. To delete them:

```bash
$ kubectl delete pvc -l release=${my-release},component=data
```

## Configuration

The following tables lists the configurable parameters of the elasticsearch chart and their default values.

| Parameter                            | Description                                              | Default                                             |
|--------------------------------------|----------------------------------------------------------|-----------------------------------------------------|
| `image.repository`                   | Container image name                                     | `docker.elastic.co/elasticsearch/elasticsearch-oss` |
| `image.tag`                          | Container image tag                                      | `6.2.1`                                             |
| `image.pullPolicy`                   | Container pull policy                                    | `IfNotPresent`                                      |
| `cluster.name`                       | Cluster name                                             | `elasticsearch`                                     |
| `cluster.config`                     | Additional cluster config appended                       | `{}`                                                |
| `cluster.env`                        | Cluster environment variables                            | `{}`                                                |
| `cluster.zen.hosts`                  | Other cluster nodes                                      | `{}`                                                |
| `coordinating.replicas`              | Coordinating node replicas (deployment)                  | `3`                                                 |
| `coordinating.serviceType`           | Coordinating service type                                | `ClusterIP`                                         |
| `coordinating.heapSize`              | Coordinating node heap size                              | `512m`                                              |
| `coordinating.antiAffinity`          | Coordinating node anti-affinity                          | `soft`                                              |
| `coordinating.resources`             | Coordinating node resources requests & limits            | `{}`                                                |
| `coordinating.podAnnotations`        | Coordinating deployment annotations                      | `{}`                                                |
| `coordinating.serviceAnnotations`    | Coordinating service annotations                         | `{}`                                                |
| `master.replicas`                    | Master node replicas (deployment)                        | `3`                                                 |
| `master.exposeHttp`                  | Expose http port 9200 on master Pods for monitoring, etc | `false`                                             |
| `master.heapSize`                    | Master node heap size                                    | `256m`                                              |
| `master.persistence.enabled`         | Master persistent enabled/disabled                       | `true`                                              |
| `master.persistence.size`            | Master persistent volume size                            | `4Gi`                                               |
| `master.persistence.storageClass`    | Master persistent volume class                           | `nil`                                               |
| `master.persistence.accessMode`      | Master persistent access mode                            | `ReadWriteOnce`                                     |
| `master.antiAffinity`                | Master node anti-affinity                                | `soft`                                              |
| `master.resources`                   | Master node resources requests & limits                  | `{}`                                                |
| `master.podAnnotations`              | Master Deployment annotations                            | `{}`                                                |
| `data.replicas`                      | Data node replicas (statefulset)                         | `2`                                                 |
| `data.exposeHttp`                    | Expose http port 9200 on data Pods for monitoring, etc   | `false`                                             |
| `data.heapSize`                      | Data node heap size                                      | `1024m`                                             |
| `data.persistence.enabled`           | Data persistent enabled/disabled                         | `true`                                              |
| `data.persistence.size`              | Data persistent volume size                              | `100Gi`                                             |
| `data.persistence.storageClass`      | Data persistent volume Class                             | `nil`                                               |
| `data.persistence.accessMode`        | Data persistent Access Mode                              | `ReadWriteOnce`                                     |
| `data.resources`                     | Data node resources requests & limits                    | `{}`                                                |
| `data.podAnnotations`                | Data statefulset annotations                             | `{}`                                                |
| `data.terminationGracePeriodSeconds` | Data termination grace period (seconds)                  | `3600`                                              |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

For example:
```bash
$ helm install --name ${my-es} . --set coordinating.replicas=3,master.replicas=3
```
```bash
$ helm install --name ${my-es} . --set data.persistence.storageClass=ssd,master.persistence.storageClass=ssd,cluster.name=graylog
```

The YAML value of `cluster.config` is appended to elasticsearch.yml file for additional customization.
For example 
```
$ helm install --name ${my-es} . --set cluster.config="processors: 2"
```

## Minimum Master Nodes
The `minimum_master_nodes` setting is automatically calculate `minimum_master_nodes` using a quorum formula. A quorum is `(number of master-eligible nodes / 2) + 1` If you want to change `minimum_master_nodes`, you have to edit `templates/configmap.yaml` manually

More info: https://www.elastic.co/guide/en/elasticsearch/reference/6.2/modules-node.html#split-brain

## Coordinating Nodes
When querying Elasticsearch, you have to send request to *Coordinating* nodes. This chart will create *Coordinating* cluster endpoint automatically. You can change *Coordinating* service endpoint by setting this parameter `coordinating.serviceType`

More Coordinating Nodes Info:
  - https://www.elastic.co/guide/en/elasticsearch/reference/6.2/modules-node.html#coordinating-node

More Service Endpoint Info: 
  - https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services---service-types

## Creates StorageClass for SSD-PD
To maximize the Elasticsearch performance, it is recommend to create both *Data* nodes and *Master* node with SSD storage type. 

To create SSD storage type on GCE

```
$ kubectl create -f - <<EOF
kind: StorageClass
apiVersion: extensions/v1beta1
metadata:
  name: ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
EOF
```

Then deploy a cluster with specified `data.storageClass` value `ssd`
```
$ helm install incubator/elasticsearch --name ${my-release} --set data.storageClass=ssd,data.storage=100Gi --set master.storageClass=ssd,master.storage=20Gi
```

## Affinity
It is recommend to use NodeAffinity to deploy Data node on high I/O, memory, and CPU nodes pool. First you need to create a node pool with high I/O in Kubernetes clusters. Then append these lines to `data:` node

```
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cloud.google.com/gke-nodepool
            operator: In
             values:
             - {YourNodePoolName}
```

## Prometheus Exporter
[Prometheus](https://prometheus.io/) is open source metrics and monitoring tool. You can use Prometheus to monitor Elasticsearch status. By default, chart will install [Prometheus Exporter Plugin](https://github.com/vvanholl/elasticsearch-prometheus-exporter) on Elasticsearch. You can configure Prometheus to automatically discovery Elasticsearch nodes by insert this configure in `prometheus.yml` 

```
- job_name: k8s-elasticsearch
  scrape_interval: 1m
  metrics_path: "/_prometheus/metrics"
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_scrape]
      regex: ^true$
      action: keep
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_pod_label_app]
      action: replace
      target_label: job
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: pod_name
```

Sample of Prometheus Metrics on Grafana
![Prometheus Metrics](http://github.com/KongZ/elasticsearch-charts/raw/master/prometheus.png) 

## Join other clusters
If you already have existing cluster and you want this cluster to join them, you can specify the address of existing cluster from

```
$ helm install --name ${my-es} . --set cluster.zen.hosts="{10.148.0.22}"
```

The `cluster.zen.hosts` parameter accept an array of zen nodes. To pass an array value on helm see https://github.com/kubernetes/helm/blob/master/docs/using_helm.md#the-format-and-limitations-of---set

