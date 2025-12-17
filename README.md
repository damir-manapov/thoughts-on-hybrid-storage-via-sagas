# Thoughts on Hybrid Storage via Sagas

## Introduction

We need a storage system that can handle massive amounts of data while still providing some guarantees typically offered by transactional databases.

We need this because we sometimes requested to deliver systems to clients whose data volumes are an order of magnitude larger than previous installations. Ideally, we should not have to reconsider the system’s architecture in such time. Instead, we should design using approaches that operate far from their limits. That means the system should work reliably without heavy optimization of underlying systems or storages. We also want customers to be able to configure storage for new, previously unknown data sources. If customers configure it themselves, the system should work reasonably well without fine tuning or heavy normalization. This is even more important given that some requests may be heavy (e.g., deduplication, entity matching) and might be configured by customer staff without deep technical knowledge, so indexes won’t help in most cases. Additionally, we don’t know the amount of data customers will store from future connectors, so the ability to scale horizontally and cheaply is a strong selling point.

So, in short, we expect that the developed system can be spun up for another client with an order of magnitude more data than the previous one, can be hosted on usual mid-grade not specialized virtual machines, data is available to analytics and BIs, can be accessed by configurable requests not known beforehand, and can be configured and extended by customers' non-technical staff without prior strong technical knowledge.

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

Such in-process microbatching may be built into the Trino client or into a wrapper around it.

### Microbatch writer example

```typescript
/**
 * Microbatch writer for Trino/Iceberg.
 *
 * Problem solved:
 * Trino/Iceberg has high per-write overhead (~100ms+ per INSERT). Writing rows
 * individually results in terrible throughput (~10 ops/sec). By batching multiple
 * rows into a single INSERT statement, we can achieve ~8,500 ops/sec.
 *
 * How it works:
 * 1. Callers call write(row) which returns a Promise
 * 2. Rows accumulate in an internal buffer
 * 3. After `flushIntervalMs` (default 500ms), all buffered rows are flushed
 *    as a single SQL INSERT statement
 * 4. All pending Promises resolve/reject based on the batch result
 *
 * Trade-offs:
 * - Latency: Each write waits up to `flushIntervalMs` before execution
 * - Atomicity: All rows in a batch succeed or fail together
 * - Contention: Multiple concurrent batchers writing to the same table/shards
 *   cause conflicts (see measurements in docs - 50 workers = 50% failure rate)
 *
 * @template T - The row type (e.g., { id: number, name: string })
 */
export class TrinoBatchWriter<T> {
  /**
   * Buffer holding rows waiting to be flushed.
   * Cleared on each flush; new rows go to a fresh buffer.
   */
  private buffer: T[] = [];

  /**
   * Promise callbacks for each buffered row.
   * Parallel array to `buffer` - pending[i] corresponds to buffer[i].
   * On flush success, all resolve(); on failure, all reject(error).
   */
  private pending: Array<{
    resolve: () => void;
    reject: (_err: Error) => void;
  }> = [];

  /**
   * Timer handle for the scheduled flush.
   * null when no flush is scheduled (either just flushed or currently flushing).
   */
  private timer: ReturnType<typeof setTimeout> | null = null;

  /**
   * Guard flag to prevent overlapping flushes.
   * While true, new writes won't schedule a timer - they'll be picked up
   * after the current flush completes.
   */
  private flushing = false;

  /**
   * User-provided function that converts an array of rows into a SQL INSERT.
   * Example: (rows) => `INSERT INTO t VALUES ${rows.map(r => `(${r.id}, '${r.name}')`).join(',')}`
   * NOTE: Caller is responsible for proper SQL escaping!
   */
  private buildSql: (_rows: T[]) => string;

  /**
   * Milliseconds to wait before flushing accumulated rows.
   * Lower = less latency but smaller batches; Higher = larger batches but more latency.
   * Sweet spot depends on your write pattern and acceptable latency.
   */
  private flushIntervalMs: number;

  /**
   * @param buildSql - Function that builds INSERT SQL from array of rows
   * @param flushIntervalMs - Max time to buffer rows before flushing (default: 500ms)
   */
  constructor(buildSql: (_rows: T[]) => string, flushIntervalMs: number = 500) {
    this.buildSql = buildSql;
    this.flushIntervalMs = flushIntervalMs;
  }

  /**
   * Queue a row for writing.
   *
   * The returned Promise resolves when this row's batch is successfully written,
   * or rejects if the batch fails. Note: if the batch fails, ALL rows in that
   * batch fail together - there's no partial success.
   *
   * @param row - The row data to write
   * @returns Promise that resolves on successful flush, rejects on failure
   */
  write(row: T): Promise<void> {
    return new Promise((resolve, reject) => {
      // Add row and its callbacks to the pending batch
      this.buffer.push(row);
      this.pending.push({ resolve, reject });

      // Schedule a flush if one isn't already scheduled or in progress.
      // If we're currently flushing, the flush() finally block will
      // schedule the next flush for any rows that arrived during flush.
      if (!this.timer && !this.flushing) {
        this.timer = setTimeout(() => this.flush(), this.flushIntervalMs);
      }
    });
  }

  /**
   * Internal flush implementation.
   *
   * Takes a snapshot of current buffer/pending, clears them for new writes,
   * then executes the batch. This allows new writes to accumulate during
   * the async Trino call without blocking.
   */
  private async flush(): Promise<void> {
    // Nothing to flush - can happen if flushNow() called on empty buffer
    if (this.buffer.length === 0) {
      this.timer = null;
      return;
    }

    // Mark as flushing to prevent new timers during the async operation
    this.flushing = true;
    this.timer = null;

    // Snapshot and clear - new writes during flush go to fresh arrays
    const rows = this.buffer;
    const callbacks = this.pending;
    this.buffer = [];
    this.pending = [];

    try {
      // Build and execute the batched INSERT
      const sql = this.buildSql(rows);
      console.log(`[TrinoBatch] Flushing ${rows.length} rows`);
      await trinoQuery(sql);

      // Success - resolve all promises from this batch
      callbacks.forEach((cb) => cb.resolve());
    } catch (e) {
      // Failure - reject all promises from this batch
      // Common causes: write conflicts, schema issues, Trino unavailable
      console.error(`[TrinoBatch] Flush failed for ${rows.length} rows:`, e);
      callbacks.forEach((cb) => cb.reject(e as Error));
    } finally {
      this.flushing = false;

      // If new rows arrived during the flush, schedule next flush.
      // This ensures continuous throughput under sustained write load.
      if (this.buffer.length > 0 && !this.timer) {
        this.timer = setTimeout(() => this.flush(), this.flushIntervalMs);
      }
    }
  }

  /**
   * Force immediate flush of any pending writes.
   *
   * Use cases:
   * - Graceful shutdown: await writer.flushNow() before process exit
   * - Testing: ensure writes complete before assertions
   * - Low-latency requirement: flush after critical writes
   *
   * @returns Promise that resolves when flush completes
   */
  async flushNow(): Promise<void> {
    // Cancel scheduled flush - we're flushing now
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }
    await this.flush();
  }

  /**
   * Number of rows currently buffered awaiting flush.
   * Useful for monitoring/metrics.
   */
  get pendingCount(): number {
    return this.buffer.length;
  }
}
```

### One more approach for batching

As long as we already have a transactional database that handles concurrency well, we could write full data for the table to a special table with a jsonb column. Then we could read payloads from there using "FOR UPDATE SKIP LOCKED" so the same rows won't be handled by different workers. But we still need a way to acknowledge to the caller that the write was completed.

## Optimistic Writes

It's highly unlikely that a write will be unsuccessful if a durable queue is used for writing to the storage database. Therefore, a successful response can be sent back as soon as the write to the transactional database succeeds. Optimistic writes can also be considered without a queue, but in that case, it's much more likely that an operation reported as successful will eventually be rolled back.

If optimistic writes are implemented, they can be enabled on a per-table basis through settings. Another possible scenario is to opt in optimistic behaviour on write cuntion call level if calling client considers possible inconsistencies bearable.

An additional drawback of optimistic writes is that you often won't see an entry you just wrote if you query it immediately after the optimistic write succeeds.

## Balance Checks

Balance checks verify that balance changes stored in certain fields don't become negative when aggregated by given dimensions. There could be a more general approach — for example, more complex checks than just non-negative aggregations — but this case is the most common in our experience, so we've left generalization out of scope for now. Such checks are enforced by the transactional database. We plan to implement them using triggers since aggregations need to be performed.

## Trigger example

```sql
CREATE OR REPLACE FUNCTION check_balance_not_negative()
RETURNS TRIGGER AS $$
DECLARE
  new_balance INTEGER;
  current_balance INTEGER;
  check_profile_id INTEGER;
  change_amount INTEGER;
BEGIN
  -- Determine which profile to check
  IF TG_OP = 'DELETE' THEN
    check_profile_id := OLD.profile_id;
    change_amount := -OLD.amount;
  ELSIF TG_OP = 'UPDATE' THEN
    check_profile_id := NEW.profile_id;
    change_amount := NEW.amount - OLD.amount;
  ELSE
    check_profile_id := NEW.profile_id;
    change_amount := NEW.amount;
  END IF;

  -- Get current balance
  SELECT COALESCE(SUM(amount), 0) INTO current_balance
  FROM balance_changes
  WHERE profile_id = check_profile_id;

  -- Calculate new balance
  new_balance := current_balance + change_amount;

  IF new_balance < 0 THEN
    RAISE EXCEPTION USING
      ERRCODE = 'P0001',
      MESSAGE = 'BALANCE_NEGATIVE',
      DETAIL = jsonb_build_object(
        'profile_id', check_profile_id,
        'current_balance', current_balance,
        'change_amount', change_amount,
        'new_balance', new_balance,
        'deficit', ABS(new_balance)
      )::text;
  END IF;

  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  ELSE
    RETURN NEW;
  END IF;
END;
$$ LANGUAGE plpgsql;
```

## Example of assigning trigger to table

```sql
DROP TRIGGER IF EXISTS trigger_check_balance ON balance_changes;
CREATE TRIGGER trigger_check_balance
BEFORE INSERT OR UPDATE OR DELETE ON balance_changes
FOR EACH ROW
EXECUTE FUNCTION check_balance_not_negative();
```

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

## Potential extensions

These are thoughts on how to extend the storage layer further. They don't depend on a particular implementation of the underlying storage; it may simply be a usual transactional database. It expects such storage to be SQL-compatible, with common functionality one expects from such a database.

### Decoupling entities extensions owned by different subsistems

### Data workflows

### Workflows of selections and filtering for particular entity 

### Extended read functions as base for simple and aggregations

### Reports

## Preliminary result

These aren't reliable results; they're more about upper limits. More thorough measurements need to be taken.

Environments: developer's notebook, one node for Trino, trivial table with one column, one process handler, writes to Iceberg only. Another variant is the same but with locale trino cluster of corrdinator and two workers and locale postgres cluster of master, sync replica and async replica.

Writes without microbatching yield about 10 operations per second; with microbatching at 100 milliseconds — 8.5 thousand operations per second.

This shows that batching improves write performance drastically.

## Preliminary testing of microbatching on Trino

In-process microbatching is expected to degrade with an increased number of processes handling writes. This is clearly shown with a simple check:

Half of writes fail with 50 concurrent workers, while with 5 workers all writes succeed. Latency is around 1 second with 5 workers, the same at p95, while with 10 workers it's around 1.1 seconds on average and 2.5 seconds at p95. So latency increases with contention as well, while being stable without contention.

For a local Trino cluster (coordinator + 2 workers), results for 5 workers are similar regarding throughput, average and p95 latency.

It's also worth mentioning that contention is based on writing to the same shards, so if different parts of the system work with different tables, they may scale independently.

Measurements can be found in Appendix A.

## Preliminary testing of balance checks

Small data prefilling with 1 million entries spread across 100 profiles (10k ops per profile), no index on profile, checking balance on profile solely, creation row only in balance changes table:

## Further operations on saved data

The main focus here is on how to save data. But a big part of the application is how data will be handled further. Here is a list of typical request patterns we expect from a system. All of them are read-only operations, so they all go to the storage database, not hitting the transactional one.

### Deduplication

### Segments

### Tags assignments

### Filters on related entities

## Note on SQL escaping

The Trino client is based on HTTP requests. So query is vulnerable to SQL injection. Even though it has prepared statements, it doesn't help because calling such a statement assumes providing arguments through basic strings. This way, any filter that comes from user input will eventually be concatenated to the string sent as a query to Trino. Given that, we need to escape parameters. There are several libraries for this, but none of them are properly supported. We use `pg-format`. It is pretty old, archived, and hasn't been updated for 9 years. But it is based on the escaping algorithm implementation of PostgreSQL, has one source file of 250 lines, and no dependencies. As far as I know, there is no better alternative for escaping SQL identifiers and parameters.

## Not covered

* How to define abandoned sagas?
* How to handle partial updates?
* What is the key metrics?
* What is the key use cases we want to measure performance for to asses applicability of the concept?
* Possibility of migration one uniquiness schema to another;
* What is a typical entity schamas we want to use in performanse measures?
* How to mutate schema?

# Appendix A

Simple checks of write contention with different numbers of workers and microbatching for 100 milliseconds. Write is Iceberg only, table is trivial with one column, no data prefilling. Local Compose setup with one Trino node.

#### 50 workers

status is 201: 58% — ✓ 13640 / ✗ 9501

* failed_writes: 9501 - 747.826194/s
* successful_writes: 13640 - 1073.607966/s
* write_latency: avg=2357.609956, min=865, med=2632, max=8140, p(90)=3477, p(95)=3964

#### 40 workers

status is 201: 69% — ✓ 18520 / ✗ 8248

failed_writes: 8248 - 658.628748/s
successful_writes: 18520 - 1478.880263/s
write_latency: avg=2033.658137 min=647, med=1866, max=5901, p(90)=2933, p(95)=3091

#### 30 workers

status is 201: 72% — ✓ 20596 / ✗ 7833

* failed_writes: 7833 - 663.175549/s
* successful_writes: 20596 - 1743.746153/s
* write_latency: avg=1872.555559 min=587, med=1439, max=5729, p(90)=2996, p(95)=3476

#### 20 workers

status is 201: 84% — ✓ 29923 / ✗ 5660

* failed_writes: 5660 - 486.457881/s
*successful_writes: 29923 - 2571.78077/s
*write_latency: avg=1475.367085 min=552, med=1102, max=5315, p(90)=2693, p(95)=2838.9

#### 10 workers

status is 201: 96%

* failed_writes: 1425 - 130.351699/s
* successful_writes: 45099 - 4125.425464/s
* write_latency: avg=1108.33933 min=709, med=938, max=4345, p(90)=1328, p(95)=2584

#### 5 workers

status is 201: 100%

* successful_writes: 51696 - 4773.13579/s
* write_latency: avg=1002.483654 min=720, med=955, max=2697, p(90)=1056, p(95)=1089
