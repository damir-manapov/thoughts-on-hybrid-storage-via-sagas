# Thoughts on Hybrid Storage via Sagas

## Introduction

We need a storage system that can handle massive amounts of data while still providing some guarantees typically offered by transactional databases.

We need this because we sometimes requested to deliver systems to clients whose data volumes are an order of magnitude larger than previous installations. Ideally, we should not have to reconsider the system’s architecture in such time. Instead, we should design using approaches that operate far from their limits. That means the system should work reliably without heavy optimization of underlying systems or storages. We also want customers to be able to configure storage for new, previously unknown data sources. If customers configure it themselves, the system should work reasonably well without fine tuning or heavy normalization. This is even more important given that some requests may be heavy (e.g., deduplication, entity matching) and might be configured by customer staff without deep technical knowledge, so indexes won’t help in most cases. Additionally, we don’t know the amount of data customers will store from future connectors, so the ability to scale horizontally and cheaply is a strong selling point.

The basic idea is to use two databases: a classic transactional database for ID generation and constraint checks, and a storage database that scales well but offers limited traditional database functionality. This may let us benefit from both worlds, though their disadvantages also apply. Whether such a system is good enough is a subject for research.

## Obvious limitation

This approach will generally be slower than using a single classic transactional database, except in some narrow cases. However, it scales much better and may still be good enough for large entities that grow over time or are big in some installations. On the other hand, if a table is unlikely ever to grow large (tens/hundreds of millions of rows), keep it as a simple table in the transactional database for better performance and simplicity.

## Duality of mutations

There are two main mutation styles: OLTP-like and ETL-like. This paper focuses on OLTP-like mutations, but ETL-like processing can be handled alongside, applying the same constraints and operations in much larger batches. That way you can achieve significantly higher throughput. Don’t process entries one by one if you receive them batched or can batch them yourself.

## Constraints on data to maintain

* Uniqueness of a column or a set of columns
* Positive balances across dimensions (e.g., per user profile or per accrual document)

## Basic idea

Use the transactional database for ID generation and constraint checks. Only a small portion of the data is stored in the transactional database. The rest is kept in memory while checks are pending. If checks pass, the data is written to the storage database. Mutation progress is tracked by sagas in the transactional database. As soon as data is written to the storage database, the saga can be finalized. If problems arise, the saga rolls back by deleting partially written data. All readers go straight to the storage database, they dont touch transactional one. Also it worth mention that reads may see newly written data immediately, even if the saga is not yet finalized.

## Sagas

A saga is analogous to a transaction. We create one saga per set of entries that must be committed or rolled back together. It’s possible to run multiple mutations simultaneously, but you must decide how to group them into sagas.

## Tables required in the transactional database

We have the following tables:

* A saga table, with saga ID, creation date and state
* One ID-generation table per entity, with columns for the entity ID and the saga ID
* One uniqueness-check table per entity. Columns: entity ID, saga ID, uniqueness name, and a column holding a hash of all fields participating in that particular uniqueness. If an entity has two distinct uniqueness constraints (two different sets of columns), this table will hold two rows: one with the hash for the first set and one for the second. To mimic traditional behavior that ignores uniqueness when any field is null, simply do not create a row for that entry’s constraint when any participating field is null.
* One balance-tracking table per every balance we need to maintain for a given entity. Columns: entity ID, saga ID, balance change, and one column per balance dimension to check (see examples below).

## Examples of balance-tracking tables

Case 1 — balances per profile, no additional dimensions.

No extra dimensions; just the entry ID and the balance change.

Case 2 — balances per profile, tracked via another entity, say “operations.”

We track rows per operation, with the row ID being the operation ID. We add one dimension column to record the profile ID and one column for the balance change.

Case 3 — balances by profiles and accrual documents.

Similar to Case 2, but with a second dimension: document ID (or a pair: document type and document ID). This lets us verify not only that a profile has enough balance, but also that each particular accrual document still has sufficient remaining balance. In such a case, we withdraw balance not from the profile in general, but from the specific document that awarded the balance. This way we can see which accruals were already spent and which still have some balance left.

## Operations

### Updates (full rewrite)

For updates, first create new check rows. For some time, both the old and new checks are present. If the new check rows are created successfully, update the data in the storage database, then delete the old check rows that are no longer relevant (some of them may still apply).

### Deletes

Delete sagas don’t have rollbacks per se. If the first step—deleting from the transactional database was completed, there is no way back; you must propagate the deletion to the storage database. However, deletions from the transactional database can still fail checks. For example, deleting an accrual that has already been spent would make the balance negative and must be prevented.

## Transactions

We still use transactions in the transactional database to create the saga and all related entries. Uniqueness and balance checks are enforced by the database itself, so if a constraint is violated, the transaction is rolled back automatically.

## Housekeeping

We need to periodically detect abandoned sagas and either roll them back or proceed in case of deletions.

## Known consistency issues

It’s possible to write to the storage database and then roll back the saga for some reason. In that case, someone may briefly observe records that will eventually be rolled back as if they never existed. We consider such cases rare and acceptable for most scenarios, but you must assess whether this level of inconsistency is acceptable in your use case.

## Bottlenecks

The main bottlenecks are the transactional database’s throughput and the memory needed to hold data while processing completes. Writes to the storage database can be heavily batched when targeting throughput over latency. If latency matters too, many small writes to the storage database will significantly affect it.

## Not covered

* How to define abandoned sagas?
* How to handle partial updates?
* What is the key metrics?
* What is the key use cases we want to measure performance for to asses applicability of the concept?
