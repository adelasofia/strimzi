#### INSTALLATION

Create a KafkaConnect image that includes the connector binaries using the following example Dockerfile:
 
```
FROM strimzi/kafka:0.16.1-kafka-2.4.0
USER root:root
 
ENV PLUGIN_NAME=debezium
ENV PLUGIN_BASE_URL=https://repo1.maven.org/maven2/io/debezium/debezium-connector-sqlserver
ENV PLUGIN_VERSION=1.1.0.Final
ENV PLUGIN_FILE=debezium-connector-sqlserver-1.1.0.Final-plugin.tar.gz
ENV PLUGIN_DIR=/opt/kafka/plugins/$PLUGIN_NAME
 
RUN curl -LO $PLUGIN_BASE_URL/$PLUGIN_VERSION/$PLUGIN_FILE.sha1 \
    && echo ' ' $PLUGIN_FILE >> $PLUGIN_FILE.sha1
 
RUN curl -LO $PLUGIN_BASE_URL/$PLUGIN_VERSION/$PLUGIN_FILE \
    && sha1sum --check $PLUGIN_FILE.sha1 \
    && mkdir -p $PLUGIN_DIR \
    && tar xvfz $PLUGIN_FILE -C $PLUGIN_DIR \
    && rm -f $PLUGIN_FILE*
 
USER 1001
```

Build an image from this Dockerfile and push to your repository.
 
```
docker build . -t <docker-org>/connect-debezium-sqlserver
docker push <docker-org>/connect-debezium-sqlserver
```

Create a KafkaConnect cluster based on the image you created using the following Custom Resource (kafka-connect.yaml):
```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  image: <docker-org>/connect-debezium-sqlserver
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

Note the `spec.config` replication factors of 3. This indicates that the topics created by Kafka Connect will be replicated across 3 brokers. 
If you have fewer brokers in your cluster you should reduce this number. 

Deploy Kafka Connect:
 
```
kubectl create -f kafka-connect.yaml
```

Create a Custom Resource for the Connector with the following contents (sqlserver-connector.yaml):

```
apiVersion: "kafka.strimzi.io/v1alpha1"
kind: "KafkaConnector"
metadata:
  name: "sqlserver-connector"
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: io.debezium.connector.sqlserver.SqlServerConnector
  tasksMax: 1
  config:
    database.hostname: "<db-hostname>"
    database.port: "<db-port>"
    database.user: "<db-username>" 
    database.password: "<db-password>" 
    database.server.name: "dbserver1"
    database.dbname: "<db-name>"
    database.history.kafka.bootstrap.servers: "<strimzi-cluster-name>-kafka-bootstrap:9092"
    database.history.kafka.topic: <topic-name>
```

The `database.user` and `database.password` values can be injected from a Secret to avoid having plaintext username and password in the resource. 
[Details](https://strimzi.io/docs/operators/master/using.html#proc-kafka-connect-mounting-volumes-deployment-configuration-kafka-connect) are provided in the documentation 

Deploy the Custom Resource to your Kubernetes cluster:
```
kubectl apply -f sqlserver-connector.yaml
```

Check that the resource was created:
```
kubectl get kctr --selector strimzi.io/cluster=my-connect-cluster -o yaml
```
