# Data Model Documentation

## Overview

This document describes the data models for each of the four NoSQL databases used in the Pasture Management System. Each database is chosen for its specific strengths and handles a particular aspect of the data pipeline.

---

## 1. Apache Cassandra - Time-Series Storage

### Purpose
High-throughput, scalable storage for sensor telemetry data with efficient time-range queries.

### Keyspace Configuration

```cql
CREATE KEYSPACE IF NOT EXISTS pasture_mgmt
WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 3
}
AND durable_writes = true;
```

### Table: sensor_data_by_field

**Purpose**: Store all sensor readings partitioned by field for efficient per-field time-range queries.

```cql
CREATE TABLE IF NOT EXISTS pasture_mgmt.sensor_data_by_field (
    field_id        TEXT,
    sensor_ts       TIMESTAMP,
    sensor_id       TEXT,
    metric_type     TEXT,
    metric_value    DOUBLE,
    quality_flag    INT,
    unit            TEXT,
    PRIMARY KEY ((field_id), sensor_ts, sensor_id, metric_type)
) WITH CLUSTERING ORDER BY (sensor_ts DESC, sensor_id ASC, metric_type ASC)
  AND compaction = {
      'class': 'TimeWindowCompactionStrategy',
      'compaction_window_size': 1,
      'compaction_window_unit': 'DAYS'
  }
  AND default_time_to_live = 31536000  -- 1 year TTL
  AND gc_grace_seconds = 864000;       -- 10 days
```

**Partition Key**: `field_id` - Groups all sensor data for a field together
**Clustering Columns**: 
- `sensor_ts DESC` - Most recent data first
- `sensor_id ASC` - Group by sensor
- `metric_type ASC` - Group by metric

**Example Records**:
```json
{
    "field_id": "field_001",
    "sensor_ts": "2024-10-15T14:30:00Z",
    "sensor_id": "sensor_soil_001",
    "metric_type": "soil_moisture",
    "metric_value": 23.5,
    "quality_flag": 1,
    "unit": "%"
}
```

```json
{
    "field_id": "field_001",
    "sensor_ts": "2024-10-15T14:30:00Z",
    "sensor_id": "sensor_weather_001",
    "metric_type": "temperature",
    "metric_value": 18.2,
    "quality_flag": 1,
    "unit": "celsius"
}
```

### Table: daily_aggregates_by_field

**Purpose**: Pre-computed daily aggregates for faster analytical queries.

```cql
CREATE TABLE IF NOT EXISTS pasture_mgmt.daily_aggregates_by_field (
    field_id        TEXT,
    date            DATE,
    metric_type     TEXT,
    min_value       DOUBLE,
    max_value       DOUBLE,
    avg_value       DOUBLE,
    sum_value       DOUBLE,
    count           INT,
    PRIMARY KEY ((field_id, metric_type), date)
) WITH CLUSTERING ORDER BY (date DESC)
  AND compaction = {
      'class': 'LeveledCompactionStrategy'
  }
  AND default_time_to_live = 94608000;  -- 3 years TTL
```

### Table: sensor_data_by_farm

**Purpose**: Cross-field queries at the farm level.

```cql
CREATE TABLE IF NOT EXISTS pasture_mgmt.sensor_data_by_farm (
    farm_id         TEXT,
    sensor_ts       TIMESTAMP,
    field_id        TEXT,
    sensor_id       TEXT,
    metric_type     TEXT,
    metric_value    DOUBLE,
    PRIMARY KEY ((farm_id), sensor_ts, field_id)
) WITH CLUSTERING ORDER BY (sensor_ts DESC, field_id ASC)
  AND default_time_to_live = 604800;  -- 7 days TTL (hot data only)
```

### Indexing Strategy

1. **Primary indexes** are built into partition and clustering keys
2. **Secondary index** on `metric_type` for filtering (use sparingly):
```cql
CREATE INDEX IF NOT EXISTS idx_metric_type 
ON pasture_mgmt.sensor_data_by_field (metric_type);
```

### Retention/Compaction Policy

| Table                    | TTL        | Compaction Strategy          |
|--------------------------|------------|------------------------------|
| sensor_data_by_field     | 1 year     | TimeWindowCompactionStrategy |
| daily_aggregates_by_field| 3 years    | LeveledCompactionStrategy    |
| sensor_data_by_farm      | 7 days     | SizeTieredCompactionStrategy |

---

## 2. MongoDB - Document Storage

### Purpose
Flexible document storage for farm/field metadata, farmer profiles, and event logging with geospatial query support.

### Database: pasture_mgmt

### Collection: farms

**Purpose**: Store farm-level information and ownership details.

```javascript
// Schema
{
    _id: ObjectId,
    farm_id: String,          // Unique identifier
    name: String,
    owner: {
        farmer_id: String,
        name: String,
        contact: {
            email: String,
            phone: String
        }
    },
    location: {
        address: String,
        region: String,
        country: String,
        center: {
            type: "Point",
            coordinates: [Number, Number]  // [longitude, latitude]
        }
    },
    total_area_ha: Number,
    established_date: Date,
    certifications: [String],
    metadata: Object
}

// Example Document
{
    "_id": ObjectId("6543210987654321fedcba98"),
    "farm_id": "farm_001",
    "name": "Green Valley Farm",
    "owner": {
        "farmer_id": "farmer_001",
        "name": "John Smith",
        "contact": {
            "email": "john@greenvalley.com",
            "phone": "+1-555-0123"
        }
    },
    "location": {
        "address": "123 Rural Road, Farmville",
        "region": "Midwest",
        "country": "USA",
        "center": {
            "type": "Point",
            "coordinates": [-89.4012, 43.0731]
        }
    },
    "total_area_ha": 250.5,
    "established_date": ISODate("2015-03-15"),
    "certifications": ["Organic", "Grass-Fed"],
    "metadata": {
        "last_inspection": ISODate("2024-06-01"),
        "risk_score": 0.23
    }
}
```

### Collection: fields

**Purpose**: Store field-level data with geospatial boundaries.

```javascript
// Schema
{
    _id: ObjectId,
    field_id: String,
    farm_id: String,
    name: String,
    boundary: {
        type: "Polygon",
        coordinates: [[[Number, Number]]]  // GeoJSON Polygon
    },
    area_ha: Number,
    soil_type: String,
    terrain: {
        elevation_m: Number,
        slope_pct: Number,
        aspect: String
    },
    establishment_date: Date,
    current_species: [{
        species_id: String,
        name: String,
        coverage_pct: Number
    }],
    sensors: [String],
    latest_metrics: {
        ndvi: Number,
        soil_moisture: Number,
        grass_height_cm: Number,
        temperature_c: Number,
        updated_at: Date
    },
    notes: [String]
}

// Example Document
{
    "_id": ObjectId("7654321098765432fedcba98"),
    "field_id": "field_001",
    "farm_id": "farm_001",
    "name": "North Pasture",
    "boundary": {
        "type": "Polygon",
        "coordinates": [[
            [-89.4050, 43.0780],
            [-89.4000, 43.0780],
            [-89.4000, 43.0730],
            [-89.4050, 43.0730],
            [-89.4050, 43.0780]
        ]]
    },
    "area_ha": 45.2,
    "soil_type": "loam",
    "terrain": {
        "elevation_m": 285,
        "slope_pct": 3.5,
        "aspect": "south"
    },
    "establishment_date": ISODate("2018-04-10"),
    "current_species": [
        {"species_id": "sp_001", "name": "Perennial Ryegrass", "coverage_pct": 60},
        {"species_id": "sp_002", "name": "White Clover", "coverage_pct": 25},
        {"species_id": "sp_003", "name": "Timothy", "coverage_pct": 15}
    ],
    "sensors": ["sensor_soil_001", "sensor_weather_001", "sensor_ndvi_001"],
    "latest_metrics": {
        "ndvi": 0.72,
        "soil_moisture": 28.5,
        "grass_height_cm": 12.3,
        "temperature_c": 18.5,
        "updated_at": ISODate("2024-10-15T14:00:00Z")
    },
    "notes": ["Reseeded in 2022", "Excellent drainage"]
}
```

### Collection: treatment_events

**Purpose**: Log all field treatments (fertilizer, lime, irrigation, etc.)

```javascript
// Schema
{
    _id: ObjectId,
    event_id: String,
    field_id: String,
    farm_id: String,
    treatment_type: String,
    treatment_details: {
        product: String,
        rate: Number,
        unit: String,
        method: String
    },
    applied_date: Date,
    applied_by: String,
    weather_conditions: Object,
    cost_usd: Number,
    notes: String,
    created_at: Date
}

// Example Document
{
    "_id": ObjectId("8765432109876543fedcba98"),
    "event_id": "event_001",
    "field_id": "field_001",
    "farm_id": "farm_001",
    "treatment_type": "fertilizer",
    "treatment_details": {
        "product": "Urea 46-0-0",
        "rate": 50,
        "unit": "kg/ha",
        "method": "broadcast"
    },
    "applied_date": ISODate("2024-09-01"),
    "applied_by": "farmer_001",
    "weather_conditions": {
        "temperature_c": 22,
        "wind_speed_kmh": 8,
        "precipitation_mm": 0
    },
    "cost_usd": 125.50,
    "notes": "Applied before predicted rain",
    "created_at": ISODate("2024-09-01T10:30:00Z")
}
```

### Collection: farmer_profiles

**Purpose**: Store farmer information and preferences.

```javascript
{
    "_id": ObjectId("9876543210987654fedcba98"),
    "farmer_id": "farmer_001",
    "name": "John Smith",
    "email": "john@greenvalley.com",
    "phone": "+1-555-0123",
    "farms": ["farm_001"],
    "preferences": {
        "alert_threshold_soil_moisture": 15,
        "alert_threshold_ndvi": 0.4,
        "notification_methods": ["email", "sms"]
    },
    "subscription_tier": "premium",
    "created_at": ISODate("2020-01-15")
}
```

### Collection: grazing_events

**Purpose**: Track livestock grazing schedules and intensity.

```javascript
{
    "_id": ObjectId("0987654321098765fedcba98"),
    "event_id": "graze_001",
    "field_id": "field_001",
    "livestock_type": "cattle",
    "head_count": 45,
    "start_date": ISODate("2024-10-01"),
    "end_date": ISODate("2024-10-07"),
    "stocking_rate": 2.5,
    "utilization_pct": 55,
    "notes": "Rotational grazing - paddock 1"
}
```

### Indexes

```javascript
// Geospatial index for field boundaries
db.fields.createIndex({ "boundary": "2dsphere" });

// Compound index for farm queries
db.fields.createIndex({ "farm_id": 1, "field_id": 1 });

// Index for treatment event queries
db.treatment_events.createIndex({ "field_id": 1, "applied_date": -1 });

// Index for latest metrics queries
db.fields.createIndex({ "latest_metrics.ndvi": 1 });
db.fields.createIndex({ "latest_metrics.soil_moisture": 1 });

// Text index for notes search
db.fields.createIndex({ "notes": "text" });
```

---

## 3. Redis - Real-Time Layer

### Purpose
In-memory storage for real-time metrics, alerts, caching, and pub/sub messaging.

### Key Naming Convention

```
{entity_type}:{entity_id}:{data_type}
```

### Data Structures

#### 1. Hashes - Latest Metrics per Field

**Key Pattern**: `field:{field_id}:latest`

```redis
HSET field:field_001:latest 
    ndvi 0.72
    soil_moisture 28.5
    grass_height_cm 12.3
    temperature_c 18.5
    humidity_pct 65
    updated_at "2024-10-15T14:30:00Z"
    risk_score 0.25
```

**TTL**: 15 minutes (auto-expire if no updates)

#### 2. Hashes - Rolling Aggregates

**Key Pattern**: `field:{field_id}:rolling:{period}`

```redis
HSET field:field_001:rolling:7d
    avg_ndvi 0.68
    avg_soil_moisture 25.3
    min_soil_moisture 18.2
    max_temperature_c 28.5
    ndvi_trend -0.04
    computed_at "2024-10-15T15:00:00Z"

EXPIRE field:field_001:rolling:7d 3600  # 1 hour TTL
```

#### 3. Streams - Alert Events

**Stream Name**: `alerts`

```redis
XADD alerts * 
    field_id field_001
    alert_type moisture_low
    value 9.2
    threshold 15.0
    severity warning
    timestamp "2024-10-15T14:35:00Z"
    message "Soil moisture below threshold"
```

**Consumer Groups** for processing:
```redis
XGROUP CREATE alerts alert_processors $ MKSTREAM
XREADGROUP GROUP alert_processors worker1 COUNT 10 STREAMS alerts >
```

#### 4. Sorted Sets - Maintenance Schedules

**Key Pattern**: `farm:{farm_id}:schedule`

```redis
ZADD farm:farm_001:schedule 
    1729036800 '{"task":"fertilize","field_id":"field_001","priority":"high"}'
    1729123200 '{"task":"irrigate","field_id":"field_002","priority":"medium"}'
    1729209600 '{"task":"inspect","field_id":"field_003","priority":"low"}'
```

**Score**: Unix timestamp for scheduled date

#### 5. Sets - Fields at Risk

**Key Pattern**: `risk:{risk_level}:fields`

```redis
SADD risk:high:fields field_001 field_005 field_012
SADD risk:medium:fields field_002 field_003 field_008
SADD risk:low:fields field_004 field_006 field_007
```

#### 6. Pub/Sub Channels

```redis
# Subscribe to alerts for a specific farm
SUBSCRIBE farm:farm_001:alerts

# Publish alert
PUBLISH farm:farm_001:alerts '{"field_id":"field_001","type":"drought_warning"}'
```

#### 7. Strings - Cached Query Results

**Key Pattern**: `cache:{query_hash}`

```redis
SET cache:geo_query_5km_station1 '[{"field_id":"field_001","distance_m":1200}]'
EXPIRE cache:geo_query_5km_station1 300  # 5 minute TTL
```

### TTL Policies

| Data Type          | Key Pattern                 | TTL       | Rationale                    |
|--------------------|-----------------------------|-----------|------------------------------|
| Latest Metrics     | field:*:latest              | 15 min    | Stale data should expire     |
| Rolling Aggregates | field:*:rolling:*           | 1 hour    | Recomputed hourly            |
| Query Cache        | cache:*                     | 5 min     | Prevent stale cache          |
| Alert Streams      | alerts                      | 24 hours  | Historical window            |
| Risk Sets          | risk:*:fields               | 30 min    | Recomputed frequently        |

---

## 4. Neo4j - Knowledge Graph

### Purpose
Model relationships between entities and encode advisory rules for recommendations.

### Node Types

```cypher
// Farm Node
CREATE (f:Farm {
    farm_id: "farm_001",
    name: "Green Valley Farm",
    region: "Midwest",
    total_area_ha: 250.5,
    established_date: date("2015-03-15")
})

// Field Node
CREATE (field:Field {
    field_id: "field_001",
    name: "North Pasture",
    area_ha: 45.2,
    soil_type: "loam",
    slope_pct: 3.5,
    current_risk_level: "low"
})

// Farmer Node
CREATE (farmer:Farmer {
    farmer_id: "farmer_001",
    name: "John Smith",
    email: "john@greenvalley.com",
    experience_years: 15
})

// Sensor Node
CREATE (sensor:Sensor {
    sensor_id: "sensor_soil_001",
    type: "soil_moisture",
    manufacturer: "SoilTech",
    installed_date: date("2023-01-15"),
    calibration_date: date("2024-06-01")
})

// CropSpecies Node
CREATE (species:CropSpecies {
    species_id: "sp_001",
    name: "Perennial Ryegrass",
    scientific_name: "Lolium perenne",
    drought_tolerance: "medium",
    optimal_ph_min: 5.5,
    optimal_ph_max: 7.0,
    optimal_temp_min: 10,
    optimal_temp_max: 25
})

// Treatment Node (historical events)
CREATE (treatment:Treatment {
    treatment_id: "treat_001",
    type: "fertilizer",
    product: "Urea 46-0-0",
    applied_date: date("2024-09-01"),
    rate_kg_ha: 50
})

// AdvisoryRule Node
CREATE (rule:AdvisoryRule {
    rule_id: "rule_001",
    name: "Low Moisture Alert",
    description: "Trigger irrigation when soil moisture drops below threshold",
    trigger_condition: "soil_moisture < 15 AND forecast_rain_mm < 5",
    priority: 1,
    action: "Schedule irrigation within 48 hours",
    expected_outcome: "Prevent forage stress and maintain NDVI above 0.5"
})

// RiskEvent Node (when thresholds are crossed)
CREATE (event:RiskEvent {
    event_id: "risk_001",
    event_type: "moisture_low",
    detected_at: datetime("2024-10-15T14:35:00Z"),
    metric_value: 9.2,
    threshold_value: 15.0,
    severity: "warning",
    resolved: false
})

// WeatherStation Node
CREATE (station:WeatherStation {
    station_id: "ws_001",
    name: "County Weather Station",
    latitude: 43.0731,
    longitude: -89.4012
})
```

### Relationship Types

```cypher
// Ownership and management relationships
(Farm)-[:OWNED_BY]->(Farmer)
(Farmer)-[:MANAGES]->(Farm)

// Field-Farm relationships
(Farm)-[:CONTAINS]->(Field)
(Field)-[:BELONGS_TO]->(Farm)

// Sensor relationships
(Field)-[:HAS_SENSOR]->(Sensor)
(Sensor)-[:MONITORS]->(Field)

// Species relationships
(Field)-[:GROWS]->(CropSpecies)
(CropSpecies)-[:PLANTED_IN]->(Field)

// Treatment relationships
(Field)-[:RECEIVED_TREATMENT]->(Treatment)
(Treatment)-[:APPLIED_TO]->(Field)
(Farmer)-[:APPLIED]->(Treatment)

// Advisory rule relationships
(AdvisoryRule)-[:APPLIES_TO]->(CropSpecies)
(AdvisoryRule)-[:APPLIES_TO]->(Field)
(AdvisoryRule)-[:RECOMMENDED_FOR {context: "drought"}]->(Field)

// Risk event relationships
(Field)-[:HAS_RISK_EVENT]->(RiskEvent)
(RiskEvent)-[:TRIGGERED_BY]->(AdvisoryRule)
(RiskEvent)-[:RESOLVED_BY]->(Treatment)

// Proximity relationships
(Field)-[:NEAR {distance_km: 2.5}]->(WeatherStation)
(Field)-[:ADJACENT_TO]->(Field)
```

### Example Graph Data

```cypher
// Create complete example graph
CREATE (farm:Farm {farm_id: "farm_001", name: "Green Valley Farm"})
CREATE (farmer:Farmer {farmer_id: "farmer_001", name: "John Smith"})
CREATE (field1:Field {field_id: "field_001", name: "North Pasture", soil_type: "loam"})
CREATE (field2:Field {field_id: "field_002", name: "South Meadow", soil_type: "clay"})
CREATE (sensor1:Sensor {sensor_id: "sensor_soil_001", type: "soil_moisture"})
CREATE (sensor2:Sensor {sensor_id: "sensor_weather_001", type: "weather"})
CREATE (grass:CropSpecies {species_id: "sp_001", name: "Perennial Ryegrass"})
CREATE (clover:CropSpecies {species_id: "sp_002", name: "White Clover"})
CREATE (rule1:AdvisoryRule {
    rule_id: "rule_001",
    name: "Low Moisture Alert",
    priority: 1
})
CREATE (rule2:AdvisoryRule {
    rule_id: "rule_002",
    name: "Overgrazing Prevention",
    priority: 2
})

// Create relationships
CREATE (farm)-[:OWNED_BY]->(farmer)
CREATE (farmer)-[:MANAGES]->(farm)
CREATE (farm)-[:CONTAINS]->(field1)
CREATE (farm)-[:CONTAINS]->(field2)
CREATE (field1)-[:HAS_SENSOR]->(sensor1)
CREATE (field1)-[:HAS_SENSOR]->(sensor2)
CREATE (field1)-[:GROWS {coverage_pct: 60}]->(grass)
CREATE (field1)-[:GROWS {coverage_pct: 25}]->(clover)
CREATE (field2)-[:GROWS {coverage_pct: 80}]->(grass)
CREATE (rule1)-[:APPLIES_TO]->(grass)
CREATE (rule1)-[:APPLIES_TO]->(clover)
CREATE (rule2)-[:APPLIES_TO]->(grass)
CREATE (field1)-[:ADJACENT_TO]->(field2)
CREATE (field2)-[:ADJACENT_TO]->(field1)
```

### Indexes

```cypher
// Unique constraints
CREATE CONSTRAINT farm_id_unique FOR (f:Farm) REQUIRE f.farm_id IS UNIQUE;
CREATE CONSTRAINT field_id_unique FOR (f:Field) REQUIRE f.field_id IS UNIQUE;
CREATE CONSTRAINT farmer_id_unique FOR (f:Farmer) REQUIRE f.farmer_id IS UNIQUE;
CREATE CONSTRAINT sensor_id_unique FOR (s:Sensor) REQUIRE s.sensor_id IS UNIQUE;
CREATE CONSTRAINT species_id_unique FOR (s:CropSpecies) REQUIRE s.species_id IS UNIQUE;
CREATE CONSTRAINT rule_id_unique FOR (r:AdvisoryRule) REQUIRE r.rule_id IS UNIQUE;

// Performance indexes
CREATE INDEX field_soil_type FOR (f:Field) ON (f.soil_type);
CREATE INDEX field_risk_level FOR (f:Field) ON (f.current_risk_level);
CREATE INDEX treatment_type FOR (t:Treatment) ON (t.type);
CREATE INDEX rule_priority FOR (r:AdvisoryRule) ON (r.priority);
```

---

## Integration Points

### Data Flow Summary

```
1. Sensors → Cassandra (raw telemetry, every 5-15 minutes)
2. Cassandra → Redis (aggregated metrics, every 15 minutes)
3. Redis → MongoDB (update latest_metrics in field documents, every 15 minutes)
4. MongoDB → Neo4j (sync field/treatment relationships, on event)
5. Neo4j → Redis (push advisory recommendations to alert stream)
```

### Cross-System Queries

For complex queries spanning multiple systems:

1. **Redis**: Get current metrics and identify at-risk fields
2. **Cassandra**: Pull historical trends for context
3. **MongoDB**: Get field metadata and geospatial context
4. **Neo4j**: Find related advisory rules and recommendations
