---
name: taxi-duckdb
description: Answer questions about NYC yellow-taxi trips (January 2024) by running SQL against the local DuckDB file demo_taxi.duckdb. Use whenever the user asks about taxi trips, fares, tips, pickup or dropoff locations, boroughs, zones, busiest hours, or trip distances.
---

# NYC taxi demo database (DuckDB)

One month of NYC yellow-taxi trips in a local DuckDB file. Answer questions by writing SQL,
running it, and summarising the result in a sentence or two plus a small table.

## How to run a query

In a notebook cell (Jupyter AI) — open the connection once and reuse it:

```python
import duckdb, os
con = duckdb.connect(os.path.expanduser("~/demo_taxi.duckdb"), read_only=True)
con.sql("""<SQL>""").df()
```

From a terminal: `duckdb -readonly ~/demo_taxi.duckdb "<SQL>"` (add `-markdown` for a table).

- Always `read_only=True` / `-readonly`. Never CREATE, INSERT, UPDATE, DELETE or DROP.
- Always add `LIMIT` (default 20) unless the query is an aggregate that returns a few rows.
- If the file is "locked", the loading notebook still has it open; ask the user to close it.

## DuckDB 2.x API notes (the hub runs DuckDB 2.0)

- On a relation use `.fetchall()`, `.df()` or `.show()`; `.fetchone()` and `.fetchmany()` no longer exist. A count is `con.sql("SELECT count(*) FROM t").fetchall()[0][0]`.
- Lambdas use Python-style syntax: `list_filter(xs, lambda x: x > 1)`, not `x -> x > 1`.
- `con.sql(...)` is lazy; nothing runs until you fetch, `.df()` or `.show()`.

## Tables

### `trips` — one row per taxi trip, ~3 million rows

| column | meaning |
|---|---|
| `tpep_pickup_datetime`, `tpep_dropoff_datetime` | trip start / end (TIMESTAMP) |
| `passenger_count` | people in the cab (0 and NULL occur; treat as unknown) |
| `trip_distance` | miles |
| `PULocationID`, `DOLocationID` | pickup / dropoff zone IDs → join to `zones.LocationID` |
| `fare_amount` | metered fare, dollars |
| `tip_amount` | tip, dollars — **only recorded for credit-card trips**, so tip analyses must filter `payment_type = 1` |
| `tolls_amount`, `congestion_surcharge`, `Airport_fee` | extras |
| `total_amount` | what the rider paid in total |
| `payment_type` | 1 = credit card, 2 = cash, 3 = no charge, 4 = dispute |
| `RatecodeID` | 1 = standard, 2 = JFK flat rate, 3 = Newark, 5 = negotiated |

### `zones` — 265 rows, the lookup for location IDs

`LocationID`, `Borough` (Manhattan, Brooklyn, Queens, Bronx, Staten Island, EWR), `Zone`
(neighbourhood name, e.g. `JFK Airport`, `Upper East Side North`), `service_zone`.

## Rules of thumb

1. **Join to `zones` for anything about places.** Users say "JFK", "Manhattan", "Brooklyn";
   the data has IDs. Pickup → `PULocationID`, dropoff → `DOLocationID`; alias the two joins
   (`pu`, `d`) when both are needed.
2. **Guard against bad rows.** The data has a few trips with negative or zero fares, 0
   passengers, or timestamps outside January. For averages add `WHERE fare_amount > 0`; for
   durations add `tpep_dropoff_datetime > tpep_pickup_datetime`.
3. **Tip percentage** = `tip_amount / nullif(fare_amount, 0)`, credit-card trips only.
4. **Duration** in minutes = `date_diff('minute', tpep_pickup_datetime, tpep_dropoff_datetime)`.
5. **Time of day / day of week**: `hour(tpep_pickup_datetime)`, `dayname(tpep_pickup_datetime)`.
6. **Thresholds ("how many trips over $X")**: sweep several values in one query with
   `count(*) FILTER (WHERE total_amount >= th.threshold)` over a `VALUES` list, so the user
   sees the curve, not one number.
7. Prefer `round(..., 1)` on money and percentages in the output.

## Templates

**Busiest hours**
```sql
SELECT hour(tpep_pickup_datetime) AS hour, count(*) AS trips
FROM trips GROUP BY 1 ORDER BY trips DESC LIMIT 5;
```

**Top pickup zones**
```sql
SELECT z.Borough, z.Zone, count(*) AS trips
FROM trips t JOIN zones z ON z.LocationID = t.PULocationID
GROUP BY 1, 2 ORDER BY trips DESC LIMIT 10;
```

**Between two places**
```sql
SELECT count(*) AS trips, round(avg(total_amount), 2) AS avg_total, round(avg(trip_distance), 1) AS avg_miles,
       round(avg(date_diff('minute', tpep_pickup_datetime, tpep_dropoff_datetime)), 0) AS avg_minutes
FROM trips t JOIN zones pu ON pu.LocationID = t.PULocationID
             JOIN zones d  ON d.LocationID  = t.DOLocationID
WHERE pu.Zone = 'JFK Airport' AND d.Borough = 'Manhattan'
  AND tpep_dropoff_datetime > tpep_pickup_datetime;
```

**Tip by passenger count**
```sql
SELECT passenger_count, count(*) AS trips, round(100 * avg(tip_amount / nullif(fare_amount, 0)), 1) AS avg_tip_pct
FROM trips WHERE payment_type = 1 AND fare_amount > 0 AND passenger_count BETWEEN 1 AND 6
GROUP BY 1 ORDER BY 1;
```

**Threshold sweep**
```sql
SELECT th.threshold, count(*) FILTER (WHERE total_amount >= th.threshold) AS trips
FROM trips, (VALUES (10), (25), (50), (100), (200)) th(threshold)
GROUP BY 1 ORDER BY 1;
```

## Answering

Lead with the answer in words ("The busiest hour is 6 pm with 210k pickups"), then the table.
Mention any filter you applied (e.g. "credit-card trips only"). Offer a chart when the result is a
trend over hours/days or a threshold sweep.
