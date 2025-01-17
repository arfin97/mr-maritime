## Task 4: Data Compression and Archival

Design a strategy to compress and archive historical AIS data (older than 1 year):

1.  Compress historical data to minimize storage usage.
2.  Archive data into cost-effective cold storage while ensuring it can still be queried.
    

Deliverables:

1.  Compression and archival strategy (e.g., extensions, commands).
2.  Steps for querying archived data.

# AIS Data Compression and Archival Strategy

## Introduction

In managing the AIS data system that handles 700-900 million messages daily with a 5-year retention requirement, we need an efficient strategy to compress and archive historical data while maintaining its queryability. This document outlines our comprehensive approach to data compression and archival for data older than one year.

## 1. Data Compression Strategy

### Native PostgreSQL Table Compression

First, we'll implement table-level compression using PostgreSQL's built-in features. This approach maintains good query performance while reducing storage requirements.

```sql
-- Enable compression-related extensions
CREATE EXTENSION IF NOT EXISTS pg_compression;

-- Create a compressed partition template for historical data
CREATE TABLE ais_messages_historical (
    LIKE ais_messages INCLUDING ALL,
    compression pg_lz4
) PARTITION BY RANGE (timestamp);

-- Create function to automatically create compressed partitions
CREATE OR REPLACE FUNCTION create_compressed_partition(
    start_date date,
    end_date date
) RETURNS void AS $$
BEGIN
    EXECUTE format(
        'CREATE TABLE ais_messages_%s_%s 
         PARTITION OF ais_messages_historical
         FOR VALUES FROM (%L) TO (%L)
         WITH (compression = pg_lz4)',
        to_char(start_date, 'YYYY'),
        to_char(start_date, 'MM'),
        start_date,
        end_date
    );
END;
$$ LANGUAGE plpgsql;

-- Function to compress specific columns
CREATE OR REPLACE FUNCTION compress_numeric_columns() RETURNS void AS $$
BEGIN
    -- Round numeric values to reduce storage while maintaining acceptable precision
    UPDATE ais_messages_historical SET
        lat = round(lat::numeric, 6),
        lon = round(lon::numeric, 6),
        course = round(course::numeric, 1),
        speed = round(speed::numeric, 1),
        heading = round(heading::numeric, 1);
END;
$$ LANGUAGE plpgsql;
```

### Data Summarization Strategy

For historical data, we'll create summary tables that maintain essential information while reducing storage requirements.

```sql
-- Create summary table for historical statistics
CREATE TABLE ais_messages_summary (
    summary_id bigserial PRIMARY KEY,
    day date NOT NULL,
    mmsi bigint NOT NULL,
    message_count integer,
    avg_speed numeric(5,2),
    avg_course numeric(5,2),
    min_lat numeric(9,6),
    max_lat numeric(9,6),
    min_lon numeric(9,6),
    max_lon numeric(9,6),
    path_geometry geometry(LINESTRING, 4326),
    compressed_payload bytea
);

-- Create function to generate summaries
CREATE OR REPLACE FUNCTION summarize_daily_data(
    target_date date
) RETURNS void AS $$
BEGIN
    INSERT INTO ais_messages_summary (
        day, mmsi, message_count, avg_speed, avg_course,
        min_lat, max_lat, min_lon, max_lon, path_geometry
    )
    SELECT 
        DATE_TRUNC('day', timestamp)::date,
        mmsi,
        COUNT(*),
        AVG(speed),
        AVG(course),
        MIN(lat),
        MAX(lat),
        MIN(lon),
        MAX(lon),
        ST_MakeLine(geo_point ORDER BY timestamp) as path_geometry
    FROM ais_messages
    WHERE timestamp::date = target_date
    GROUP BY DATE_TRUNC('day', timestamp)::date, mmsi;
END;
$$ LANGUAGE plpgsql;
```

## 2. Archival Strategy

### Implementing a Tiered Storage System

We'll create a tiered storage system that moves data through different stages based on age and access patterns.

```sql
-- Create foreign table for archived data
CREATE SERVER archive_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'archive-db.example.com', port '5432', dbname 'ais_archive');

CREATE FOREIGN TABLE ais_messages_archive (
    LIKE ais_messages
)
SERVER archive_server;

-- Create function to move data to archive
CREATE OR REPLACE FUNCTION archive_old_data(
    older_than interval
) RETURNS void AS $$
BEGIN
    -- Move data to archive server
    INSERT INTO ais_messages_archive
    SELECT * FROM ais_messages
    WHERE timestamp < (CURRENT_TIMESTAMP - older_than);
    
    -- Remove archived data from main table
    DELETE FROM ais_messages
    WHERE timestamp < (CURRENT_TIMESTAMP - older_than);
    
    -- Update statistics
    ANALYZE ais_messages;
    ANALYZE ais_messages_archive;
END;
$$ LANGUAGE plpgsql;
```

### Data Retention Policy Implementation

```sql
-- Create retention policy function
CREATE OR REPLACE FUNCTION implement_retention_policy() RETURNS void AS $$
BEGIN
    -- Move data older than 1 year to compressed storage
    EXECUTE create_compressed_partition(
        DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 year'),
        DATE_TRUNC('month', CURRENT_DATE - INTERVAL '11 months')
    );
    
    -- Archive data older than 2 years
    PERFORM archive_old_data(INTERVAL '2 years');
    
    -- Generate summaries for archived data
    PERFORM summarize_daily_data(CURRENT_DATE - INTERVAL '2 years');
END;
$$ LANGUAGE plpgsql;
```

## 3. Querying Archived Data

### Direct Query Access

To query archived data, use the following approach:

```sql
-- Create view combining current and archived data
CREATE OR REPLACE VIEW ais_messages_complete AS
    SELECT * FROM ais_messages
    UNION ALL
    SELECT * FROM ais_messages_archive;

-- Example query spanning current and archived data
SELECT 
    DATE_TRUNC('month', timestamp) as month,
    COUNT(*) as message_count,
    COUNT(DISTINCT mmsi) as unique_vessels
FROM ais_messages_complete
WHERE timestamp >= '2024-01-01'
GROUP BY DATE_TRUNC('month', timestamp)
ORDER BY month;
```

### Using Summary Tables for Historical Analysis

```sql
-- Query using summary tables for historical analysis
SELECT 
    day,
    COUNT(DISTINCT mmsi) as active_vessels,
    SUM(message_count) as total_messages,
    AVG(avg_speed) as fleet_average_speed
FROM ais_messages_summary
WHERE day BETWEEN '2023-01-01' AND '2023-12-31'
GROUP BY day
ORDER BY day;
```

## 4. Maintenance and Monitoring

### Regular Maintenance Tasks

```sql
-- Create maintenance schedule
CREATE OR REPLACE FUNCTION perform_maintenance() RETURNS void AS $$
BEGIN
    -- Compress old partitions
    PERFORM compress_numeric_columns();
    
    -- Update statistics
    ANALYZE ais_messages_historical;
    ANALYZE ais_messages_summary;
    
    -- Vacuum archived tables
    VACUUM ANALYZE ais_messages_archive;
END;
$$ LANGUAGE plpgsql;
```

### Monitoring Storage Usage

```sql
-- Create function to monitor storage
CREATE OR REPLACE FUNCTION check_storage_usage() RETURNS TABLE (
    table_name text,
    total_size text,
    index_size text,
    compression_ratio numeric
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        schemaname || '.' || tablename as table_name,
        pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
        pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) as index_size,
        CASE 
            WHEN pg_total_relation_size(schemaname || '.' || tablename) > 0 
            THEN round(pg_total_relation_size(schemaname || '.' || tablename)::numeric / 
                      pg_relation_size(schemaname || '.' || tablename)::numeric, 2)
        END as compression_ratio
    FROM pg_tables
    WHERE tablename LIKE 'ais_messages%'
    ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;
END;
$$ LANGUAGE plpgsql;
```

## Implementation Steps

1. Initialize compression infrastructure
2. Create necessary partitions for historical data
3. Set up archival server and foreign tables
4. Implement maintenance schedules
5. Monitor storage usage and query performance
6. Regular testing of archived data accessibility

By following this strategy, we can effectively manage the large volume of AIS data while maintaining accessibility and reducing storage costs.
