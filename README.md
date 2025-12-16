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

Use the transactional database for ID generation and constraint checks. Only a small portion of the data is stored in the transactional database. The rest is kept in memory while checks are pending. If checks pass, the data is written to the storage database. Mutation progress is tracked by sagas in the transactional database. As soon as data is written to the storage database, the saga can be finalized. If problems arise, the saga rolls back by deleting partially written data. All readers go straight to the storage database, they dont touch transactional one. Also it's worth to mention that reads may see newly written data immediately, even if the saga is not yet finalized.

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

## Typical Cases

### Balance System for External Consumers

Let's assume we need a balance system that allows tracking profile balances, accruing, and withdrawing them. There are multiple external systems that will use ours to display balances to their users and allow them to spend them. All such systems operate on the same users and the same balances. These could be our partners with whom we share internal balances, or different subdivisions of our company, each with their own mobile or web applications.

We consider 150 million profiles with 2 interactions per day (accrual, balance check, and withdrawal in each interaction). We assume that all interactions will occur within 12 hours (excluding nights), the busiest hour will have three times the average load, and seasonal/unplanned peaks will be three times heavier than usual days.

150m * 2 / 365 / 12 * 3 * 3 = ~620k — this is the number of interactions per hour we're targeting. That's roughly 200 interactions per second. So it would be great if our approach could handle such a hypothetical load.

#### Request Patterns

We will measure throughput and latency for both composed patterns (accrual, balance, withdrawal) and atomic operations: balance request, accrual, successful withdrawal, and unsuccessful withdrawal (insufficient balance). We will also measure mutation performance in the transactional database in isolation, in the storage database in isolation, and in the composition of both databases (balance queries are storage database isolated queries by nature).

#### Already Stored Data Volumes

As a target, we consider that such a system will consume 5 years of historical data from the old system and will operate for at least 5 more years. So we're interested in performance measurements for up to 10 years of data. Performance may degrade significantly as data volume grows. We will measure parameters for 0, 1, 5, and 10 years of history.

10 years of history gives us 6 billion operations.

## Table Width

Different cases require different table widths. Our target is 100 columns per table. We would also like to consider smaller tables with 10 and 50 columns per table.

## Number of Writers

For simplicity, we assume that all load is handled by one process. More complex setups may be measured later.

## Hardware for Performance Measurement

We will gradually move from simple setups to more production-grade configurations. We'll start with simple setups on a developer's laptop: no clusters, just one plain PostgreSQL instance + one Trino worker. Then we'll proceed with a more complex setup, still on a developer's laptop: a three-node PostgreSQL cluster (master, sync replica, async replica) and a three-node Trino cluster (coordinator, two worker nodes). After that, we'll measure performance on a more production-like setup: the same clusters but on rented servers.

## Improving Reliability with a Queue

We have two main stages for handling writes: writing to the transactional database and writing to the storage database. In most cases, problems will occur at the transactional database stage due to constraint violations. An unsuccessful write at the first stage is much more expected and doesn't leave any partial state — the operation is discarded entirely since the transaction was rejected. The situation is different at the storage database stage: problems are unexpected and in most cases not related to the data itself. Furthermore, when writing to the storage database, there is already partial state saved in the transactional database. Additionally, writes to the storage database suffer from small, frequent writes. Therefore, it's worth considering writing to the storage database through a queue. This allows aggregating all writes from many application instances and handling them by one or a small number of writers in batches. This mechanism can be omitted if you don't have many application instances or are satisfied with performance without an additional queue. In such cases, it may still be worthwhile to batch all writes to the storage database within a single application instance.

### Implementation Example with RxJS

It's totally fine to implement this dependency-free, but an RxJS implementation might look like the following:

### Implementation Example without Additional Dependencies

## Optimistic Writes

It's highly unlikely that a write will be unsuccessful if a durable queue is used for writing to the storage database. Therefore, a successful response can be sent back as soon as the write to the transactional database succeeds. Optimistic writes can also be considered without a queue, but in that case, it's much more likely that an operation reported as successful will eventually be rolled back.

If optimistic writes are implemented, they can be enabled on a per-table basis through settings. Another possible scenario is to opt in optimistic behaviour on write cuntion call level if calling client considers possible inconsistencies bearable.

An additional drawback of optimistic writes is that you often won't see an entry you just wrote if you query it immediately after the optimistic write succeeds.

## Balance Checks

Balance checks verify that balance changes stored in certain fields don't become negative when aggregated by given dimensions. There could be a more general approach — for example, more complex checks than just non-negative aggregations — but this case is the most common in our experience, so we've left generalization out of scope for now. Such checks are enforced by the transactional database. We plan to implement them using triggers since aggregations need to be performed.

### Example Check on Create

### More Complex Example for Create, Update, and Delete at the Same Time

## Composing with ELT-ish Writes

### Drawbacks of Mixing Different Styles of Loading for the Same Table

While it's possible to apply different styles of working with data on the same table, we consider it to be a bad practice. Usually, you would want to implement additional logic around functions that write data — some on-the-fly data transformations, side effects, etc. Since different styles of data writing are quite different in implementation, you would need to reimplement such logic twice. It's hard to keep it consistent and debug. So it would be wise to keep only one writing style per table. You could keep both if one of them is really occasional and bypasses any additional logic. Users or systems that would be using such an additional method of writing should be well aware of such unsafe behavior though.

### Steps to Write Data in ELT-ish Style

Often we need to do some transformations on data before we write it to the main table: field renaming, mapping, linking to existing dictionaries, creating missing dictionary entries, etc. We consider such actions to be performed beforehand — in memory, in stage tables, etc. When all preparations are done, we expect an Iceberg stage table with the same columns and rows filled as expected to add or update in our target database. The ID column is expected to be filled with nulls.

At that point, the following set of steps is taken:

* Generate stage tables in the storage database as they are intended to be written to the transactional database
* Check uniqueness inside the temporary table by unique field sets
* Transfer stage tables to the transactional database
* Check uniqueness on the same field sets but against rows in the target table in the transactional database
* Write stage tables to target tables in the transactional database. Identifiers will be generated on such writes
* Transfer generated identifiers to the full stage table in the storage database. IDs not present mean the row wasn't written due to an invariant violation or some other problem
* Update the target table in the storage database with rows that have assigned identifiers

Rows that don't pass checks may be marked in an additional column on the temp table or in an additional temp table.

For transferring identifiers, you need a way to uniquely identify a row beyond its ID, which you obviously can't use while it's not yet assigned to the stage table in the storage database.

This is a draft of the algorithm. Corner cases should be considered thoroughly. There are obvious problems with creating/updating rows and interaction with several tables of invariants at the same time.

### Reporting Problems with Row Handling

There are two approaches to handling data of rows with problems that prevent them from further processing. You may delete them and send information about them to logs, a different storage, or a table. Alternatively, you could keep them and just ignore them during further handling, optionally writing additional information about status or errors to temporary columns or tables.

### Simplifications in Comparison with OLTP-ish Style of Writing

While in OLTP-style writing we allow writes to different tables within the same saga, the ELT-style approach is simpler: we allow writing only to one table at a time. It is the developer of the ELT pipeline who is responsible for keeping cross-table data consistent. You need to define the order of writing to dependent tables and how to correct intended writes if not all rows of the previous table were written successfully.

As long as reads go only to the storage database, we don't bother to somehow mark temporary rows in the transactional database. They are considered to be possibly rolled back anyway.

## Bottlenecks

The main bottlenecks are the transactional database’s throughput and the memory needed to hold data while processing completes. Writes to the storage database can be heavily batched when targeting throughput over latency. If latency matters too, many small writes to the storage database will significantly affect it.

## Not covered

* How to define abandoned sagas?
* How to handle partial updates?
* What is the key metrics?
* What is the key use cases we want to measure performance for to asses applicability of the concept?
* Possibility of migration one uniquiness schema to another;
* What is a typical entity schamas we want to use in performanse measures?
* How to mutate schema?
