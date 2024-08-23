**Purpose**  
Manifests to achieves consistent sharding of Prometheus metrics across the StatefulSet pods using Dynatrace Collector.   

Using Shard is recommended to achieve the outcomes:  
1. Each metric is consistently assigned to a specific pod based on its address.  
2. Only the assigned pod will keep and process that metric.  
3. Distributes the workload evenly across all pods in the StatefulSet.  
4. This configuration ensures that each OTel Collector instance handles a specific subset of the metrics, preventing duplication and allowing for horizontal scaling of metric collection and processing.

**Manifests**  
Stateful configuration as below deploys DynatraceCollector in a StatefulSet. A StatefulSet will ensure stable, unique network identifiers, persistent storage, and ordered deployment and scaling.  
```
kind: StatefulSet
metadata:
  name: dynatrace-collector
spec:
  replicas: 3
  template:
    spec:
      containers:
      - env:
        - name: SHARDS
          value: "3"
        - name: POD_NAME_PREFIX
          value: dynatrace-collector
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
```  
**Key points**:
1. The StatefulSet is named as "dynatrace-collector".    
2. It will create 3 replicas (pods).  
3. Each pod has environment variables:    
    SHARDS: Set to "3", matching the number of replicas.  
    POD_NAME_PREFIX: Set to "dynatrace-collector", matching the StatefulSet name.  
    POD_NAME: Dynamically set to the pod's name.  

**Prometheus Receiver Configuration**  
The second important piece is the configuration for the Prometheus receiver in the Dynatrace Collector:  
```
receivers:
  prometheus:
    config:
      scrape_configs:
        - relabel_configs:
            - action: hashmod
              modulus: ${env:SHARDS}
              source_labels:
                - __address__
              target_label: __tmp_shard_id
            - action: replace
              replacement: ${env:POD_NAME_PREFIX}-$$1
              source_labels:
                - __tmp_shard_id
              target_label: __tmp_shard_pod_name
            - action: keep
              regex: ${env:POD_NAME}
              source_labels:
                - __tmp_shard_pod_name
```
This configuration sets up relabeling rules for the Prometheus scraper:  
1. The first rule uses a hash of the __address__ label to assign a shard ID (0, 1, or 2).  
2. The second rule creates a potential pod name based on the shard ID.  
3. The third rule keeps only the metrics that match the current pod's name.  

In order to setup DynatraceCollector as StatefulSet, download the manifests and run the below commands:
```
kubectl apply -n otel-collector-secret.yaml -n <namespace-name>

kubectl apply -n otel-collector.yaml -n <namespace-name>
```
