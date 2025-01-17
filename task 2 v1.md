
# Task 2: Event Detection Queries
Write advanced PostgreSQL queries to detect the following events:
1.  **Docking Event:** A ship enters a docking area (defined as a polygon) and stays for at least 30 minutes.
2.  **Undocking Event:** A ship leaves a docking area and moves at least 2 km away within 15 minutes.
3.  **Idle Ship Detection:** A ship with a speed of ≤ 0.5 knots for more than 1 hour.

**Deliverables:**
1.  SQL queries for each event, optimized for large datasets.
2.  Execution plans for each query and suggestions to improve performance.

## Event Detection Queries and Optimization Analysis
I introduced a new table called docking_areas that wasn't in the original entity diagram.

Looking at the original requirements from Task 1, we only had two tables:

ships - For static ship information
ais_messages - For dynamic position reports

 I added:
```sql
CREATE TABLE docking_areas (
    area_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    geometry GEOMETRY(POLYGON, 4326)
);
```

# 1. Docking Event Detection

This query identifies vessels that have entered a docking area and remained stationary for at least 30 minutes.

```sql
WITH ship_positions AS (
    SELECT 
        m.mmsi,
        m.timestamp,
        m.geo_point,
        m.speed,
        da.area_id,
        LEAD(m.timestamp) OVER (
            PARTITION BY m.mmsi, da.area_id 
            ORDER BY m.timestamp
        ) as next_timestamp
    FROM ais_messages m
    CROSS JOIN docking_areas da
    WHERE 
        m.timestamp >= NOW() - INTERVAL '2 hours'
        AND ST_DWithin(m.geo_point::geography, da.geometry::geography, 100)
),
docking_candidates AS (
    SELECT 
        mmsi,
        area_id,
        timestamp as dock_start,
        next_timestamp as dock_end
    FROM ship_positions
    WHERE 
        speed <= 1.0
        AND next_timestamp IS NOT NULL
        AND EXTRACT(EPOCH FROM (next_timestamp - timestamp)) >= 1800
)
SELECT 
    dc.mmsi,
    s.shipname,
    dc.area_id,
    da.name as dock_name,
    dc.dock_start,
    dc.dock_end
FROM docking_candidates dc
JOIN ships s ON dc.mmsi = s.mmsi
JOIN docking_areas da ON dc.area_id = da.area_id;
```

## Query Explanation

The query identifies ships that may have docked near known docking areas within the last 2 hours. It works in three steps:

## 1. `ship_positions`
- Filters ship positions (`ais_messages`) near docking areas (`docking_areas`) within 100 meters.
- Retrieves each ship's next timestamp in the same area, ordered by time.

## 2. `docking_candidates`
- Identifies stationary ships with:
  - Speed ≤ 1.0 (indicating they might be docking).
  - A time difference of at least 30 minutes (≥ 1800 seconds) between consecutive timestamps.

## 3. Final Query
- Combines docking candidates with ship and docking area details.
- Displays:
  - Ship ID and name.
  - Docking area ID and name.
  - Docking start and end times.

## Execution Plan Analysis
- Uses GiST spatial index for efficient docking area proximity checks
- Utilizes index-only scans for ship lookup

## Performance Optimization Suggestions
- Create partial index on speed for slow-moving vessels
- Implement partition pruning by timestamp
- Consider materialized views for frequently accessed docking areas

# 2. Undocking Event Detection

This query identifies vessels that leave a docking area and move at least 2km within 15 minutes.

```sql
WITH undocking_candidates AS (
    SELECT 
        m1.mmsi,
        m1.timestamp as undock_start,
        m2.timestamp as movement_time,
        ST_Distance(
            m1.geo_point::geography,
            m2.geo_point::geography
        ) as distance_moved
    FROM ais_messages m1
    JOIN ais_messages m2 ON 
        m1.mmsi = m2.mmsi
        AND m2.timestamp > m1.timestamp
        AND m2.timestamp <= m1.timestamp + INTERVAL '15 minutes'
    JOIN docking_areas da ON ST_Contains(da.geometry, m1.geo_point)
    WHERE 
        m1.timestamp >= NOW() - INTERVAL '2 hours'
        AND m1.speed <= 1.0
        AND m2.speed > 1.0
)
SELECT DISTINCT ON (mmsi)
    uc.mmsi,
    s.shipname,
    uc.undock_start,
    uc.movement_time,
    uc.distance_moved
FROM undocking_candidates uc
JOIN ships s ON uc.mmsi = s.mmsi
WHERE distance_moved >= 2000
ORDER BY mmsi, undock_start DESC;
```
## Query Explanation

The query identifies ships that have recently undocked from a docking area. It works in two main steps:

## 1. `undocking_candidates`
- Finds ship positions (`ais_messages`) within docking areas (`docking_areas`) where:
  - The ship was stationary (speed ≤ 1.0) at the starting timestamp.
  - The ship started moving (speed > 1.0) within 15 minutes.
  - Calculates the distance moved between the two timestamps.

## 2. Final Query
- Filters results where the ship moved at least 2000 meters.
- Combines the undocking data with ship details from the `ships` table.
- Selects the most recent undocking event for each ship, displaying:
  - Ship ID and name.
  - Undocking start time, movement time, and distance moved.

## Summary
The query identifies recent undocking events, showing which ships left a docking area, when they started moving, and how far they traveled.


## Execution Plan Analysis
- Leverages spatial indexes for containment checks
- Uses time-based filtering to limit data scan
- Implements efficient joins with proper indexes
- Employs DISTINCT ON for deduplication

## Performance Optimization Suggestions
- Create composite index on (mmsi, timestamp, speed)
- Implement parallel query execution
- Consider partitioning by geographic region
- Cache frequently accessed docking area geometries

# 3. Idle Ship Detection

This query identifies vessels maintaining a speed of ≤ 0.5 knots for more than 1 hour.

```sql
WITH continuous_idle AS (
    SELECT 
        mmsi,
        MIN(timestamp) as idle_start,
        MAX(timestamp) as last_seen,
        COUNT(*) as position_count,
        AVG(speed) as avg_speed
    FROM (
        SELECT 
            mmsi,
            timestamp,
            speed,
            SUM(speed_change::int) OVER (
                PARTITION BY mmsi 
                ORDER BY timestamp
            ) as idle_group
        FROM (
            SELECT 
                mmsi,
                timestamp,
                speed,
                CASE 
                    WHEN LAG(speed) OVER (
                        PARTITION BY mmsi 
                        ORDER BY timestamp
                    ) > 0.5 AND speed <= 0.5 THEN 1
                    ELSE 0
                END as speed_change
            FROM ais_messages
            WHERE timestamp >= NOW() - INTERVAL '4 hours'
        ) speed_changes
    ) grouped_speeds
    GROUP BY mmsi, idle_group
    HAVING 
        MAX(timestamp) - MIN(timestamp) >= INTERVAL '1 hour'
        AND AVG(speed) <= 0.5
)
SELECT 
    ci.mmsi,
    s.shipname,
    s.shiptype,
    ci.idle_start,
    ci.last_seen,
    ci.avg_speed
FROM continuous_idle ci
JOIN ships s ON ci.mmsi = s.mmsi
ORDER BY idle_start DESC;
```
## Query Explanation

The query identifies ships that have been continuously idle (average speed ≤ 0.5) for at least 1 hour in the past 4 hours. It works in two main steps:

## 1. `continuous_idle`
- **Identify Idle Groups**:
  - Analyzes ship positions (`ais_messages`) to detect periods where a ship's speed transitions from > 0.5 to ≤ 0.5.
  - Groups consecutive idle periods using a cumulative sum (`SUM(speed_change::int)`).
- **Aggregate Idle Data**:
  - For each idle group, calculates:
    - Start time (`MIN(timestamp)`), end time (`MAX(timestamp)`), position count, and average speed.
  - Filters groups where:
    - The duration is at least 1 hour.
    - The average speed is ≤ 0.5.

## 2. Final Query
- Joins the idle data with ship details from the `ships` table.
- Displays:
  - Ship ID, name, type, idle start time, last seen time, and average speed during the idle period.
- Results are ordered by the most recent idle start time.

## Summary
The query identifies ships that have been stationary for extended periods, providing details about the ships and their idle durations.

## Execution Plan Analysis
- Uses window functions for efficient grouping
- Implements partial indexes for speed filtering
- Employs time-based partition pruning
- Utilizes index-only scans where possible

## Performance Optimization Suggestions
- Create partial index on low-speed records
- Implement parallel query processing
- Consider materialized views for long-term idle ships
- Use timestamp-based partitioning

## Supporting Indexes

```sql
-- Support indexes for optimizing the above queries
CREATE INDEX idx_ais_messages_speed_timestamp 
ON ais_messages (speed, timestamp) 
WHERE speed <= 1.0;

CREATE INDEX idx_ais_messages_mmsi_timestamp_speed 
ON ais_messages (mmsi, timestamp, speed);

CREATE INDEX idx_ais_messages_geo 
ON ais_messages USING GIST (geo_point);

CREATE INDEX idx_docking_areas_geo 
ON docking_areas USING GIST (geometry);
```

# General Performance Recommendations

1. **Data Partitioning Strategy:**
   - Implement time-based partitioning for ais_messages
   - Consider geographic partitioning for large ports
   - Use partition pruning in queries

2. **Index Optimization:**
   - Regular index maintenance
   - Monitor index usage statistics
   - Remove unused indexes
   - Consider partial indexes for specific conditions

3. **Query Optimization:**
   - Use parallelization for large datasets
   - Implement proper statistics gathering
   - Consider materialized views for common patterns
   - Use proper join techniques
