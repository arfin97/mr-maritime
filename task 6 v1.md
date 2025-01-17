
# Task 6 (Bonus): Collision Detection

Write a SQL query to detect potential ship collisions where:

1.  Two ships come within 100 meters of each other.
2.  Their courses intersect within 5 minutes.

# Ship Collision Detection System

## Understanding the Problem

To detect potential collisions, we need to consider two main factors:
1. Spatial proximity between vessels (within 100 meters)
2. Course intersection within a short time frame (5 minutes)

The solution involves creating projected trajectories based on current position, speed, and course, then checking if these trajectories intersect while the ships are in close proximity.

## The Solution

```sql
WITH vessel_trajectories AS (
    -- First, we calculate the projected position of each vessel after 5 minutes
    -- based on their current course and speed
    SELECT 
        m1.mmsi,
        m1.timestamp,
        m1.geo_point as current_position,
        m1.speed as speed_knots,
        m1.course,
        -- Calculate projected position after 5 minutes
        ST_Project(
            m1.geo_point::geography,
            (m1.speed * 0.514444 * 300),  -- Convert knots to m/s and multiply by 300 seconds
            radians(m1.course)
        )::geometry as projected_position
    FROM ais_messages m1
    WHERE 
        m1.timestamp >= NOW() - INTERVAL '5 minutes'
        AND m1.speed > 0  -- Only consider moving vessels
        AND m1.course IS NOT NULL
),
potential_collisions AS (
    -- Find pairs of vessels that are close to each other
    SELECT 
        v1.mmsi as vessel1_mmsi,
        v2.mmsi as vessel2_mmsi,
        v1.timestamp,
        v1.speed_knots as vessel1_speed,
        v2.speed_knots as vessel2_speed,
        v1.course as vessel1_course,
        v2.course as vessel2_course,
        -- Calculate current distance between vessels
        ST_Distance(
            v1.current_position::geography, 
            v2.current_position::geography
        ) as current_distance_meters,
        -- Create trajectory lines
        ST_MakeLine(
            v1.current_position, 
            v1.projected_position
        ) as trajectory1,
        ST_MakeLine(
            v2.current_position, 
            v2.projected_position
        ) as trajectory2,
        -- Check if trajectories intersect
        ST_Intersects(
            ST_MakeLine(v1.current_position, v1.projected_position),
            ST_MakeLine(v2.current_position, v2.projected_position)
        ) as trajectories_intersect,
        -- Calculate closest point of approach (CPA)
        ST_ClosestPoint(
            ST_MakeLine(v1.current_position, v1.projected_position),
            ST_MakeLine(v2.current_position, v2.projected_position)
        ) as closest_approach_point
    FROM vessel_trajectories v1
    JOIN vessel_trajectories v2 ON 
        v1.mmsi < v2.mmsi  -- Avoid duplicate pairs
        AND v1.timestamp = v2.timestamp  -- Same time window
        AND ST_DWithin(  -- Initial proximity check
            v1.current_position::geography,
            v2.current_position::geography,
            1000  -- Look at vessels within 1km for trajectory analysis
        )
)
SELECT 
    pc.vessel1_mmsi,
    s1.shipname as vessel1_name,
    pc.vessel2_mmsi,
    s2.shipname as vessel2_name,
    pc.timestamp as detection_time,
    ROUND(pc.current_distance_meters::numeric, 2) as current_distance_meters,
    pc.vessel1_speed,
    pc.vessel2_speed,
    pc.vessel1_course,
    pc.vessel2_course,
    CASE 
        WHEN pc.trajectories_intersect THEN 'YES'
        ELSE 'NO'
    END as collision_risk,
    -- Calculate estimated time to collision point
    ROUND(
        (ST_Distance(
            pc.current_position::geography,
            pc.closest_approach_point::geography
        ) / (GREATEST(pc.vessel1_speed, pc.vessel2_speed) * 0.514444))::numeric,
        2
    ) as estimated_seconds_to_closest_approach
FROM potential_collisions pc
JOIN ships s1 ON pc.vessel1_mmsi = s1.mmsi
JOIN ships s2 ON pc.vessel2_mmsi = s2.mmsi
WHERE 
    pc.current_distance_meters <= 100  -- Final proximity threshold
    AND pc.trajectories_intersect = true
ORDER BY pc.current_distance_meters ASC;
```

## Query Explanation

The collision detection query works in three main stages:

### 1. Vessel Trajectories (First CTE)
- Calculates each vessel's projected position after 5 minutes
- Uses the vessel's current speed and course
- Converts speed from knots to meters per second for calculations
- Projects the position using PostGIS ST_Project function

### 2. Potential Collisions (Second CTE)
- Pairs vessels that are within initial proximity (1km)
- Calculates current distance between each pair
- Creates trajectory lines for both vessels
- Checks if trajectories intersect
- Calculates closest point of approach

### 3. Final Results
- Filters for vessels within 100 meters
- Confirms trajectory intersection
- Joins with ships table to get vessel names
- Calculates estimated time to closest approach
- Orders results by current distance

## Performance Optimization

To optimize this query for large datasets:

```sql
-- Create supporting indexes
CREATE INDEX idx_ais_messages_collision 
ON ais_messages (timestamp, speed, course)
WHERE speed > 0 AND course IS NOT NULL;

CREATE INDEX idx_ais_messages_geo_time 
ON ais_messages USING GIST (geo_point, timestamp);

-- Create a partial index for recent data
CREATE INDEX idx_ais_messages_recent 
ON ais_messages (mmsi, timestamp) 
WHERE timestamp >= NOW() - INTERVAL '10 minutes';
```

## Maintenance Considerations

1. Regular cleanup of old data:
```sql
-- Remove old position data
DELETE FROM ais_messages 
WHERE timestamp < NOW() - INTERVAL '1 hour';

-- Keep statistics updated
ANALYZE ais_messages;
```

2. Monitor performance:
```sql
-- Check execution time and plan
EXPLAIN ANALYZE [collision detection query];
```

## Usage Notes

1. The query assumes:
   - Positions are in WGS84 (SRID 4326)
   - Speed is in knots
   - Course is in degrees (0-360)

2. Limitations:
   - Assumes linear trajectories
   - Does not account for acceleration/deceleration
   - Does not consider vessel dimensions
   - Weather and sea conditions not factored in

## Future Improvements

1. Add vessel size consideration:
   - Use vessel dimensions from ships table
   - Adjust collision detection radius based on vessel size

2. Enhanced trajectory calculation:
   - Consider rate of turn
   - Account for acceleration/deceleration
   - Include historical movement patterns

3. Risk assessment:
   - Add risk scoring based on vessel types
   - Consider weather conditions
   - Factor in vessel maneuverability
