# CDC for Postgres using Kafka

We will be using kafka to capture row level data changes for the postgres. In the `charts` folder, there are helm charts for `postgres`, postgres dashboard `pgadmin` and `kafka operator`.  

## Prerequisites
- Helm3+
- Kubernetes Cluster
- Postgres with `wal_level=logical` and atleast one free `replication_slot`


### Install the Postgres(optional)  

In case there is no postgres deployed and if you need to deploy a postgres, there is a helm chart for the postgres in the charts folder. It has a values file with minimum values to deploy a one node postgres db.

```
helm install postgres charts/postgres/postgresql --namespace postgres --create-namespace -f charts/postgres/postgresql/values-deploy.yaml
```

### Install the kafka operator
```
helm install kafka charts/strimzi-kafka-operator --namespace kafka --create-namespace
```
> This will deploy the kafka operator. The kafka operator will create CRDs of Kafka type, Kafka Connector, Kafka connect. The Kafka resource will be used to create a kafka cluster. While the kafka connect and connector will be used to connect kafka to postgres.

### Deploy the Kafka Cluster
Firstly change the storage class needed by the Kafka cluster `spec.kafka.storage[].class` and `spec.zookeeper.storage.class`.  
Then deploy the kafka cluster using commnad,
```
kubectl apply -f kafka-cluster.yaml -n kafka
```
This will deploy a one node kafka cluster, with one zookeeper node.

### Insert data into Postgres

We will need to create a database and tables to capture data change using Kafka. For that we will need to download, the postgresql cli tool `psql`. Then follow the below commands;
```
# Get the ip for the postgres service
kubectl get svc -n postgres

# Login to the postgres using following command and enter the password
psql -h <IP> -P <PORT> -U postgres -W

# Then create a database and table in postgres.
```

### Build an image for Kafka Connect
Next we will create a Kafka connect image loaded with debezium connector to use to deploy kafka connect.
```
docker build -t <image-name> .
```
Next push this image to an image registry. This image will be used to deploy the kafka connect.

### Deploy Kafka Connect
* Set `spec.image` to the image pushed in `kafka-connect.yaml`
* Set `annotations.strimzi.io/use-connector-resources: "true"`
* Set `spec.bootstrapServers` to the addrress of the Kafka cluster.
```
kubectl apply -f kafka-connect.yaml -n kafka
```

### Deploy the Kafka Connector
Kafka Connector will be used to configure Kafka connect. The important fields are:
* `labels.strimzi.io/cluster: <name of the kafka connect>`
* Change `database.dbname` to the db you created
* Change `database.hostname` to the hostname of database
* Change `schema.whitelist` and `table.whitelist` to the table and schema name to be monitored. The table name should be of form `schema.table`
* Change the other required configs in the manifest.  
To deploy the Kafka Connector, use the command:
```
kubectl apply -f kafka-connector.yaml -n kafka
```

### Checking the deployment
Now to check if it is CDC is working as expected, we will run a consumer pod to see the data capture.
```
kubectl run kafka-consumer -n default --restart='Never' --image docker.io/bitnami/kafka:3.1.0-debian-10-r20 --command -- sleep infinity

# Now exec ito the pod kafka-consumer
kubectl exec -it kafka-consumer -n default -- bash

# Run the following command and substitute the values from the connector config file
kafka-console-consumer.sh \
  --bootstrap-server kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092 \
  --topic <database.server.name>.<schema.whitelist>.<table.whitelist> \
  --from-beginning

```
We will see the consumer is consuming the data changes in the postgres. At first it will take a snapshot of the current state. Next if we make any changes, it will be instantly reflected in the kafka stream.