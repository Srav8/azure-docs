---
title: Partitioning and horizontal scaling in Azure Cosmos DB | Microsoft Docs
description: Learn about how partitioning works in Azure Cosmos DB, how to configure partitioning and partition keys, and how to pick the right partition key for your application.
services: cosmos-db
author: SnehaGunda
manager: kfile
documentationcenter: ''

ms.assetid: cac9a8cd-b5a3-4827-8505-d40bb61b2416
ms.service: cosmos-db
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 04/14/2018
ms.author: rimman
ms.custom: H1Hack27Feb2017

---

# Partition and scale in Azure Cosmos DB

[Azure Cosmos DB](https://azure.microsoft.com/services/cosmos-db/) is a globally distributed, multi-model database service designed to help you achieve fast, predictable performance. It scales seamlessly along with your application as it grows. This article provides an overview of how partitioning works for all the data models in Azure Cosmos DB. It also describes how you can configure Azure Cosmos DB containers to effectively scale your applications.

Partitioning and partition keys are discussed in this video:

> [!VIDEO https://www.youtube.com/embed/SS6WrQ-HJ30]
> 

## Partitioning in Azure Cosmos DB
In Azure Cosmos DB, you can store and query schema-less data with a single digit millisecond latency at any scale. Azure Cosmos DB provides containers for storing data called *collections* (for documents), *graphs*, or *tables*. 

Containers are logical resources and can span one or more physical partitions or servers. The number of partitions is determined by Azure Cosmos DB based on the storage size and provisioned throughput of the container. 

A *physical* partition is a fixed amount of reserved SSD-backed storage. Each physical partition is replicated for high availability. One or more physical partitions make up a container. Physical partition management is fully managed by Azure Cosmos DB, and you don't have to write complex code or manage your partitions. Azure Cosmos DB containers are unlimited in terms of storage and throughput. 

A *logical* partition is a partition within a physical partition that stores all the data associated with a single partition key value. Multiple logical partitions can end up in the same physical partition. In the following diagram, a single container has three logical partitions. Each logical partition stores the data for one partition key, LAX, AMS, and MEL respectively. Each of the LAX, AMS, and MEL logical partitions cannot grow beyond the maximum logical partition limit of 10 GB. 

![Resource partitioning](./media/introduction/azure-cosmos-db-partitioning.png) 

When a container meets the [partitioning prerequisites](#prerequisites), partitioning is completely transparent to your application. Azure Cosmos DB supports fast reads and writes, queries, transactional logic, consistency levels, and fine-grained access control via methods/APIs to a single container resource. The service handles distributing data across physical and logical partitions and routing of query requests to the right partition. 

## How does partitioning work

How does partitioning work? Each item must have a *partition key* and a *row key*, which uniquely identify it. Your partition key acts as a logical partition for your data and provides Azure Cosmos DB with a natural boundary for distributing data across physical partitions. The data for a single logical partition must reside inside a single physical partition, but physical partition management is managed by Azure Cosmos DB. 

In brief, here's how partitioning works in Azure Cosmos DB:

* You provision a Azure Cosmos DB container with **T** RU/s (requests per second) throughput.
* Behind the scene, Azure Cosmos DB provisions partitions needed to serve **T** requests per second. If **T** is higher than the maximum throughput per partition **t**, then Azure Cosmos DB provisions **N = T/t** partitions.
* Azure Cosmos DB allocates the key space of partition key hashes evenly across the **N** partitions. So, each partition (physical partition) hosts **1/N** partition key values (logical partitions).
* When a physical partition **p** reaches its storage limit, Azure Cosmos DB seamlessly splits **p** into two new partitions, **p1** and **p2**. It distributes values corresponding to roughly half of the keys to each of the new partitions. This split operation is completely invisible to your application. If a physical partition reaches its storage limit and all of the data in the physical partition belongs to the same logical partition key, the split operation does not occur. This is because all the data for a single logical partition key must reside in the same physical partition. In this case a different partition key strategy should be employed.
* When you provision throughput higher than **t*N**, Azure Cosmos DB splits one or more of your partitions to support the higher throughput.

The semantics for partition keys are slightly different to match the semantics of each API, as shown in the following table:

| API | Partition key | Row key |
| --- | --- | --- |
| Azure Cosmos DB | Custom partition key path | Fixed `id` | 
| MongoDB | Custom shared key  | Fixed `_id` | 
| Gremlin | Custom partition key property | Fixed `id` | 
| Table | Fixed `PartitionKey` | Fixed `RowKey` | 

Azure Cosmos DB uses hash-based partitioning. When you write an item, Azure Cosmos DB hashes the partition key value and uses the hashed result to determine which partition to store the item in. Azure Cosmos DB stores all items with the same partition key in the same physical partition. The choice of the partition key is an important decision that you have to make at design time. You must pick a property name that has a wide range of values and has even access patterns. If a physical partition reaches it's storage limit and the data in the partition has the same partition key, Azure Cosmos DB returns the *"Partition key reached maximum size of 10 GB"* message, and the partition is not split. Choosing a good partition key is a very important decision.

> [!NOTE]
> It's a best practice to have a partition key with a large number of distinct values (e.g., hundreds or thousands). It lets you distribute your workload evenly across these values. An ideal partition key is one that appears frequently as a filter in your queries and has sufficient cardinality to ensure your solution is scalable.
>

Azure Cosmos DB containers can be created as *fixed* or *unlimited* in the Azure portal. Fixed-size containers have a maximum limit of 10 GB and 10,000 RU/s throughput. To create a container as unlimited, you must specify a partition key and a minimum throughput of 1,000 RU/s. 

It is a good idea to check how your data is distributed across partitions. To check this in the portal, go to your Azure Cosmos DB account and click on **Metrics** in **Monitoring** section and then click on **storage** tab to see how your data is partitioned across different physical partitions.

![Resource partitioning](./media/partition-data/partitionkey-example.png)

The left image above shows the result of a bad partition key and the right image above shows the result when a good partition key was chosen. In the left image, you can see that the data is not evenly distributed across the partitions. You should strive to choose a partition key that distributes your data so it looks similar to the right image.

<a name="prerequisites"></a>
## Prerequisites for partitioning

For physical partitions to auto-split into **p1** and **p2** as described in [How does partitioning work](#how-does-partitioning-work), the container must be created with a throughput of 1,000 RU/s or more, and a partition key must be provided. When creating a container (e.g., a collection, a graph or a table) in the Azure portal, select the **Unlimited** storage capacity option to take advantage of unlimited scaling. 

If you created a container in the Azure portal or programmatically and the initial throughput was 1,000 RU/s or more, and you provided a partition key, you can take advantage of unlimited scaling with no changes to your container. This includes **Fixed** containers, so long as the initial container was created with at least 1,000 RU/s of throughput and a partition key is specified.

If you created a **Fixed** container with no partition key or throughput less than 1,000 RU/s, the container will not auto-scale as described in this article. To migrate the data from the container like this to an unlimited container (one with at least 1,000 RU/s and a partition key), you need to use the [Data Migration tool](import-data.md) or the [Change Feed library](change-feed.md). 

## Partitioning and provisioned throughput
Azure Cosmos DB is designed for predictable performance. When you create a container, you reserve the throughput in terms of *[Request Units](request-units.md) (RU) per second*. Each request makes an RU charge that is proportional to the amount of system resources like CPU, memory, and IO consumed by the operation. A read of a 1 KB document with session consistency consumes 1 RU. A read is 1 RU regardless of the number of items stored or the number of concurrent requests running at the same time. Larger items require higher RUs depending on the size. If you know the size of your entities and the number of reads you need to support for your application, you can provision the exact amount of throughput required for your application's needs. 

> [!NOTE]
> To fully utilize provisioned throughput of a container, you must choose a partition key that allows you to evenly distribute requests across all distinct partition key values.
> 
> 

<a name="designing-for-partitioning"></a>
## Work with the Azure Cosmos DB APIs
You can use the Azure portal or Azure CLI to create containers and scale them at any time. This section shows how to create containers and specify the provisioned throughput and partition key using each API.


### Azure Cosmos DB API
The following sample shows how to create a container (a collection) using SQL API. 

```csharp
DocumentClient client = new DocumentClient(new Uri(endpoint), authKey);
await client.CreateDatabaseAsync(new Database { Id = "db" });

DocumentCollection myCollection = new DocumentCollection();
myCollection.Id = "coll";
myCollection.PartitionKey.Paths.Add("/deviceId");

await client.CreateDocumentCollectionAsync(
    UriFactory.CreateDatabaseUri("db"),
    myCollection,
    new RequestOptions { OfferThroughput = 20000 });
```

You can read an item (document) using the `GET` method in the REST API or using `ReadDocumentAsync` in one of the SDKs.

```csharp
// Read document. Needs the partition key and the ID to be specified
DeviceReading document = await client.ReadDocumentAsync<DeviceReading>(
  UriFactory.CreateDocumentUri("db", "coll", "XMS-001-FE24C"), 
  new RequestOptions { PartitionKey = new PartitionKey("XMS-0001") });
```

For more information, see [Partitioning in Azure Cosmos DB using the SQL API](sql-api-partition-data.md).

### MongoDB API
With the MongoDB API, you can create a sharded collection through your favorite tool, driver, or SDK. In this example, we use the Mongo Shell to create a collection.

In the Mongo Shell:

```
db.runCommand( { shardCollection: "admin.people", key: { region: "hashed" } } )
```
    
Results:

```JSON
{
    "_t" : "ShardCollectionResponse",
    "ok" : 1,
    "collectionsharded" : "admin.people"
}
```

### Table API

To create a table using the Table API, use the `CreateIfNotExists` method. 

```csharp
CloudTableClient tableClient = storageAccount.CreateCloudTableClient();

CloudTable table = tableClient.GetTableReference("people");
table.CreateIfNotExists(throughput: 800);
```

Provisioned throughput is set as an argument of `CreateIfNotExists`. The partition key is implicitly created as the `PartitionKey` value. 

You can retrieve a single entity by using the following code:

```csharp
// Create a retrieve operation that takes a customer entity.
TableOperation retrieveOperation = TableOperation.Retrieve<CustomerEntity>("Smith", "Ben");

// Execute the retrieve operation.
TableResult retrievedResult = table.Execute(retrieveOperation);
```
For more information, see [Develop with the Table API](tutorial-develop-table-dotnet.md).

### Gremlin API

With the Gremlin API, you can use the Azure portal or Azure CLI to create a container which represents a graph. Alternatively, because Azure Cosmos DB is multi-model, you can use one of the other APIs to create and scale your graph container.

You can read any vertex or edge by using the partition key and ID in Gremlin. For example, for a graph with region ("USA") as the partition key and "Seattle" as the row key, you can find a vertex by using the following syntax:

```
g.V(['USA', 'Seattle'])
```

You can reference an edge by using the partition key and the row key.

```
g.E(['USA', 'I5'])
```

For more information, see [Using a partitioned graph in Azure Cosmos DB](graph-partitioning.md).


<a name="designing-for-scale"></a>
## Design for scale
To scale effectively with Azure Cosmos DB, you need to pick a good partition key when you create your container. There are two main considerations for choosing a good partition key:

* **Query boundary and transactions**. Your choice of partition key should balance the need to use transactions against the requirement to distribute your entities across multiple partition keys to ensure a scalable solution. At one extreme, you can set the same partition key for all your items, but this option might limit the scalability of your solution. At the other extreme, you can assign a unique partition key for each item. This choice is highly scalable, but it prevents you from using cross-document transactions via stored procedures and triggers. An ideal partition key enables you to use efficient queries and has sufficient cardinality to ensure your solution is scalable. 
* **No storage and performance bottlenecks**. It's important to pick a property that allows writes to be distributed across various distinct values. Requests to the same partition key can't exceed the provisioned throughput allocated to a partition and will be rate-limited. So it's important to pick a partition key that doesn't result in "hot spots" within your application. Because all the data for a single partition key must be stored within a partition, you should avoid partition keys that have high volumes of data for the same value. 

Let's look at a few real-world scenarios and good partition keys for each:
* If you're implementing a user profile backend, the *user ID* is a good choice for a partition key.
* If you're storing IoT data, for example, device state, a *device ID* is a good choice for a partition key.
* If you're using Azure Cosmos DB for logging time-series data, the *hostname* or *process ID* is a good choice for a partition key.
* If you have a multitenant architecture, the *tenant ID* is a good choice for a partition key.

In some use cases, like IoT and user profiles, the partition key might be the same as your *ID* (document key). In others, like the time-series data, you might have a partition key that's different from the *ID*.

### Partitioning and logging/time-series data
One of the common use cases in Azure Cosmos DB is logging and telemetry. It's important to pick a good partition key in this scenario, because you might need to read/write vast volumes of data. The choice for a partition key depends on your read-and-write rates and the kinds of queries you expect to run. Here are some tips on how to choose a good partition key:

* If your use case involves a small rate of writes that accumulate over a long time and you need to query by ranges of timestamps with other filters, use a rollup of the timestamp. For example, a good approach is to use date as a partition key. With this approach, you can query over all the data for a given date from a single partition. 
* If your workload is write-heavy, which is very common in this scenario, use a partition key that is not based on the timestamp. As such, Azure Cosmos DB can distribute and scale writes evenly across various partitions. Here a *hostname*, *process ID*, *activity ID*, or another property with high cardinality is a good choice. 
* Another approach is a hybrid approach, where you have multiple containers, one for each day/month, and the partition key is a more granular property like *hostname*. This approach has the benefit that you can set different throughput for each container based on the time window and the scale and performance needs. For example, a container for the current month may be provisioned with a higher throughput, because it serves reads and writes. Previous months may be provisioned with a lower throughput, because they only serve reads.

### Partitioning and multitenancy
If you are implementing a multitenant application using Azure Cosmos DB, there are two popular designs to consider: *one partition key per tenant* and *one container per tenant*. Here are the pros and cons for each:

* **One partition key per tenant**. In this model, tenants are colocated within a single container. Queries and inserts for a single tenant can be performed against a single partition. You can also implement transactional logic across all items belonging to a tenant. Because multiple tenants share a container, you can better utilize storage and provisioned throughput by pooling resources for all tenants within a single container rather than provisioning for each tenant. The drawback is that you don't have performance isolation per tenant. Increasing throughput to guarantee performance will apply to the entire container with all the tenants versus targeted increases for an individual tenant.
* **One container per tenant**. In this model, each tenant has its own container, and you can reserve throughput with guaranteed performance per tenant. This model is more cost-effective for multitenant applications with a few tenants.

You can also use a hybrid approach that colocates small tenants together and isolates larger tenants to their own containers.

## Next steps
In this article, we provided an overview of concepts and best practices for scaling and partitioning in Azure Cosmos DB. 

* Learn about [provisioned throughput in Azure Cosmos DB](request-units.md).
* Learn about [global distribution in Azure Cosmos DB](distribute-data-globally.md).



