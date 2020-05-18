#### INSTALLATION

The required JAR file for the FileStreamSourceConnector is pre-installed in the Strimzi images.
 
Create a KafkaConnect cluster based on the image you created using the following Custom Resource (kafka-connect.yaml):

```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  image: strimzi/kafka:0.16.1-kafka-2.4.0
  replicas: 1
  bootstrapServers: <strimzi-cluster-name>-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: <strimzi-cluster-name>-cluster-ca-cert
        certificate: ca.crt
  config:
    config.storage.replication.factor: 3 
    offset.storage.replication.factor: 3
    status.storage.replication.factor: 3
```
Note the `spec.config` replication factors of 3. 
This indicates that the topics created by Kafka Connect will be replicated across 3 brokers. 
If you have fewer brokers in your cluster you should reduce this number. 

Deploy Kafka Connect:
 
```
kubectl create -f kafka-connect.yaml
```

Create a custom resource with the following contents (source-connector.yaml):

```
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: my-source-connector 
  labels:
    strimzi.io/cluster: my-connect-cluster 
spec:
  class: org.apache.kafka.connect.file.FileStreamSourceConnector 
  tasksMax: 2 
  config: 
    file: "/path/to/source-file"
    topic: my-topic
```

Deploy the custom resource to your Kubernetes cluster:

```
kubectl apply -f source-connector.yaml
```

Check that the resource was created:
```
kubectl get kctr --selector strimzi.io/cluster=my-connect-cluster
```
