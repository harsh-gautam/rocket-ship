# 10 Spark Anti-Patterns That Kill Performance (And How to Fix Them)

## Collab

## Introduction
If you've spent any real time running Spark jobs in production, you've hit at least one of these walls: a job that silently balloons from 10 minutes to 4 hours, a driver that OOMs for no obvious reason, or a single task that runs 100x longer than every other task in its stage. These aren't bad luck - they're almost always the result of a handful of recurring anti-patterns.

Spark's Catalyst optimizer and Adaptive Query Execution (AQE) work hard to prevent exactly this, planning efficient execution up front and adjusting on the fly using real runtime stats. But they can only work with what you give them - hide your pipeline behind unnecessary disk writes or bury a filter after a join, and you take away the information they need to build a smart plan.

This article walks through 10 practices that consistently separate Spark jobs that scale from Spark jobs that fall over. Each one comes with a "before" (the anti-pattern) and "after" (the fix), because it's easier to internalize why something matters when you can see the shape of the mistake.

## 1. Column Pruning & Predicate Pushdown
If you filter rows or select columns after a join, Spark has already shuffled the full, unfiltered dataset across the network before doing any of that trimming. All that wasted movement could have been avoided. 

Spark can push filters and column selections down into the file scan itself (especially for Parquet/ORC) - but only if it can see that opportunity before the data gets tangled up in a join. Filtering after the join is too late.

### Example
**Before**

**Problem**
- Full datasets loaded into memory
- All columns from every dataset carried through the join
- Causes massive memory overhead & unnecessary I/O

**Impact**
- Null rows participate in the join
- Wasted compute and network resources

```
df = claims.join(customers, "customer_id", "inner") \
    .join(policies, "policy_id", "inner") \
    .join(hospitals, "hospital_id", "inner") \
    .join(payments, "claim_id", "left")

# Filter applied AFTER all joins - null rows already shuffled
df = df.filter(col("claim_amount").isNotNull())

```

**After**

**Benefit**
- Drastically reduced memory footprint
- Nulls never enter the pipeline

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

**Rule of thumb:** *Filter rows and trim columns before the join, not after. Same result, far less data shuffled.*

## 2. Avoid Cross Join unless one side is tiny
A cross join pairs every row on one side with every row on the other. 10,000 rows × 10,000 rows = 100M combinations - even if you only wanted matching categories. Without an actual join key, Spark can't use its efficient join strategies (like sort-merge join). It has no choice but to compute every possible pairing, which grows explosively as your data grows.

### Example

**Before**
```
result = orders_df.crossJoin(products_df) \
.filter(orders_df.category == products_df.category)
```

**Problem**
- Every row in orders_df is paired with every row in products_df before any filtering happens
- No join key means Spark can't use join strategies - it's forced into a brute-force nested loop
- The filter condition (category == category) reveals there was a real join key all along - it just applied too late

**Impact**
- Combination count explodes: 10K orders x 10K products = 100M rows generated, most immediately discarded
- Massive shuffle and compute spent building rows that never survive the filter
- High risk of executor OOM or a stage that never finishes as data volumes grow

**After**
```
filtered_products = products_df.filter(products_df.active == True) \
    .dropDuplicates(["category", "product_id"])

result = orders_df.join(
    filtered_products,
    orders_df.category == filtered_products.category,
    "inner"
)
```
**Benefits**
- Spark can choose an efficient strategy (sort-merge or broadcast join) instead of brute-forcing every pairing
- Row count stays proportional to matching keys, not the full cross product
- Filtering inactive/duplicate products beforehand shrinks the build side further, reducing shuffle volume and memory pressure

**Rule of thumb:** *If you're filtering right after a crossJoin, that filter condition is almost always meant to be your join key. Use join() with a real condition instead.*

## 3. Aggregate or deduplicate before joining



