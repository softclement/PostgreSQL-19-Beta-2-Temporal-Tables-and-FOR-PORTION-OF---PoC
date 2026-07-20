# PostgreSQL 19 Beta 2 – Temporal Tables and FOR PORTION OF (PoC)

## Overview

This repository contains a Proof of Concept (PoC) demonstrating the new temporal data management capabilities introduced in PostgreSQL 19 Beta 2.

The primary focus is testing:

- `WITHOUT OVERLAPS`
- Temporal Primary Keys
- `FOR PORTION OF` for `UPDATE`
- Automatic range splitting
- Date range (`daterange`) based validity management

Using these features, PostgreSQL can automatically maintain historical periods without requiring complex application logic, triggers, or manual row splitting.

---

## Environment

### Host

- Windows 11
- WSL2 (Ubuntu)

### Container Runtime

- Docker Desktop
- Docker Engine running inside WSL

### Database

```sql
PostgreSQL 19beta2
```

Verified using:

```sql
SELECT version();
```

Output:

```text
PostgreSQL 19beta2 (Debian 19~beta2-1.pgdg13+1)
```

---

# Why This Feature Matters

Many enterprise systems manage data that is valid only for specific periods:

- Product pricing
- Employee salaries
- Insurance policies
- Subscription plans
- Exchange rates
- Contracts
- Promotions

Historically applications had to:

1. Detect date overlaps
2. Split existing records
3. Insert replacement periods
4. Maintain historical integrity

PostgreSQL 19 introduces native temporal operations that automate these activities.

---

# Pull PostgreSQL 19 Beta 2

Download image:

```bash
docker pull postgres:19beta2
```

---

# Start PostgreSQL Container

```bash
docker run -d \
  --name pg19beta \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=testdb \
  -p 5432:5432 \
  postgres:19beta2
```


Connect:

```bash
docker exec -it pg19beta psql -U postgres -d testdb
```

---

# Create Extension

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

---

# Create Temporal Table

```sql
CREATE TABLE product_prices (
    product_id int,
    valid_range daterange NOT NULL,
    price numeric(10,2),

    PRIMARY KEY (
        product_id,
        valid_range WITHOUT OVERLAPS
    )
);
```

---

# Insert Initial Record

```sql
INSERT INTO product_prices
VALUES (
    1,
    daterange(
        '2026-01-01',
        '2027-01-01',
        '[)'
    ),
    100
);
```

---

# Verify Initial State

```sql
SELECT * FROM product_prices;
```

Output:

```text
 product_id |       valid_range       | price
------------+-------------------------+--------
          1 | [2026-01-01,2027-01-01) | 100.00
```

---

# Test 1 - UPDATE FOR PORTION OF

Business Requirement:

Increase the price from:

```text
2026-06-01
```

to

```text
2026-09-01
```

Price:

```text
100 -> 120
```

Execute:

```sql
UPDATE product_prices
FOR PORTION OF valid_range
    FROM DATE '2026-06-01'
    TO DATE '2026-09-01'
SET price = 120
WHERE product_id = 1;
```

---

# Results

```sql
SELECT *
FROM product_prices
ORDER BY lower(valid_range);
```

Output:

```text
 product_id | valid_range               | price
------------+--------------------------+--------
 1          | [2026-01-01,2026-06-01)  | 100.00
 1          | [2026-06-01,2026-09-01)  | 120.00
 1          | [2026-09-01,2027-01-01)  | 100.00
```

Notice:

PostgreSQL automatically split the original period into three logical segments.

No manual inserts or updates required.

---

# Test 2 - Nested Temporal Update

Business Requirement:

Run a special promotion only in July.

Period:

```text
2026-07-01 -> 2026-08-01
```

Price:

```text
120 -> 99
```

Execute:

```sql
UPDATE product_prices
FOR PORTION OF valid_range
    FROM DATE '2026-07-01'
    TO DATE '2026-08-01'
SET price = 99
WHERE product_id = 1;
```

---

# Results

```sql
SELECT *
FROM product_prices
ORDER BY lower(valid_range);
```

Output:

```text
 product_id | valid_range               | price
------------+--------------------------+--------
 1          | [2026-01-01,2026-06-01)  | 100.00
 1          | [2026-06-01,2026-07-01)  | 120.00
 1          | [2026-07-01,2026-08-01)  |  99.00
 1          | [2026-08-01,2026-09-01)  | 120.00
 1          | [2026-09-01,2027-01-01)  | 100.00
```

Again, PostgreSQL automatically handled all timeline splitting.

---

# Real-Time Price Lookup

## Find Price for Any Date

Example:

```sql
SELECT price
FROM product_prices
WHERE product_id = 1
AND valid_range @> DATE '2026-07-15';
```

Output:

```text
99.00
```

Because:

```text
[2026-07-01,2026-08-01)
```

contains:

```text
2026-07-15
```

---

## Current Price

```sql
SELECT price
FROM product_prices
WHERE product_id = 1
AND valid_range @> CURRENT_DATE;
```

---

# Comparison With Traditional Approach

## Traditional Design

```sql
CREATE TABLE product_price (
    product_id int,
    start_date date,
    end_date date,
    price numeric
);
```

Query:

```sql
SELECT price
FROM product_price
WHERE CURRENT_DATE
BETWEEN start_date
    AND end_date;
```

### Challenges

- Overlapping periods possible
- Requires application validation
- Requires custom logic for splitting periods
- History maintenance becomes complex

---

## PostgreSQL 19 Temporal Design

```sql
CREATE TABLE product_prices (
    product_id int,
    valid_range daterange,
    price numeric,

    PRIMARY KEY (
        product_id,
        valid_range WITHOUT OVERLAPS
    )
);
```

### Benefits

- Built-in overlap protection
- Temporal integrity
- Automatic period splitting
- Simplified application logic
- SQL-standard temporal functionality

---

# Example Use Cases

This feature is particularly useful for:

- Retail product pricing
- Employee salary revisions
- Insurance premium history
- Subscription plans
- Membership tiers
- Tax slabs
- Currency conversion rates
- Utility billing tariffs
- Promotional campaigns
- Contract management

---

# Key Observations

✅ PostgreSQL automatically splits overlapping periods

✅ No custom trigger required

✅ No procedural code required

✅ History is preserved automatically

✅ Range-based querying is simple and efficient

✅ Prevents temporal overlap anomalies

---

# Conclusion

PostgreSQL 19 Beta 2 introduces a significant enhancement for temporal data management through:

```sql
FOR PORTION OF
```

Combined with:

```sql
WITHOUT OVERLAPS
```

and native range types:

```sql
daterange
tsrange
tstzrange
```

it becomes much easier to maintain historical business data while preserving data integrity.

This PoC successfully validated:

- Temporal constraints
- Automated range splitting
- Historical preservation
- Point-in-time querying

and demonstrates how PostgreSQL can now manage temporal business data with significantly less application-side complexity.
