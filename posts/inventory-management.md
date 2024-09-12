---
title: "Design an Inventory Management System"
subtitle: "Design an iventory management system like hotel booking, item purchase"
date: "2024-09-08"
---

An inventory management system is a system that manages inventory update and get. A very typical inventory management system is Amazon.com, or a hotel booking system like Airbnb. If you are asked this question during an interview, the key part that the interviewer wants to challenge you is how to manage parallel writes and parallel reads.

## Problem Statement
### Functional Requirements
Design an inventory system like Amazon.com. Supported functionalities:
1. Get/list/update of inventory
2. We can abstract the inventory as inventory id and quantity
3. We don't need to worry about purchase, search and restock - those are managed by separate services.

### Non Functional Requirements
The servie need to scale, the quantity should be relatively accurate.

## Design

![images/inventory-management-1](/images/inventory-management-1.jpg)

The high level design is nothing fancy - it is probably the most typical architecture in a system. We have an Inventory Service, which consists of a load balancer (not shown in the diagram) and a bunch of hosts. We also have an Inventory Database, which is a distributed database. But the interviewer will not stop here, if the architecture is simple, the questions will be deeper.

### API and Database Schema
We will discuss how the API works here with the Database Schema. We will design three APIs: get, update and list.

```
get(inventory_id): -> count
update(inventory_id, count): -> null
list(): -> list<inventory_id, count>
```

And the database schema will look like:

| inventory_id  |  remaining_count                   |
| ------- | ------------------------------ |
|  id1 | 10             |
| id2 | 20 |

The primary key of the table is invetory_id. For the case, we could return the entry from the table; for the list case, we could do a full table scan; for the update case, we could update the entry in the table.

This might work for a single machine, but what if the service scale is big, and two clients are trying to call update APIs at the same time? It creates a race condition. For example, if you call 

```
update(id1, 10)
update(id1, 20)
```
Will the entry be 10 or 20 finally?

To resolve this issue, we need to use something called optimistic locking. There is also a pessimistic locking mechanism, but generally not as good as optimistic locking. For the difference see [here](https://stackoverflow.com/questions/129329/optimistic-vs-pessimistic-locking). There are many tutorials about the difference between those on the Internet. So here I will not discuss the theory, I will focus on the actual implementation of this case.

We update our APIs to:

```
get(inventory_id): -> <count, version_start_time>
update(inventory_id, version_start_time, count): -> null
list(): -> list<inventory_id, version_start_time, count>, continuation_token
```

And the DB schema will look like:

| inventory_id  |  version_start_time | version_end_time | count          |
| ------- | ------------------------------ |
|  id1 | 100 | null | 10 |
|  id2 | 100 | null | 10  |

Primary key is (inventory_id, version_start_time).

In the write operations, a version must be provided to perform the write. If the version is not the same as the one in the database, then the write will fail. During the write operation, we will terminate the current version, and create a new entry in the same transaction. For example, if user calls 

```
update(id1, 100, 20)
```

Then the DB will become:

| inventory_id  |  version_start_time | version_end_time | count          |
| ------- | ------------------------------ |
|  id1 | 100 | 200 | 10 |
|  id1 | 200| null | 20 |
|  id2 | 100 | null | 10  |

In the read operations, a version_start_time is returned for inventories. We only return the version_start_time of the latest version (version_start_time is null). For example, if user calls 

```
list()
```

We will return 

```
{
inventory_id: id1
version_start_time: 200
count: 20
}, {
inventory_id: id2
version_start_time: 100
count: 10
}
```
Also a continuation token is provided to perform pagination. 

### List By Category
In the real world case, we often could see a filter by category. For example, in a hotel booking website like Booking.com, we could filter based on room type, in an instrument retail website like Reverb, we could filter based on the instrument type. 


![images/inventory-management-2](/images/inventory-management-2.png)

To support this, we have another column Metadata, so the db will look like:


| inventory_id  |  version_start_time | version_end_time | count | metadata |
| ------- | ---------------|--------------- | -----| ------|
|  id1 | 100 | 200 | 10 | metadata1 |
|  id1 | 200| null | 20 | metadata1 |
|  id2 | 100 | null | 10  | metadata2 |

When we list, we could scan the database, and return the list of items. This could work, if the number of items is not that big. But each list operation will perform a full table scan. Can we do better?

We could add another global secondary index, or a table to map from the metadata to the inventory. In this case, we will create another table

| metadata | inventory_id |
| ----|--------------------|
| metadata1 | id1 |
| metadata2 | id2 |

Primary key will be (metadata, inventory_id). This table will only store a mapping from metadata to the inventory_id, not the detailed inventory info, to save storage. When we list by metadata, the database first performs a list on the metadata table on the prefix, then joins this with the inventory table to get the response.

### Database sharding
Now we have two tables, Inventory Table and Metadata Table. Let’s take a look at how the database scales. First, let’s talk about database sharding and replication.

Sharding, is that we keep different sets of data in each database, for example, we could split Inventory Table based on the inventory_id: Host 1 contains inventory_id 1 - 100, and Host 2 contains inventory_id 101-200, so the diagram will look like this:

![images/inventory-management-3](/images/inventory-management-3.jpg)

When we get the request from the service, the load balancer will check the inventory id, and direct the read and write traffic to the specific host. On top of this, we need replication, which is to duplicate each host multiple times, to form a replication group. In each group, there are multiple machines, and they all store the same data. Now it looks like:

![images/inventory-management-4](/images/inventory-management-4.jpg)

This has two benefits:
In each replication group, there is one leader (Host1) and multiple followers (Host2 and Host3). The leader will perform write and read operations, while the followers perform only the read operations. In this manner, the read traffic is spread out.
In the case a leader fails, one follower will be promoted to the leader, then the service is still available.

This is a very typical leader follower structure in a distributed database. There are many other topics for this. But we will not go into more details here. Instead, we will discuss how those two tables are stored in a database like that.

Suppose that for inventory_id, replication group 1 stores inventory_id1, and replication group 2 stores inventory_id2. For metadata, replication group 1 stores metadata1 and replication group stores metadata2. This is a perfect example of sharding. This is problematic, however, as inventory_id1 may have metadata2, so they are actually stored in different hosts! Each read or write request will update two tables at the same time, this is not happening in the single transaction. Can we resolve this?

The answer is, not sharding. Sharding has its pros and cons, and unfortunately we can’t achieve consistency and availability at the same time. Or in this specific example, we could partition only the Inventory Table, but not the Metadata Table, since the metadata table is small, we could store that in every replication group, no partitioning.

## Summary
In this article, we discussed a specific example of inventory management. Its idea is simple, but it mainly challenges the interviewee’s knowledge on distributed databases includes:
Transaction locking
DB schema design
Replication and Partitioning
