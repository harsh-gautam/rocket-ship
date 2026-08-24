# 10 Spark Anti-Patterns That Kill Performance (And How to Fix Them)

## Contributiors:

## Introduction
If you've spent any real time running Spark jobs in production, you've hit at least one of these walls: a job that silently balloons from 10 minutes to 4 hours, a driver that OOMs for no obvious reason, or a single task that runs 100x longer than every other task in its stage. These aren't bad luck - they're almost always the result of a handful of recurring anti-patterns.

Spark's Catalyst optimizer and Adaptive Query Execution (AQE) work hard to prevent exactly this, planning efficient execution up front and adjusting on the fly using real runtime stats. But they can only work with what you give them - hide your pipeline behind unnecessary disk writes or bury a filter after a join, and you take away the information they need to build a smart plan.

This article walks through 10 practices that consistently separate Spark jobs that scale from Spark jobs that fall over. Each one comes with a "before" (the anti-pattern) and "after" (the fix), because it's easier to internalize why something matters when you can see the shape of the mistake.

## 1. Column Pruning & Predicate Pushdown
If you filter rows or select columns after a join, Spark has already shuffled the full, unfiltered dataset across the network before doing any of that trimming. All that wasted movement could have been avoided. 

Spark can push filters and column selections down into the file scan itself (especially for Parquet/ORC) - but only if it can see that opportunity before the data gets tangled up in a join. Filtering after the join is too late.

### Example
**Before:**

**Problem -**
- Full datasets loaded into memory
- All columns from every dataset carried through the join
- Causes massive memory overhead & unnecessary I/O

**Impact -**
- Null rows participate in the join
- Wasted compute and network resources

<table>
<tr>
<td>

```
df = claims.join(customers, "customer_id", "inner") \
    .join(policies, "policy_id", "inner") \
    .join(hospitals, "hospital_id", "inner") \
    .join(payments, "claim_id", "left")

# Filter applied AFTER all joins - null rows already shuffled
df = df.filter(col("claim_amount").isNotNull())

```

</td>
<td>

![Performance optimization](images/spark_optimization_1_1.png)

</td>
</tr>
</table>



**After:**

**Benefit -**
- Drastically reduced memory footprint
- Nulls never enter the pipeline
<table>
<tr>
<td>

```
# Only required columns selected upfront, filter applied early
claims_df = (
    claims
    .select("claim_id", "customer_id", "policy_id", "hospital_id", "claim_amount")
    .filter(col("claim_amount").isNotNull())
)
customers_df = customers.select("customer_id", "city", "state")
df = claims_df.join(customers_df, "customer_id", "inner")
```
</td>
<td>

![performance_optimization_after](images/spark_optimization_1_2.png)

</td>
</tr>
</table>

> **Rule of thumb:** *Filter rows and trim columns before the join, not after. Same result, far less data shuffled.*

## 2. Avoid Cross Join unless one side is tiny
A cross join pairs every row on one side with every row on the other. 10,000 rows × 10,000 rows = 100M combinations - even if you only wanted matching categories. Without an actual join key, Spark can't use its efficient join strategies (like sort-merge join). It has no choice but to compute every possible pairing, which grows explosively as your data grows.

### Example

**Before:**
```
result = orders_df.crossJoin(products_df) \
.filter(orders_df.category == products_df.category)
```

**Problem -**
- Every row in orders_df is paired with every row in products_df before any filtering happens
- No join key means Spark can't use join strategies - it's forced into a brute-force nested loop
- The filter condition (category == category) reveals there was a real join key all along - it just applied too late

**Impact -**
- Combination count explodes: 10K orders x 10K products = 100M rows generated, most immediately discarded
- Massive shuffle and compute spent building rows that never survive the filter
- High risk of executor OOM or a stage that never finishes as data volumes grow

**After:**
```
filtered_products = products_df.filter(products_df.active == True) \
    .dropDuplicates(["category", "product_id"])

result = orders_df.join(
    filtered_products,
    orders_df.category == filtered_products.category,
    "inner"
)
```
**Benefits -**
- Spark can choose an efficient strategy (sort-merge or broadcast join) instead of brute-forcing every pairing
- Row count stays proportional to matching keys, not the full cross product
- Filtering inactive/duplicate products beforehand shrinks the build side further, reducing shuffle volume and memory pressure

> **Rule of thumb:** *If you're filtering right after a crossJoin, that filter condition is almost always meant to be your join key. Use join() with a real condition instead.*

## 3. Aggregate or deduplicate before joining
If your end goal is a summary (a sum, a count, an average), you join the raw data first and aggregate after the join processes every single row - even though most of that detail gets thrown away moments later. The cost of a join scale with how many rows go into it. Summarizing first shrinks the data before the expensive part even starts.

### Example

**Before:**
```
result = transactions_df.join(accounts_df, "account_id") \
    .groupBy("account_id") \
    .agg(F.sum("amount"). alias("total_amount"))
```

**Problem -**
- Join runs on every raw transaction row, not the final summarized rows
- Unnecessary columns participating in join can cause memory pressure and increase shuffle overhead

**Impact -**
- Shuffle scales with total transactions instead of distinct accounts
- Wasted compute joining detail that gets discarded right after

**After:**
```
agg_transactions = transactions_df.groupBy("account_id") \
    .agg(F.sum("amount").alias("total_amount"))

result = agg_transactions.join(accounts_df, "account_id")
```
**Benefits -**
- Join operates on far fewer rows - one per account, not one per transaction
- Smaller shuffle, less memory pressure, faster runtime
- Same result, since aggregation and join order don't change correctness here

> **Rule of thumb:** *If you're aggregating right after a join, ask whether you can aggregate first instead - the results are identical, but the join has far less work to do.*

## 4. Broadcast only genuinely small tables

A broadcast join sends a full copy of one table to every machine in the cluster, avoiding a shuffle entirely. This is a great trick - but only if the table is small. Broadcasting a multi-gigabyte table can crash every executor at once.
Broadcasting relies on the table, fitting comfortably in memory on every executor. If it doesn't, you get out-of-memory errors that can be confusing to diagnose because they don't look like a "size" problem at first glance.

## Example

**Before:**  All joins treated as sort-merge joins
```
# All joins are sort-merge - full shuffle on both sides
df = claims.join(customers, "customer_id", "inner") \
    .join(policies, "policy_id", "inner") \
    .join(hospitals, "hospital_id", "inner") \
    .join(payments, "claim_id", "left")
```
**Problem:**
- Small, static dimension tables (customers, policies, hospitals) get shuffled every time, even though they rarely change and are tiny compared to claims

**Impact:**
- Unnecessary shuffle cost on every join stage
- Slower runtime than needed, since Spark could have skipped the shuffle entirely for the small side


**After:** Broadcast small dimension tables
```
from pyspark.sql.functions import broadcast

# Dimension tables broadcasted - zero shuffle cost
base_df = (
    claims_df
    .join(broadcast(customers_df), "customer_id", "inner")
    .join(broadcast(policies_df),  "policy_id",   "inner")
    .join(broadcast(hospitals_df), "hospital_id", "inner")
    .join(payments_df, "claim_id", "left")  # Only this one shuffles
)
```

**Benefits:**
- No shuffle needed for the broadcast tables -huge speedup on those joins
- claims_df (the large side) never needs to move, only the small tables are copied out
- Leaves payments as the only real shuffle, since it's not a good broadcast candidate

> **Rule of thumb:** *If you're not 100% sure a table is small, don't force a broadcast - let Spark's built-in threshold and AQE make that call for you based on real data size.*

## 5. Detect & Fix Skewed Join keys
Sometimes one value in your join key - a specific customer ID, or a "null"/"unknown" placeholder - has way more rows than everything else combined. When that happens, one task ends up doing 80% of the work while all the other tasks finish and sit idle.

Why this happens: Spark distributes join work by hashing the key and assigning rows to partitions. If one key value dominates, its partition becomes enormous, and Spark's stage can't finish until that one slow task finishes.

### Example
"Salting" artificially spreads a hot key across multiple sub-keys, so the work gets distributed. If you're on Spark 3.x+, try enabling AQE's skew join handling first - it often solves this automatically.

**Before:** No skew handling
```
# No skew handling - hot keys overload single executors
customer_stats = df.groupBy("customer_id").agg(
    _sum("claim_amount").alias("total_claims"),
    count("*").alias("claim_count")
)
# If customer_id "C001" has 10M rows → single executor overloaded
```
**Problem:**
- One customer_id value dominates the dataset, so its partition is far larger than every other partition

**Impact:**
- One task processes the bulk of the data while all other tasks sit idle
- The whole stage waits for that single slow task, stretching runtime dramatically

   
**After:**  
1. Option 1 - let AQE handle it automatically
```
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

result = claims_df.join(customers_df, "customer_id", "inner")
```
2. Option 2 - Manual Salting
```
from pyspark.sql.functions import rand, floor, concat, lit, explode, array
NUM_SALTS = 10
# Spread the hot key across multiple sub-keys on the large side
claims_salted = claims_df.withColumn(
    "customer_salt",
    concat(col("customer_id"), lit("_"), floor(rand() * NUM_SALTS))
)
# Explode the small side so every salt bucket has a match
customers_salted = (
    customers_df
    .withColumn("salt_val", explode(array([lit(i) for i in range(NUM_SALTS)])))
    .withColumn("customer_salt", concat(col("customer_id"), lit("_"), col("salt_val")))
)
result = claims_salted.join(customers_salted, "customer_salt", "inner")
```
**Benefits:**
- Work for the hot key is spread across multiple tasks instead of one
- Stage completion time is driven by average load, not by the single worst partition
- Other tasks no longer sit idle waiting on the skewed one

> **Rule of thumb:** *Try AQE's skew join handling first on Spark 3.x+ - it often fixes this with zero code changes. Fall back to manual salting only if AQE isn't available or doesn't fully resolve it.*

## 6. Repartitioning the data

Repartitioning shuffles your data based on a chosen key - but that's only useful if the next operation uses that same key. If you repartition once early on and then just carry that same partitioning through every later step regardless of what those steps need, you're paying for shuffles that don't help.

Each operation (join, groupBy, window) needs data grouped by its own specific key to run efficiently. If the current partitioning doesn't match, Spark shuffles again anyway, so the earlier repartition was wasted effort.

### Example
**Before:** Repartitioned by a key that only helps the join, not the window function that follows
```
# Repartitioned early by policy_id for the upcoming join...
claims_df = claims_df.repartition("policy_id")

result = claims_df.join(policies_df, "policy_id", "inner")

# ...but the very next step is a window function partitioned by hospital_id
# Partitioning doesn't match - Spark shuffles AGAIN to satisfy the window
w = Window.partitionBy("hospital_id").orderBy(col("claim_amount").desc())

ranked = result.withColumn("claim_rank", rank().over(w))
```

**Problem:**
- Data is repartitioned on policy_id to prep for the join, but the window function right after needs data partitioned by hospital_id
- The repartition only serves the join step - it doesn't consider what happens downstream

**Impact:**
- Spark shuffles a second time to satisfy the window function's partitioning requirement
- The policy_id repartition is essentially thrown away - it helped the join, but that benefit doesn't carry forward
- Two shuffles executed where the pipeline could have been arranged to need fewer

**After:** Repartition on the key that serves the full downstream chain
```
# Join first without a manual repartition - let Spark handle it based on join keys
result = claims_df.join(policies_df, "policy_id", "inner")

# Repartition once, directly on the key the window function actually needs
result = result.repartition("hospital_id")

w = Window.partitionBy("hospital_id").orderBy(col("claim_amount").desc())
ranked = result.withColumn("claim_rank", rank().over(w))
```

**Benefits:**
- Only one repartition shuffle happens, right before the step that needs it
- The window function runs on correctly-partitioned data immediately no extra shuffle
- Lower total shuffle volume and faster runtime for the same result

> **Rule of thumb:** *Before repartitioning, always look ahead: what's the very next expensive operation, and what key does it actually need?*

## 7. Never collect() large results

collect() pulls all the data from across the cluster back onto a single machine - the driver. This works fine for small results, but pulling millions of rows onto one machine will exhaust its memory and crash the job.

The driver typically has far less memory available than the combined memory of all your executors. Asking it to hold everything defeats the whole purpose of distributed computing.

### Example

**Before:** Pulling the full dataset onto the driver
```
# Pulls every row back to a single machine - crashes for large claims_df
all_claims = claims_df.collect()
for row in all_claims:
    process(row)
```
**Problem:**
- All of claims_df gets copied onto the driver, no matter how large it is

**Impact:**
- Driver runs out of memory and crashes, often with no obvious link back to collect()
- Loses the benefit of distributed processing entirely

**After:** Keep the data distributed, flow it to a sink
```
# Data stays distributed, written straight to a sink
claims_df.write.mode("overwrite").parquet("s3://bucket/claims_output/")

# If you genuinely need a small, bounded sample - limit() first, then collect
sample_rows = claims_df.limit(100).collect()
```

**Benefits:**
- Writes scale to any data size without risking driver OOM
- Work stays parallelized across the cluster
- limit() before collect() gives a safe way to sample data

> **Rule of thumb:** *Think of the driver as a coordinator, not a worker. Data should stay distributed and flow to a sink - it shouldn't round-trip through a single machine unless it's genuinely small and bounded*

## 8. Caching & Persisting
Caching keeps data in memory, so you don't have to recompute it every time you use it - great when you reuse the same DataFrame multiple times. But if you never release that memory (unpersist()), it sits there indefinitely, crowding out memory that later stages need for their own work. 

Cached data occupies "storage memory" in each executor. If it's never released, there's less memory available for the "execution memory" that joins, sorts, and aggregations need - increasing the chance those operations spill to disk and slow down.

### Example

**Before:**
```
customer_stats = df.groupBy("customer_id").agg(
    _sum("claim_amount").alias("total_claims"),
    count("*").alias("claim_count")
) # Recompute #1

hospital_stats = df.groupBy("hospital_id").agg(
    _sum("claim_amount").alias("hospital_claims")
) # Recompute #2 - df is re-read and re-processed from the start
```

**Problem:**
- df (and whatever transformation chain built it) is recomputed from scratch for every downstream action that uses it
- Spark has no memory of the earlier work - each groupBy triggers its own full read/transform pass
Impact:
- The same expensive upstream computation (joins, filters, salting, etc.) runs twice, tripling cost if reused a third time
- Wasted CPU and I/O redoing identical work
- Runtime grows linearly with the number of reuses instead of staying flat

**After:** Persist once, reuse, then release
```
from pyspark import StorageLevel

# Persist after the expensive joins/salting, before multiple reuses
salted_df = salted_df.persist(StorageLevel.MEMORY_AND_DISK)

customer_stats = salted_df.groupBy("customer_id").agg(
    _sum("claim_amount").alias("total_claims"),
    count("*").alias("claim_count")
)
hospital_stats = salted_df.groupBy("hospital_id").agg(
    _sum("claim_amount").alias("hospital_claims")
)

# Release memory the moment you're done reusing it
salted_df.unpersist()
```

**Benefits:**
- The expensive upstream chain runs once, no matter how many times salted_df is reused afterward
- unpersist() frees storage memory immediately, leaving more room for execution memory in later stages
- Reduces the risk of spills during joins/sorts/aggregations that come after this point in the pipeline

Remember to use unpersist() after final usage to release memory

> **Rule of thumb:** *When you cache something, mentally mark the point where you're done with it - and call unpersist() right there, not at the end of the script (or never, which is what usually happens by accident).*

## 9. Minimize Materialization & Execution
Every time you write data to disk and then read it back in the middle of a pipeline, you create a hard stop that Spark's optimizer can't see across. You also pay real disk I/O costs that add up fast - disk and network operations are dramatically slower than staying in memory.

Catalyst works by looking at your entire chain of transformations and finding the most efficient way to execute all of it together. Once you write to disk and read it back, that chain is broken into two separate plans that can't be optimized jointly.

### Example
**Before:** 
```
unnecessary intermediate write breaks the plan in two
claims_df.filter(col("claim_amount").isNotNull()) \
    .write.mode("overwrite").parquet("/tmp/staging/claims_filtered")

# Reading it back forces a brand-new, disconnected execution plan
staged_df = spark.read.parquet("/tmp/staging/claims_filtered")
result = staged_df.join(customers_df, "customer_id")
```
**Problem:**
- The write and subsequent read split what could be one logical plan into two separate, disconnected plans
- Catalyst optimizes each plan in isolation - it never sees the filter and the join as part of the same chain
Impact:
- Disk I/O cost is paid twice - once to write the intermediate data, once to read it back
- Catalyst can't optimize across the write boundary, so cross-step optimizations (like pushing the filter further down or choosing a join strategy based on filtered size) are lost
- The extra disk round-trip plus the missed optimizations combine to slow down end-to-end runtime

**After:** one continuous pipeline, optimized as a whole
```
# Catalyst can see the filter and join together and optimize both at once
result = (
    claims_df
    .filter(col("claim_amount").isNotNull())
    .join(customers_df, "customer_id")
)
```
**Benefits:**
- Catalyst sees the filter and join as one unit and can optimize them jointly (e.g., pushing the filter into the scan, choosing the best join strategy based on real filtered size)
- No intermediate disk write/read round-trip, saving real I/O time
- Fewer moving parts in the pipeline, one plan instead of two disconnected ones to reason about and debug

> **Rule of thumb:** *Let Spark see the whole pipeline before it has to hit disk. Only materialize intermediate results when you genuinely need to (checkpointing for fault tolerance, sharing across separate jobs) - not as a default habit*

## 10. Avoid unnecessary global sorts & repeated calculations
Sorting your entire dataset (orderBy) requires a full shuffle to get everything in the right order across the cluster - it's one of the most expensive things you can ask Spark to do. Doing this more than once or recalculating the same derived column or window function in multiple places, means paying that cost repeatedly for no added benefit.

Spark doesn't automatically notice that two separately written expressions are computing the exact same thing - it will happily redo the same expensive work if you write it twice, even if the result would be identical.

## Example

**Before:** sorted too early, window function computed twice
```
from pyspark.sql import Window
from pyspark.sql.functions import rank

w = Window.partitionBy("customer_id").orderBy(col("claim_amount").desc())

# Global sort run early, before filtering has even shrunk the data
sorted_df = claims_df.orderBy("claim_amount")

# Same window function recomputed independently in two branches
high_value = sorted_df.withColumn("rank", rank().over(w)).filter(col("rank") == 1)
claims_per_customer = sorted_df.withColumn("rank", rank().over(w)).groupBy("customer_id").count()
```

**Problem:**
- A full global sort runs before the data has even been trimmed down
- The same window function is written twice and computed twice, once per branch
Impact:
- Extra full-shuffle sorts that isn't needed this early in the pipeline
- The expensive window computation runs twice for identical results; Spark has no idea the two are the same

**After:** compute once, reuse, sort last
```
# Compute the window function once, reuse the result across branches
ranked_df = claims_df.withColumn("rank", rank().over(w)).cache()

high_value = ranked_df.filter(col("rank") == 1)
claims_per_customer = ranked_df.groupBy("customer_id").count()

# Global sort applied once, at the very end, only on the final output
final_output = high_value.orderBy(col("claim_amount").desc())

ranked_df.unpersist()
```

**Benefits:**
- Window function runs once instead of twice — same result, half the compute
- Global sort happens only once, on the smaller final output instead of the full dataset
- Fewer shuffles overall, leading to a faster pipeline

> **Rule of thumb:** *Compute shared logic once and reuse the result across branches. Save global sorting for the very end of the pipeline, and only when the final output genuinely requires strict ordering.*

## Summary

Spark performance is ultimately about doing less work, moving less data, and using resources wisely. The best optimizations often come from simple choices: filter early, join intelligently, avoid unnecessary shuffles, keep data distributed, and let Catalyst and AQE do what they do best. Write Spark code with the execution plan in mind not just the result.

**Quick Reference:**

| S.No. | Anti-pattern | Fix | Spark mechanism |
|--------|-------------|-----|-----------------|
| 1 | Filter/select after join | Filter/select before join | Predicate & column pushdown |
| 2 | `crossJoin` + filter | `join()` with a real key | Sort-merge / hash join |
| 3 | Aggregate after join | Aggregate before join | Shrinks join input size |
| 4 | Force broadcast on unverified table | Verify size / trust AQE | Broadcast hash join |
| 5 | Skewed key → one slow task | Salting / AQE skew handling | Partition rebalancing |
| 6 | Repartition key ≠ next op's key | Repartition on the key actually needed next | Avoid redundant shuffle |
| 7 | `collect()` on large data | Write to sink / `limit()` | Driver memory limits |
| 8 | `cache()` never released | `unpersist()` right after last use | Storage vs. execution memory |
| 9 | Intermediate disk read/write | Keep pipeline as one continuous plan | Catalyst whole-plan optimization |
| 10 | Repeated sort/window calc | Compute once, sort last | Avoid redundant shuffle & recompute |
 




