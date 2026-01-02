# NoSQL Pasture Management System

A comprehensive NoSQL-based solution for analyzing and improving pasture/fodder quality with practical recommendations for farmers.

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA SOURCES                                            │
├─────────────┬──────────────┬──────────────┬─────────────────┬──────────────────────┤
│   Sensors   │   Farmer     │   Satellite  │  Weather APIs   │  Simulated Data      │
│  (IoT/MQTT) │  Records     │   (NDVI)     │   (External)    │   Generator          │
└──────┬──────┴──────┬───────┴──────┬───────┴────────┬────────┴──────────┬───────────┘
       │             │              │                │                   │
       ▼             ▼              ▼                ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           INGESTION LAYER                                            │
│    Python ETL Scripts / Apache Kafka (optional) / Batch Processors                   │
└────────────────────────────────────────┬────────────────────────────────────────────┘
                                         │
         ┌───────────────────────────────┼───────────────────────────────┐
         │                               │                               │
         ▼                               ▼                               ▼
┌─────────────────┐           ┌─────────────────┐            ┌─────────────────┐
│    CASSANDRA    │           │     MONGODB     │            │      REDIS      │
│  Time-Series    │           │   Document      │            │   Real-Time     │
│    Storage      │           │    Storage      │            │     Layer       │
├─────────────────┤           ├─────────────────┤            ├─────────────────┤
│ • Sensor data   │           │ • Farm profiles │            │ • Latest metrics│
│ • High-write    │◄─────────►│ • Field metadata│◄──────────►│ • Alert streams │
│ • Time-range    │  periodic │ • Geo-spatial   │  real-time │ • Cached aggs   │
│   queries       │  sync     │ • Events/logs   │   updates  │ • Pub/Sub       │
│ • TTL policies  │           │ • Treatment hist│            │ • Sorted sets   │
└────────┬────────┘           └────────┬────────┘            └────────┬────────┘
         │                             │                              │
         │                             │                              │
         └─────────────────────────────┼──────────────────────────────┘
                                       │
                                       ▼
                            ┌─────────────────────┐
                            │       NEO4J         │
                            │   Knowledge Graph   │
                            ├─────────────────────┤
                            │ • Field-Farm-Farmer │
                            │   relationships     │
                            │ • Advisory rules    │
                            │ • Treatment history │
                            │ • Species mapping   │
                            │ • Risk propagation  │
                            └──────────┬──────────┘
                                       │
                                       ▼
                            ┌─────────────────────┐
                            │  ANALYTICS ENGINE   │
                            │  & DASHBOARD        │
                            ├─────────────────────┤
                            │ • Risk assessment   │
                            │ • Recommendations   │
                            │ • Visualization     │
                            │ • Farmer alerts     │
                            └─────────────────────┘
```

## 📦 System Components

### 1. Apache Cassandra (Time-Series Storage)
- **Role**: High-throughput storage for sensor telemetry data
- **Data**: Temperature, soil moisture, humidity, NDVI, grass height measurements
- **Features**: Time-range queries, automatic compaction, TTL for data retention
- **Write Pattern**: ~1000 writes/second per field during peak sensor activity

### 2. MongoDB (Document Storage)
- **Role**: Flexible document storage for metadata and events
- **Data**: Farm profiles, field boundaries (GeoJSON), treatment events, farmer records
- **Features**: Geospatial queries (2dsphere), aggregation pipelines, flexible schema
- **Query Pattern**: Complex aggregations, geo-near queries, full-text search

### 3. Redis (Real-Time Layer)
- **Role**: In-memory caching and real-time analytics
- **Data**: Latest metrics per field, active alerts, rolling aggregations
- **Features**: Pub/Sub for alerts, Streams for event logging, Sorted Sets for schedules
- **TTL**: Short-lived data (15 min - 24 hours depending on metric type)

### 4. Neo4j (Knowledge Graph)
- **Role**: Relationship modeling and advisory rule engine
- **Data**: Field-Farm-Farmer relationships, treatment history, advisory rules
- **Features**: Graph traversals, pattern matching, recommendation engine
- **Query Pattern**: Find related fields, propagate risks, match advisory rules

## 🚀 Quick Start

### Prerequisites
- Docker & Docker Compose
- Python 3.10+
- Node.js 18+ (for dashboard)

### Installation

```bash
# Clone the repository
git clone <repository-url>
cd nosql-pasture-management

# Start all databases with Docker Compose
docker-compose up -d

# Install Python dependencies
pip install -r requirements.txt

# Generate synthetic data
python scripts/data_generator.py

# Run ingestion pipeline
python scripts/ingest_cassandra.py
python scripts/ingest_mongodb.py
python scripts/aggregate_to_redis.py
python scripts/build_neo4j_graph.py

# Launch dashboard
cd dashboard && npm install && npm start
```

### Docker Compose Services

| Service   | Port(s)        | Description                    |
|-----------|----------------|--------------------------------|
| Cassandra | 9042           | CQL native transport           |
| MongoDB   | 27017          | MongoDB wire protocol          |
| Redis     | 6379           | Redis protocol                 |
| Neo4j     | 7474, 7687     | HTTP API & Bolt protocol       |

## 📁 Project Structure

```
nosql-pasture-management/
├── README.md                    # This file
├── docker-compose.yml           # Container orchestration
├── requirements.txt             # Python dependencies
├── docs/
│   ├── data_models.md           # Detailed data model documentation
│   ├── architecture.md          # System architecture details
│   └── recommendations.md       # Agronomic recommendations
├── scripts/
│   ├── data_generator.py        # Synthetic data generation
│   ├── ingest_cassandra.py      # Cassandra ingestion
│   ├── ingest_mongodb.py        # MongoDB ingestion
│   ├── aggregate_to_redis.py    # Redis aggregation job
│   ├── build_neo4j_graph.py     # Neo4j graph builder
│   └── analytics_queries.py     # Cross-system analytics
├── queries/
│   ├── cassandra_queries.cql    # CQL query examples
│   ├── mongodb_queries.js       # MongoDB query examples
│   ├── redis_commands.txt       # Redis command examples
│   └── neo4j_queries.cypher     # Cypher query examples
├── data/
│   ├── sample_farms.json        # Sample farm data
│   ├── sample_fields.json       # Sample field data
│   └── sensor_config.json       # Sensor configuration
└── dashboard/
    ├── index.html               # Dashboard UI
    └── app.js                   # Dashboard logic
```

## 🔧 Configuration

### Environment Variables

```bash
# Cassandra
CASSANDRA_HOST=localhost
CASSANDRA_PORT=9042
CASSANDRA_KEYSPACE=pasture_mgmt

# MongoDB
MONGODB_URI=mongodb://localhost:27017
MONGODB_DATABASE=pasture_mgmt

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password
```

# Dashboard
<img width="1900" height="957" alt="image" src="https://github.com/user-attachments/assets/8e482999-b27e-4f57-9670-f79ec46891ae" />



## 📊 Key Metrics Tracked

| Category      | Metrics                                           |
|---------------|---------------------------------------------------|
| Climate       | Temperature, precipitation, humidity, solar rad. |
| Soil          | Moisture, pH, N/P/K levels, organic matter        |
| Vegetation    | NDVI, grass height, biomass density               |
| Management    | Grazing intensity, rest days, fertilizer events   |
| Derived       | 7-day rolling averages, anomaly scores            |

## 📝 License

This project is for educational purposes as part of the NoSQL course final assignment at Riga Nordic University.

## 👥 Team
Bekjon Ibragimov
