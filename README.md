# Blueprint

1. Setup a Confluent Cloud cluster in Azure Private Link
2. Design your network topology
3. Create AKS cluster
4. Setup Kafka Connect cluster
5. Deploy Connector Pod
6. Configure the connector

# Setup a Confluent Cloud cluster in Azure Private Link

Follow [CC on Azure PL](https://docs.confluent.io/cloud/current/networking/private-links/azure-privatelink.html) to get a Confluent Cloud (CC) cluster in Azure Private Link up & running. In this example, Private DNS resolution is chosen for ease of use. Make sure connectivity is tested with producing messages into the topic. Below shows the active cluster network on Private Link.

<img width="470" alt="image" src="https://github.com/StreamingCS/KafkaConnectOnAKS/assets/111465454/352b44d7-5129-46c8-a314-d476dd93dcab">

# Design your network topology

Plan how your data traverse your network topology securely. 
In below Azure Network topology,
- PrivateLinkDemo is a VNet building connections with Confluent Cloud Private Link
  - in Default subnet, PrivateEndpointDemo1-3 are the Private Endpoints to faciliate connections per cluster creation, with those nic are corresponding Network Interface
  - cscplconn & cscplconn811 provides proxy for Confluent Cloud console UI per [configure-a-proxy](https://docs.confluent.io/cloud/current/networking/ccloud-console-access.html#configure-a-proxy), an optional component
  - k8ssubnet is the subnet designed for AKS deployment to deploy Kafka Connect
- DemoClientVNet is a VNet to simulate data storages holds the data but exposed only at private endpoints, this VNet is peered to VNet PrivateLinkDemo

<img width="756" alt="image" src="https://github.com/StreamingCS/KafkaConnectOnAKS/assets/111465454/e3b3a699-0379-42ca-921a-01c0973f30ce">

- Park of the Network planning is the DNS resolution. In this demo we used 2 DNS private zone (1 for CC and 1 for MSSQL Private endpoint) and linked both to VNet so they know the IP address. Optionally you could replace it with a DNS service.

# Create AKS cluster

Create a properly sized AKS cluster to host connectors. In this demo Azure CNI is used because it's a direct approach to utilize what's already assigned IP spaces in the Azure VNet. Optionally you can check how this AKS resolving DNS records via [DNS check on AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/basic-troubleshooting-outbound-connections#check-whether-the-dns-resolution-is-successful-for-the-endpoint)

# Setup Kafka Connect cluster

Once the AKS cluster up & running, set the cluster context e.g.

```
az account set --subscription <<Your Sub ID>>
az aks get-credentials --resource-group <<Your RG>> --name <<k8s name>>
```

and use kubectl to create namespace "confluent" and set it to current context

```
kubectl create namespace confluent
kubectl config set-context --current --namespace=confluent
```

From now on, you may follow [here](https://www.confluent.io/en-gb/blog/run-apache-kafka-on-aws-eks-fargate-with-confluent-connectors/#deploy-operator-pod) because it's the same command on k8s to install and configure the Kafka Connect. Follows the page until [deploy-the-connector](https://www.confluent.io/en-gb/blog/run-apache-kafka-on-aws-eks-fargate-with-confluent-connectors/#deploy-the-connector) because it makes more sense to connect to a Azure Data Lake Storage (ADLS) endpoint than S3 in this case.

# Deploy Connectors

Deploy Connector is as easy as creating a yaml file. Once you have the confluent-operator pod running, and secrets create, we can proceed to deploy the connector pod.

```console
[ ~ ]$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
confluent-operator-8594c62f8c-sqkjw   1/1     Running   0          18s
```

Prepare a YAML file JDBC_sink.yaml like [Attached sample](/JDBC_sink.yaml) and apply using 

```console
Downloads % kubectl apply -f ./JDBC_sink.yaml
```

Around 5 minutes, the connector pod will be up & running

```console
Downloads % kubectl get pods                
NAME                                  READY   STATUS    RESTARTS   AGE
confluent-operator-8594c62f8c-9n9x7   1/1     Running   0          19m
jdbc-0                                1/1     Running   0          5m25s
Downloads % 
```

Forward the port to local so you can remotely configure it

```
kubectl port-forward jdbc-0 8083
```

# Configure the connector

In another terminal, we can start to configure the connector. Input below command in the console (Connection detail masked)

```
curl -X PUT \
-H "Content-Type: application/json" \
--data '{
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "connection.password": "",
      "tasks.max": "1",
      "topics": "NewTopic",
      "value.converter.basic.auth.credentials.source": "USER_INFO",
      "value.converter.schema.registry.url": "",
      "value.converter.basic.auth.user.info": "",
      "task.class": "io.confluent.connect.jdbc.sink.JdbcSinkTask",
      "connection.user": "",
      "name": "jdbc",
      "auto.create": "true",
      "value.converter": "io.confluent.connect.avro.AvroConverter",
      "connection.url": ""
}' \
http://localhost:8083/connectors/jdbc/config | jq .
```

and check its status via

```
curl http://localhost:8083/connectors/jdbc/status | jq
```

Below is the sample output

```console
Downloads % curl http://localhost:8083/connectors/jdbc/status | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   216  100   216    0     0   2586      0 --:--:-- --:--:-- --:--:--  2700
{
  "name": "jdbc",
  "connector": {
    "state": "RUNNING",
    "worker_id": "jdbc-0.jdbc.confluent.svc.cluster.local:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "jdbc-0.jdbc.confluent.svc.cluster.local:8083"
    }
  ],
  "type": "sink"
}
```

And double checked the SQL Server is now keep receiving stream data as expected.
