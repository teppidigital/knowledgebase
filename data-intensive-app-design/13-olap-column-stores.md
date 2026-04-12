# OLAP and Column Stores

## Category

DDIA — Foundations (Chapter 3 extended — Analytical Processing)

## Context

OLTP (Online Transaction Processing) and OLAP (Online Analytical Processing) have fundamentally different access patterns, and this drives fundamentally different storage engine designs.

### OLTP vs OLAP

| Property | OLTP | OLAP |
|---|---|---|
| **Query pattern** | Point reads/writes on individual rows | Aggregate over millions of rows, few columns |
| **Data volume per query** | Small (KB) | Large (GB to TB) |
| **Latency requirement** | < 10ms per request | Seconds to minutes acceptable |
| **Write pattern** | High-frequency small inserts/updates | Bulk load (nightly ETL, streaming insert) |
| **Who uses it** | Application end users | Data analysts, BI tools |
| **Storage layout** | Row-oriented | Column-oriented |
| **Examples** | PostgreSQL, MySQL, Oracle | BigQuery, Redshift, ClickHouse, Snowflake |

### Column-Oriented Storage

In a row store, all columns of a single row are stored together. In a column store, all values for a single column are stored together.

**Why this matters for analytics:**

```sql
SELECT region, SUM(amount)
FROM orders
WHERE status = 'COMPLETED'
  AND created_at >= '2024-01-01'
GROUP BY region;
```

This query touches only 3 of perhaps 30 columns. A row store reads all 30 columns per row (wasted I/O). A column store reads only `region`, `amount`, `status`, `created_at` — typically 4x to 100x less I/O.

### Column Compression

Because all values in a column have the same type and often similar values, columns compress extremely well:

| Technique | How it works | Best for | Compression ratio |
|---|---|---|---|
| **Run-length encoding (RLE)** | Store (value, count) pairs | Low-cardinality sorted columns | 100x+ |
| **Bitmap encoding** | One bit per row for each distinct value | Low-cardinality columns (region, status) | 10-50x |
| **Dictionary encoding** | Map strings to integers; store integers | String columns with repeated values | 5-20x |
| **Delta encoding** | Store difference from previous value | Time series, monotonically increasing IDs | 2-10x |
| **Parquet (hybrid)** | Dictionary + bit-packing + RLE | All analytics use cases | 5-10x vs raw |

### Vectorised Execution

Column stores can operate on entire columns as arrays (vectors) rather than row-by-row. Modern CPUs have SIMD (Single Instruction, Multiple Data) instructions that apply the same operation to 8-16 values simultaneously.

- `SUM(amount)` → sum an array of doubles using SIMD AVX instructions
- `WHERE status = 'COMPLETED'` → bitwise AND of a bitmap column

This gives 10-100x CPU throughput improvement vs row-by-row evaluation.

### Sort Keys and Data Skipping

Sorting data within a column store (by sort key) enables **data skipping**:
- BigQuery: partitioning + clustering; BigQuery skips entire partitions that don't match WHERE clause
- Redshift: SORTKEY; queries on the sort key skip entire blocks
- ClickHouse: ORDER BY; the primary index stores every Nth row pointer (sparse index)

A well-chosen sort key can reduce data scanned by 10-1000x.

## Pros

- Column stores are 10-1000x faster than row stores for analytical queries
- High compression ratios reduce storage cost and increase cache efficiency
- Vectorised execution leverages CPU SIMD capabilities
- Data skipping (partition pruning, sparse indexes) reduces I/O further

## Cons

- Column stores are slow for single-row lookups and point writes
- Row inserts require touching many column files — not suitable for high-frequency OLTP writes
- Wide tables are a better fit than narrow tables (more benefit from selecting few columns of many)
- Schema design (sort keys, partitioning columns, clustering) has major performance impact

## Design Diagram

```mermaid
flowchart TD
    subgraph Row Store — PostgreSQL
        R1[Row1: ord-1 | EU | 150 | COMPLETED | 2024-01-01]
        R2[Row2: ord-2 | US | 200 | PENDING | 2024-01-02]
        R3[Row3: ord-3 | EU | 300 | COMPLETED | 2024-01-02]
        QUERY1["SELECT region, SUM(amount) WHERE status='COMPLETED'<br/>Must read ALL columns of ALL rows"]
    end

    subgraph Column Store — ClickHouse / BigQuery
        Col1["region: [EU, US, EU, ...]"]
        Col2["amount: [150, 200, 300, ...]"]
        Col3["status: [COMP, PEND, COMP, ...]<br/>→ bitmap: [1, 0, 1, ...]"]
        QUERY2["SELECT region, SUM(amount) WHERE status='COMPLETED'<br/>Read ONLY 3 columns; skip status=PENDING rows entirely"]
    end

    subgraph Compression
        RAW["status column raw: COMPLETED PENDING COMPLETED COMPLETED PENDING<br/>(50 bytes)"]
        DICT["Dictionary: {C=0, P=1}<br/>Data: [0,1,0,0,1]<br/>Bit-packed: 5 bits total<br/>(10x compression)"]
        RAW --> DICT
    end
```

## Code Sample

### Parquet-like column reader (conceptual)

```typescript
// Column-oriented in-memory storage — demonstrates the read pattern

interface ColumnStore {
  columns: Map<string, unknown[]>;
  rowCount: number;
}

function buildColumnStore(rows: Record<string, unknown>[]): ColumnStore {
  if (rows.length === 0) return { columns: new Map(), rowCount: 0 };

  const columns = new Map<string, unknown[]>();
  for (const key of Object.keys(rows[0])) {
    columns.set(key, rows.map(r => r[key]));
  }
  return { columns, rowCount: rows.length };
}

// Analytical query: reads only the needed columns
function columnQuery(
  store: ColumnStore,
  selectColumns: string[],
  where: { column: string; equals: unknown },
  aggregate: { column: string; fn: 'SUM' | 'COUNT' | 'AVG'; groupBy: string }
): Map<unknown, number> {
  const filterCol = store.columns.get(where.column)!;
  const aggCol = store.columns.get(aggregate.column)!;
  const groupCol = store.columns.get(aggregate.groupBy)!;

  // Filter (bitmap predicate) — in a real engine this is vectorised
  const matchedRows: number[] = [];
  for (let i = 0; i < store.rowCount; i++) {
    if (filterCol[i] === where.equals) matchedRows.push(i);
  }

  // Group + aggregate
  const result = new Map<unknown, number>();
  for (const row of matchedRows) {
    const groupKey = groupCol[row];
    const value = aggCol[row] as number;
    result.set(groupKey, (result.get(groupKey) ?? 0) + value);
  }
  return result;
}

// Example
const rows = [
  { region: 'EU', amount: 150, status: 'COMPLETED' },
  { region: 'US', amount: 200, status: 'PENDING' },
  { region: 'EU', amount: 300, status: 'COMPLETED' },
  { region: 'US', amount: 75,  status: 'COMPLETED' },
];

const store = buildColumnStore(rows);
const revenueByRegion = columnQuery(
  store,
  ['region', 'amount'],
  { column: 'status', equals: 'COMPLETED' },
  { column: 'amount', fn: 'SUM', groupBy: 'region' }
);
console.log([...revenueByRegion]); // [['EU', 450], ['US', 75]]
```

### Run-length encoding

```typescript
function runLengthEncode(values: string[]): Array<[string, number]> {
  if (values.length === 0) return [];
  const encoded: Array<[string, number]> = [];
  let current = values[0];
  let count = 1;

  for (let i = 1; i < values.length; i++) {
    if (values[i] === current) {
      count++;
    } else {
      encoded.push([current, count]);
      current = values[i];
      count = 1;
    }
  }
  encoded.push([current, count]);
  return encoded;
}

function runLengthDecode(encoded: Array<[string, number]>): string[] {
  return encoded.flatMap(([value, count]) => Array(count).fill(value));
}

// Sorted status column — compresses very well
const statusColumn = [
  'CANCELLED', 'CANCELLED', 'CANCELLED',
  'COMPLETED', 'COMPLETED', 'COMPLETED', 'COMPLETED', 'COMPLETED',
  'PENDING',   'PENDING',
  'SHIPPED',   'SHIPPED',   'SHIPPED'
];

const encoded = runLengthEncode(statusColumn);
// [['CANCELLED', 3], ['COMPLETED', 5], ['PENDING', 2], ['SHIPPED', 3]]
// 13 string values → 4 tuples: 4x compression (better on real data with thousands of rows)
```

## Key Patterns

### BigQuery / ClickHouse Schema Design

| Decision | Options | Guidance |
|---|---|---|
| **Partition column** | Date, month, integer range | Always partition by date/event_time for time-series queries |
| **Cluster/Sort key** | Most common WHERE column(s) | Choose highest-cardinality column used in WHERE clauses |
| **Denormalise** | Nested/repeated fields vs joins | Denormalise frequently joined tables; avoid expensive cross-table joins |
| **Column types** | Wide enum vs string | Use enums/integers for status columns — better compression than strings |
| **Materialized views** | Pre-aggregate hot queries | Pre-compute daily/hourly aggregates for dashboard queries |

### OLAP Query Optimisation Checklist

```sql
-- ✅ Good: use partition column in WHERE clause (skips most data)
SELECT region, SUM(amount)
FROM orders
WHERE DATE(created_at) >= '2024-01-01'  -- partition pruning
  AND status = 'COMPLETED'               -- column predicate
GROUP BY region;

-- ❌ Bad: no partition filter (scans entire table)
SELECT region, SUM(amount)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY region;
```

## Related Patterns

- [03 — Storage and Retrieval](./03-storage-retrieval.md) — B-tree vs LSM-tree (OLTP storage engines)
- [10 — Batch Processing](./10-batch-processing.md) — Batch output lands in OLAP stores
- [11 — Stream Processing](./11-stream-processing.md) — Real-time inserts into ClickHouse/BigQuery
- [15 — Data Architecture Patterns](./15-data-architecture-patterns.md) — OLAP as the analytical tier in Lambda/Kappa
