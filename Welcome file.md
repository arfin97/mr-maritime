# Task 1 Solution

# AIS Database Schema Design and Rationale

## Schema Overview

The database schema is designed to 
1. handle Automatic Identification System (AIS) messages efficiently.
2. supporting both real-time operations and historical analysis. 

### Table Structure

We've split the data into two main tables:



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
   - The storage savings are significant given the 700-900 million daily messages

2. **Partitioning Implementation**
   We chose RANGE partitioning by timestamp with monthly partitions because:
   - Monthly partitions provide a good balance between partition size and management overhead
   - It aligns with the requirement to handle 700-900 million messages per day
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
   These indexes were chosen to optimize the most common query patterns:
   - Combined (mmsi, timestamp) index for tracking specific vessels
   - Timestamp index for time-based queries
   - GiST index for spatial queries (docking detection)
   - Partial index on speed for efficient idle ship detection

5. **Performance Optimizations**
   - Materialized views for daily statistics reduce computation overhead
   - Automatic partition creation function ensures scalability
   - Trigger-based timestamp updates maintain data currency
   - Foreign key constraints ensure data integrity without excessive overhead

### Handling the Scale Requirements

The schema is designed to handle the specified scale:
- Daily ingestion of 700-900 million messages (~8,000-10,000 messages/second)
- 5-year retention period (~1.5 trillion rows)
- Real-time query capabilities
- Historical data access

This is achieved through:
- Efficient partitioning strategy
- Optimized index design
- Separation of static and dynamic data
- Support for compression at the partition level
- Built-in management functions for maintenance

### Monitoring and Maintenance

The schema includes features for ongoing maintenance:
- Partition management function for creating future partitions
- Timestamp tracking for data currency
- Support for efficient archival of old partitions
- Ability to rebuild indexes per partition
- Statistics gathering capabilities for query optimization

## Conclusion

This schema design balances the competing requirements of high-throughput write performance, efficient query capabilities, and manageable maintenance overhead. It provides a solid foundation for building the maritime event detection system while maintaining flexibility for future optimizations and changes in requirements.
