# Install Loki-distributed,alertmanager and grafana helm charts and enable alering on logs.

## Archeticture :
![Blank board](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/69a034f2-2583-4833-bedb-3a330033f432)

* **Loki**: Loki server serves as storage, storing the logs in a time series database, but it won’t index them. To visualize the logs, you need
 to extend Loki with Grafana in combination with LogQL.

[Grafana-loki Documentation](https://grafana.com/docs/grafana/latest/datasources/loki/)

* **Grafana**: An open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach. It provides charts, graphs, and alerts for the web when connected to supported data sources.

[Documentation](https://grafana.com/docs/grafana/latest/)

* **Promtail**: deployed as a DaemonSet, and they're in charge of collecting logs from various pods/containers of our nodes. Loki supports various types of agents, but the default one is called Promtail.
* Promtail does the following actions:

1. It discovers the targets having logs
2. It attaches labels to log streams
3. And it pushes the log stream to Loki

[Documentation](https://grafana.com/docs/loki/latest/send-data/promtail/)

* **Alertmanager**: The Alertmanager handles alerts sent by client applications such as the Prometheus server. It takes care of deduplicating, grouping, and routing them to the correct receiver integrations such as email, PagerDuty, OpsGenie, or many other mechanisms thanks to the webhook receiver. It also takes care of silencing and inhibition of alerts.

[Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)

* **AzureBlobStorage**: Unlike other logging systems, Grafana Loki is built around the idea of only indexing metadata about your logs: labels (just like Prometheus labels). Log data itself is then compressed and stored in chunks in object stores such as Azure storage account(blob storage).
**Blob Storage** is Microsoft Azure’s hosted object store. It is a good candidate for a managed object store, especially when you’re already running on Azure, and is production safe. You can authenticate Blob Storage access by using a storage account name and key or by using a Service Principal.

[Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)

* **Slack**: To receive alerts from AlertManager.


### What is Loki?:
Loki is a horizontally-scalable, highly-available, multi-tenant log aggregation system inspired by Prometheus, a popular monitoring and alerting toolkit. Like Prometheus, Loki is designed to be simple to operate and cost-effective. Unlike many other log aggregation systems(like Elasticsearch), Loki does not index the contents of logs but instead indexes a set of labels for each log stream which makes it very cost effective

### Key Features of Loki
Compared to other log aggregation systems, Loki offers several advantages:

1. **No full-text indexing on logs**: Loki stores compressed, unstructured logs and only indexes metadata. This makes Loki simpler to operate and cheaper to run compared to systems that index the entire contents of logs.
2. **Label-based indexing and grouping**: Loki indexes and groups log streams using the same labels as Prometheus. This enables seamless switching between metrics and logs using the same labels you’re already using with Prometheus.
3. **Ideal for Kubernetes Pod logs**: Loki is particularly well-suited for storing Kubernetes Pod logs, as it automatically scrapes and indexes metadata such as Pod labels.
4. **Native support in Grafana**: Loki has native support in Grafana, starting from version 6.0.


### Loki-distributed Archeticture:

![Blank diagram](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/429ec636-8f33-40df-9efe-5a614d5e8034)

### components: 

1. **Gateway**: 

2. **Distributor**:
The distributor service is responsible for handling incoming streams by clients. It’s the first stop in the write path for log data. Once the distributor receives a set of streams, each stream is validated for correctness and to ensure that it is within the configured tenant (or global) limits

3. **Ingester**:
The ingester service is responsible for writing log data to long-term storage backends (DynamoDB, S3, zurestorage, etc.) on the write path and returning log data for in-memory queries on the read path.

4. **queryFrontend**:
used to accelerate the read path. When the query frontend is in place, incoming query requests should be directed to the query frontend instead of the queriers. The querier service will be still required within the cluster, in order to execute the actual queries.
The query frontend internally performs some query adjustments and holds queries in an internal queue. In this setup, queriers act as workers which pull jobs from the queue, execute them, and return them to the query-frontend for aggregation

5. **querier**:
The querier service handles queries using the LogQL query language, fetching logs both from the ingesters and from long-term storage(azure blobstore in our case or s3 if using amazon s3 buckets).

6. **ruler**: alerts management, it evaluates the rules of loki alerts and trigger alert if any rule meets the threshold set.

7. **compactor**: reducing the size of indexes and managing the storage time of logs in the data store (retention).

8. **index-gateway**: facilitates management of index-related operations and optimize log retrieval querying to enhance overall performance.

### Why Loki-distributed?:
* We used Loki-distributed instead of loki-stack which is deprecated and lacks lots of features that loki-distributed does:

1. **Scalability**: By deploying Loki in a distributed manner, you can scale your log aggregation system to handle large volumes of logs from multiple sources. Distributed Loki setups allow for horizontal scaling, where you can add more resources and nodes to accommodate increased log ingestion and querying demands. This scalability ensures that your log management solution can grow with your application’s needs.

2. **Fault Tolerance**: In a distributed Loki setup, log data is replicated and distributed across multiple storage nodes. This redundancy ensures fault tolerance and high availability. If a node or component fails, the system can continue to operate without interruption because the data is distributed and replicated across multiple nodes. This fault tolerance prevents data loss and ensures that log data remains accessible even in the event of a failure.

3. **Load Balancing**: With a distributed Loki setup, you can distribute the load of log ingestion and querying across multiple nodes. This load balancing mechanism improves overall system performance by effectively utilizing available resources and preventing bottlenecks. Load balancing ensures that the system can handle high traffic and large query volumes without compromising performance or responsiveness.

4. **Efficient Resource Utilization**: In a distributed setup, Loki components can be deployed on separate nodes or clusters, allowing for optimized resource utilization. Each component can be scaled independently based on its specific resource requirements. This flexibility ensures efficient resource allocation and prevents resource contention, maximizing the overall efficiency of your log management infrastructure.

5. **Improved Query Performance**: Distributing the querying workload across multiple Queriers in a distributed Loki setup enables parallel processing of queries. This parallelization enhances query performance and reduces query response times, even when dealing with large volumes of log data. The distributed setup ensures that queries are efficiently distributed and processed, resulting in faster and more responsive log analysis.

6. **Enhanced Availability and Redundancy**: A distributed Loki setup provides built-in redundancy by replicating log data across multiple storage nodes. This redundancy ensures that log data is highly available and accessible, even in the face of node failures or network issues. By spreading the data across multiple nodes, the distributed setup provides increased resilience and minimizes the risk of data loss.

Overall, using Loki in a distributed manner offers significant benefits in terms of scalability, fault tolerance, load balancing, query performance, and resource utilization. It allows your log management system to handle increasing log volumes, ensures high availability, and provides a more efficient and robust solution for log aggregation and analysis.


## Scenario:

* Create **AzureBlobStorage** to store grafana-loki chunks.
* Create a kubernetes secret for azure blobstorage **storageAccount** name and key.
* Deploying **Promtail** helm chart to gather logs from kubernetes pods stored on the respective nodes.
* Deploy **Loki-distributed** helm chart to receive and store the logs sent from Promtail and evaluate the rules for any alerts to be sent to alert manager.
* Deploy **kube-prometheus-stack** helm chart and only enable **Grafana** and **Alertmanager** , Grafana for viewing the logs and Alert manager for receiving the trigerred alerts and send it to slack( can send it to email or any other system.)

### Deployment: 

1. Create **AzureBlobStorage**:

![Screenshot from 2023-11-21 23-19-01](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/b6c8a3f4-85c2-4e47-8851-a5e2557b6434)

![Screenshot from 2023-11-21 23-20-48](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/cd872098-a6a3-486c-b122-41406d78ae6e)

![Screenshot from 2023-11-21 23-21-07](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/290b4a1e-ea1b-47c6-ac3a-96ba49a70caa)

![Screenshot from 2023-11-21 23-22-46](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/2c634c11-6543-4c1b-89a8-9e57636b7c29)

![Screenshot from 2023-11-21 23-23-06](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/32594eab-6cf8-4490-8c19-07f57a495252)

![Screenshot from 2023-11-21 23-25-16](https://github.com/MohamedEssam4444/Enable-Logging-and-alerting-on-applications-hosted-in-AKS-using-Loki-Grfana-and-alertmanager/assets/68178003/9cb24402-75ae-4d4e-b973-c98379532546)

2. Create kubernetes secret for **storageAccountName & key**: 

```
kubectl create secret generic loki-objstore-config \                                                   
    --from-literal=accountName=<name> \
    --from-literal=accountKey='<secret>' -n <namespace> 
```
3. deploy **Promtail**:
```
helm repo add helm repo add grafana https://grafana.github.io/helm-charts
helm rpeo update
helm upgrade --install promtail grafana/promtail -f promtail-overrides.yaml -n <namespace>
```
4. deploy **Loki**:
```
helm upgrade --install loki grafana/loki-distributed -f values-loki-dist.yaml -n <namespace>
```
5. deploy **Grafana & AlertManager**:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm rpeo update
helm upgrade --install grafana-alertmanager prometheus-community/kube-prometheus-stack -f prometheus-grafana-overrides.yaml -n <namespace>
```

### Files:

1. **values-loki-dist.yaml**: includes loki-distributed overriden values to override existing default values in this helm chart [https://github.com/grafana/helm-charts/tree/main/charts/loki-distributed].
2. **promtail-overrides.yaml**: includes promtail values to override the default promtail chart values [https://github.com/grafana/helm-charts/blob/main/charts/promtail/values.yaml]
3. **prometheus-grafana-overrides.yaml**: includes grafana,alertmanager values to override the default kube-prometheus chart values(disabling prometheus and other unneeded components) [https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack].


### Enhancements can be made: 

1. Make bucket private not public for more security.
2. Use service account instead of storage account name and key (this will require changes in the permissions for the storage account and values of loki.)

### References:

1. [https://medium.com/@CloudifyOps/a-comprehensive-guide-to-setting-up-loki-in-a-distributed-manner-on-amazon-eks-part-1-f9a732857d41]
2. [https://grafana.com/docs/loki/latest/get-started/components/]
3. [https://grigorkh.medium.com/what-is-grafana-loki-19a7db694083]
4. [https://www.reddit.com/r/grafana/comments/12ajrnq/loki_failed_to_save_chunks_in_configured_storage/]
