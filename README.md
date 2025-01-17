# AIS Message Analysis System Design

## Scenario
Your organization is building a system to analyze AIS messages for maritime event detection (e.g., docking, undocking, collisions) and generate instant reports. The system must:
- Handle **700-900 million AIS messages per day** with minimal storage overhead.
- Enable **real-time and historical queries** for event detection and reporting.
- Optimize for **low-latency queries** on current data while maintaining historical data efficiently.
- Support **geospatial queries** with high performance.

You will design a **PostgreSQL-based solution** to store and query AIS messages while meeting these requirements.

---

## Data Overview
### Dynamic Data
- **mmsi**: 123456789  
- **lat**: 37.7749  
- **lon**: -122.4194  
- **course**: 85.6  
- **heading**: 90  
- **speed**: 12.3  
- **maneuver**: 0  
- **repeat**: 1  
- **seconds**: 1678987395  
- **status**: 0  
- **turn**: 0.0  

### Static Data
- **imo**: 8814275  
- **callsign**: ABCD  
- **shipname**: Ocean Explorer  
- **shiptype**: Cargo  
- **destination**: Port of Oakland  
- **eta**: 2025-01-10T08:00  
- **draught**: 7.5  

### Geospatial Data
- **geo_point**: SRID=4326;POINT(-122.4194 37.7749)  
- **distancetobow**: 25.0  
- **distancetoport**: 12.0  
- **distancetostarboard**: 12.0  
- **distancetostern**: 50.0  

### Metadata
- **messagetagblocktime**: 1678987395000  
- **timereceived**: 2025-01-10T07:30:00Z  
- **site**: US-West-001  
- **classtype**: Class A  

---

## Throughput & Retention
- **Daily ingestion**: 700-900 million messages (~8,000-10,000 messages/second).  
- **Retention**: Historical data for 5 years (~1.5 trillion rows).

---

## Tasks

### **Task 1: Schema Design**
Design a schema to store AIS messages with the following requirements:
- Support real-time and historical queries for docking/undocking events.
- Optimize storage usage for historical data.
- Indexes must support fast geospatial and time-based queries.
- Support partitioning for high throughput and scalability.

#### Deliverables:
- Schema design (tables, constraints, indexes).
- Explanation of design decisions (e.g., normalization, partitioning strategy).

---

### **Task 2: Event Detection Queries**
Write advanced PostgreSQL queries to detect the following events:
- **Docking Event**: A ship enters a docking area (defined as a polygon) and stays for at least 30 minutes.
- **Undocking Event**: A ship leaves a docking area and moves at least 2 km away within 15 minutes.
- **Idle Ship Detection**: A ship with a speed of ≤ 0.5 knots for more than 1 hour.

#### Deliverables:
- SQL queries for each event, optimized for large datasets.
- Execution plans for each query and suggestions to improve performance.

---

### **Task 3: Reporting and Analytics**
Design queries to generate the following reports:
- Daily docking/undocking report for a specific port (count, average docking duration, etc.).
- Top 10 busiest docking areas for a given week.
- Historical analysis of docking trends (e.g., monthly summaries).

#### Deliverables:
- SQL queries for generating reports.
- Explanation of how to optimize for low latency.

---

### **Task 4: Data Compression and Archival**
Design a strategy to compress and archive historical AIS data (older than 1 year):
- Compress historical data to minimize storage usage.
- Archive data into cost-effective cold storage while ensuring it can still be queried.

#### Deliverables:
- Compression and archival strategy (e.g., extensions, commands).
- Steps for querying archived data.

---

### **Task 5: System Optimization**
Optimize the PostgreSQL instance for the AIS workload:
- Suggest configurations for `work_mem`, `maintenance_work_mem`, `effective_cache_size`, and `max_parallel_workers`.
- Recommend query and index tuning strategies for **1 billion rows/day throughput**.

#### Deliverables:
- Configuration recommendations with justifications.
- Steps to monitor and troubleshoot slow queries.

---

### **Task 6 (Bonus): Collision Detection**
Write a SQL query to detect potential ship collisions where:
- Two ships come within **100 meters** of each other.
- Their courses intersect within **5 minutes**.

#### Deliverables:
- SQL query for collision detection.
