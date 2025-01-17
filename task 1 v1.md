# Task 1: Schema Design
Design a schema to store AIS messages with the following requirements:
1.  Support real-time and historical queries for docking/undocking events.
2.  Optimize storage usage for historical data.
3.  Indexes must support fast geospatial and time-based queries.
4.  Support partitioning for high throughput and scalability.
    

# AIS Database Schema Design and Rationale

## Schema Overview

The database schema is designed to 
1. handle Automatic Identification System (AIS) messages efficiently.
2. supporting both real-time operations and historical analysis. 

### Table Structure

We've split the data into two main tables:
![enter image description here](https://gcdnb.pbrd.co/images/OtkXLP8gXw1F.png?o=1)


```sql
CREATE TABLE ships (
    mmsi BIGINT PRIMARY KEY,
    imo BIGINT,
    callsign VARCHAR(10),
    shipname VARCHAR(100),
    shiptype ship_type,
    destination VARCHAR(100),
    length_meters NUMERIC(6,2),
    width_meters NUMERIC(6,2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ais_messages (
    id BIGSERIAL,
    mmsi BIGINT NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    lat NUMERIC(9,6),
    lon NUMERIC(9,6),
    geo_point GEOMETRY(Point, 4326),
    course NUMERIC(5,2),
    heading NUMERIC(5,2),
    speed NUMERIC(5,2),
    status ship_status,
    draught NUMERIC(4,2),
    maneuver SMALLINT,
    distance_to_bow NUMERIC(6,2),
    distance_to_port NUMERIC(6,2),
    distance_to_starboard NUMERIC(6,2),
    distance_to_stern NUMERIC(6,2),
    site VARCHAR(20),
    class_type VARCHAR(10),
    CONSTRAINT fk_ship FOREIGN KEY (mmsi) REFERENCES ships(mmsi)
) PARTITION BY RANGE (timestamp);
```

### Design Decisions Explained

1. **Data Normalization Strategy**
   The schema separates static and dynamic data into two tables: 'ships' and 'ais_messages'. This decision was made because:
   - Static ship information changes rarely but is frequently accessed
   - It reduces data redundancy since static data isn't repeated in every message
   - It allows for efficient updates to ship information without affecting message history

2. **Partitioning Implementation**
   We chose RANGE partitioning by timestamp with monthly partitions because:
   - Monthly partitions provide a good balance between partition size and management overhead
   - Allows for efficient pruning of old data when querying recent information
   - Supports the 5-year retention requirement while maintaining query performance
   - Enables easier archival of historical data by partition

3. **Data Type Selections**
   - NUMERIC(9,6) for coordinates ensures sufficient precision for maritime positioning
   - TIMESTAMP WITH TIME ZONE prevents timezone-related issues
   - Custom ENUMs for status and ship type improve data integrity and reduce storage
   - GEOMETRY(Point, 4326) for spatial data enables efficient geospatial queries
   - BIGINT for MMSI and IMO numbers ensures no overflow issues

4. **Indexing Strategy**
   ```sql
   CREATE INDEX idx_ais_messages_mmsi_timestamp ON ais_messages (mmsi, timestamp DESC);
   CREATE INDEX idx_ais_messages_timestamp ON ais_messages (timestamp DESC);
   CREATE INDEX idx_ais_messages_geo ON ais_messages USING GIST (geo_point);
   CREATE INDEX idx_ais_messages_speed ON ais_messages (speed) WHERE speed <= 0.5;
   ```

## Core Indexes

### 1. Timestamp-Only Index
```sql
CREATE INDEX idx_ais_messages_timestamp ON ais_messages (timestamp DESC);
```

**Purpose:**
- Optimizes time-based queries across all vessels
- Essential for historical data analysis

### 2. Combined MMSI-Timestamp Index
```sql
CREATE INDEX idx_ais_messages_mmsi_timestamp ON ais_messages (mmsi, timestamp DESC);
```

**Purpose:**
- Primary index for tracking individual vessel movements
- Optimizes queries that filter by both ship and time period

### 3. Geospatial Index
```sql
CREATE INDEX idx_ais_messages_geo ON ais_messages USING GIST (geo_point);
```

**Purpose:**
- Enables efficient spatial queries
- Essential for proximity and area-based searches
- Supports docking area detection
- Dramatically improves spatial query performance

### 4. Speed-Based Partial Index
```sql
CREATE INDEX idx_ais_messages_speed ON ais_messages (speed) 
WHERE speed <= 0.5;
```

**Purpose:**
- Optimizes queries for slow/stopped vessels
- Reduces index size by only including relevant records
- Supports idle ship detection
- Very efficient for finding slow/stopped vessels


This design reflects:
- Efficient partitioning strategy
- Optimized index design
- Separation of static and dynamic data

