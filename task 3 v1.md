
# Task 3: Reporting and Analytics

Design queries to generate the following reports:

1.  Daily docking/undocking report for a specific port (count, average docking duration, etc.).
2.  Top 10 busiest docking areas for a given week.
3.  Historical analysis of docking trends (e.g., monthly summaries).

Deliverables:

1.  SQL queries for generating reports.
6.  Explanation of how to optimize for low latency.
# AIS Reporting and Analytics Solution

## 1. Daily Docking/Undocking Report

The following query generates a daily report for a specific port. It calculates the number of vessels that docked and undocked, along with their average docking duration. We identify docking events when a vessel's speed drops below 0.5 knots within the port area.

```sql
WITH port_boundary AS (
    -- Define the port area using coordinates
    SELECT ST_MakeEnvelope(
        -122.428, 37.765,  -- Min longitude, latitude
        -122.410, 37.785,  -- Max longitude, latitude
        4326
    ) as geom
),
docking_events AS (
    -- Identify docking and undocking events
    SELECT 
        DATE_TRUNC('day', m1.timestamp) as event_date,
        m1.mmsi,
        m1.timestamp as docking_time,
        LEAD(m1.timestamp) OVER (
            PARTITION BY m1.mmsi 
            ORDER BY m1.timestamp
        ) as undocking_time
    FROM ais_messages m1
    CROSS JOIN port_boundary pb
    WHERE 
        m1.timestamp >= CURRENT_DATE - INTERVAL '1 day'
        AND m1.timestamp < CURRENT_DATE
        AND ST_Contains(pb.geom, m1.geo_point)
        AND m1.speed <= 0.5
)
SELECT 
    event_date,
    COUNT(DISTINCT mmsi) as unique_vessels,
    COUNT(*) as total_docking_events,
    ROUND(AVG(
        EXTRACT(EPOCH FROM (undocking_time - docking_time))/3600
    )::numeric, 2) as avg_docking_hours,
    MIN(EXTRACT(EPOCH FROM (undocking_time - docking_time))/3600) as min_docking_hours,
    MAX(EXTRACT(EPOCH FROM (undocking_time - docking_time))/3600) as max_docking_hours
FROM docking_events
WHERE undocking_time IS NOT NULL
GROUP BY event_date
ORDER BY event_date;
```
## Query Explanation

The query analyzes docking events for ships within a defined port boundary over the past day. It identifies docking and undocking times and calculates statistics for docking durations. The process works in three main steps:

## 1. `port_boundary`
- Defines the port area as a rectangular boundary using geographic coordinates (`ST_MakeEnvelope`).

## 2. `docking_events`
- Identifies docking and undocking events for ships (`ais_messages`) within the port boundary:
  - Filters positions from the past day (`CURRENT_DATE - INTERVAL '1 day'`).
  - Considers only ships with low speed (≤ 0.5), indicating docking.
  - Determines the undocking time for each docking event using the `LEAD()` function.

## 3. Final Query
- Calculates daily statistics for docking events:
  - **Unique vessels**: Count of distinct ships (`COUNT(DISTINCT mmsi)`).
  - **Total docking events**: Total number of docking records.
  - **Docking duration statistics**: Average, minimum, and maximum docking durations (in hours).
- Groups results by event date and orders them chronologically.

## Summary
The query provides insights into docking activity within a port, including the number of vessels, total events, and docking duration statistics for the past day.


## 2. Top 10 Busiest Docking Areas

This query identifies the most active docking areas based on vessel traffic over a week. We use spatial clustering to group nearby docking events and rank the areas by activity.

```sql
WITH docking_clusters AS (
    -- Group nearby docking positions into clusters
    SELECT 
        ST_SnapToGrid(geo_point, 0.001) as area_centroid,
        COUNT(DISTINCT mmsi) as unique_vessels,
        COUNT(*) as total_messages,
        AVG(speed) as avg_speed
    FROM ais_messages
    WHERE 
        timestamp >= NOW() - INTERVAL '1 week'
        AND speed <= 0.5
    GROUP BY ST_SnapToGrid(geo_point, 0.001)
    HAVING COUNT(DISTINCT mmsi) >= 3  -- Minimum vessels for valid docking area
),
ranked_areas AS (
    -- Rank areas by activity
    SELECT 
        ST_X(ST_Centroid(area_centroid)) as longitude,
        ST_Y(ST_Centroid(area_centroid)) as latitude,
        unique_vessels,
        total_messages,
        avg_speed,
        ROW_NUMBER() OVER (ORDER BY unique_vessels DESC, total_messages DESC) as rank
    FROM docking_clusters
)
SELECT 
    rank,
    ROUND(longitude::numeric, 4) as longitude,
    ROUND(latitude::numeric, 4) as latitude,
    unique_vessels,
    total_messages,
    ROUND(avg_speed::numeric, 2) as avg_speed
FROM ranked_areas
WHERE rank <= 10
ORDER BY rank;
```
## Query Explanation

The query identifies and ranks high-activity docking areas based on ship clustering over the past week. It works in three steps:

## 1. `docking_clusters`
- Groups nearby docking positions into clusters using `ST_SnapToGrid()` with a grid size of 0.001 degrees.
- Filters positions with:
  - Speed ≤ 0.5 (indicating potential docking).
  - At least 3 distinct vessels in the cluster (`COUNT(DISTINCT mmsi)`).
- Calculates:
  - Centroid of the cluster (`area_centroid`).
  - Number of unique vessels, total messages, and average speed.

## 2. `ranked_areas`
- Ranks docking clusters based on:
  - Number of unique vessels (descending).
  - Total messages (descending as a tie-breaker).
- Extracts the longitude and latitude of each cluster's centroid.

## 3. Final Query
- Selects the top 10 ranked docking areas.
- Outputs:
  - Rank, centroid coordinates (longitude, latitude), number of unique vessels, total messages, and average speed.
- Orders results by rank.

## Summary
The query identifies and ranks the top 10 most active docking areas from the past week, providing key statistics such as vessel count, message activity, and average speed.


## 3. Historical Docking Trends Analysis

This query provides monthly summaries of docking activities, including trend analysis and year-over-year comparisons.

```sql
WITH monthly_statistics AS (
    -- Calculate monthly docking statistics
    SELECT 
        DATE_TRUNC('month', timestamp) as month,
        COUNT(DISTINCT mmsi) as unique_vessels,
        COUNT(*) FILTER (WHERE speed <= 0.5) as docking_events,
        AVG(speed) as avg_speed,
        COUNT(DISTINCT 
            CASE WHEN speed <= 0.5 
            THEN DATE_TRUNC('day', timestamp) 
            END
        ) as active_days
    FROM ais_messages
    WHERE timestamp >= NOW() - INTERVAL '12 months'
    GROUP BY DATE_TRUNC('month', timestamp)
),
trend_analysis AS (
    -- Add trend analysis metrics
    SELECT 
        month,
        unique_vessels,
        docking_events,
        ROUND(avg_speed::numeric, 2) as avg_speed,
        active_days,
        ROUND(
            (docking_events::float / 
            LAG(docking_events) OVER (ORDER BY month) - 1) * 100,
            2
        ) as month_over_month_change,
        ROUND(
            (docking_events::float / 
            LAG(docking_events, 12) OVER (ORDER BY month) - 1) * 100,
            2
        ) as year_over_year_change
    FROM monthly_statistics
)
SELECT 
    month,
    unique_vessels,
    docking_events,
    avg_speed,
    active_days,
    month_over_month_change,
    year_over_year_change
FROM trend_analysis
ORDER BY month DESC;
```
## Query Explanation

The query calculates monthly docking statistics for the past year and analyzes trends, including month-over-month (MoM) and year-over-year (YoY) changes. It works in two main steps:

## 1. `monthly_statistics`
- Aggregates docking data per month:
  - **Unique vessels**: Count of distinct ships (`COUNT(DISTINCT mmsi)`).
  - **Docking events**: Count of messages where speed ≤ 0.5.
  - **Average speed**: Average speed of all messages.
  - **Active days**: Count of unique days with docking events.

- Filters data to include only messages from the past 12 months.

## 2. `trend_analysis`
- Adds trend metrics for docking events:
  - **Month-over-month change**: Percentage change compared to the previous month.
  - **Year-over-year change**: Percentage change compared to the same month last year.
  - Both metrics use `LAG()` to compare values.

## 3. Final Query
- Selects and displays:
  - Month, unique vessels, docking events, average speed, active days, MoM change, and YoY change.
- Results are sorted in descending order by month for the latest trends.

## Summary
The query provides a detailed monthly summary of docking activity and highlights trends, helping to understand seasonal or operational patterns in ship activity.


## Query Optimization Strategies

### 1. Time-Based Optimization

For optimal low-latency performance, we implement several time-based optimizations:

```sql
-- Create time-based partitions
CREATE TABLE ais_messages_partition OF ais_messages
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Create time-based indexes
CREATE INDEX idx_ais_messages_timestamp 
ON ais_messages (timestamp DESC);
```

### 2. Spatial Optimization

We optimize spatial queries using PostGIS indexes and efficient geometric operations:

```sql
-- Create spatial index
CREATE INDEX idx_ais_messages_geo 
ON ais_messages USING GIST (geo_point);

-- Create combined spatial-temporal index
CREATE INDEX idx_ais_messages_geo_time 
ON ais_messages USING GIST (geo_point, timestamp);
```

### 3. Materialized Views

For frequently accessed reports, we implement materialized views:

```sql
-- Create materialized view for daily statistics
CREATE MATERIALIZED VIEW daily_port_stats AS
SELECT 
    DATE_TRUNC('day', timestamp) as day,
    COUNT(DISTINCT mmsi) as vessels,
    COUNT(*) FILTER (WHERE speed <= 0.5) as docking_events,
    AVG(speed) as avg_speed
FROM ais_messages
GROUP BY DATE_TRUNC('day', timestamp)
WITH NO DATA;

-- Create refresh function
CREATE OR REPLACE FUNCTION refresh_daily_stats()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_port_stats;
END;
$$ LANGUAGE plpgsql;
```

### Performance Optimization Recommendations

To maintain low latency for these reporting queries:

1. Data Management:
   - Implement table partitioning by month
   - Use appropriate partial indexes
   - Maintain regular statistics updates
   - Archive historical data efficiently

2. Query Optimization:
   - Utilize parallel query execution
   - Implement result caching
   - Use appropriate join strategies
   - Leverage window functions for efficiency

3. Infrastructure Settings:
   - Adjust work_mem for complex sorts
   - Configure effective_cache_size appropriately
   - Set maintenance_work_mem for index operations
   - Enable parallel query execution

4. Monitoring:
   - Track query execution times
   - Monitor cache hit ratios
   - Analyze query plans regularly
   - Implement query performance logging

By implementing these optimization strategies, the reporting queries can maintain good performance even with large datasets and concurrent users.
