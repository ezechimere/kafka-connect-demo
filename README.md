# Kafka Connect JDBC Integration

Production-grade data integration pipeline demonstrating bidirectional data flow between PostgreSQL and Apache Kafka using Kafka Connect framework with comprehensive error handling.

## Overview

Enterprise data integration system leveraging Kafka Connect to build scalable, fault-tolerant pipelines for continuous data synchronization between relational databases and Kafka topics. Implements source and sink connectors with incremental updates, upsert capabilities, and dead letter queue (DLQ) error isolation.

## Business Case

Modern enterprises need reliable data synchronization between operational databases and streaming platforms. This implementation addresses:

- **Real-time Data Replication**: Continuous CDC (Change Data Capture) from PostgreSQL
- **Bidirectional Sync**: Read from and write to databases without custom code
- **Error Isolation**: DLQ pattern prevents pipeline failures from bad records
- **Transform in Flight**: Apply transformations during data movement
- **Operational Simplicity**: Declarative configuration, no code required
- **Scalability**: Parallel task execution across workers

## Architecture
```
PostgreSQL (customers table)
    ↓
JDBC Source Connector (incremental mode)
    ↓
Kafka Topic (postgres-customers)
    ↓
JDBC Sink Connector (upsert mode + DLQ)
    ↓
PostgreSQL (customers_sink table)
```

### Data Flow Pattern

1. **Source Phase**: JDBC Source monitors PostgreSQL for new/updated records
2. **Stream Phase**: Records flow through Kafka with full durability guarantees
3. **Transform Phase**: Optional SMTs enrich data (timestamps, masking, routing)
4. **Sink Phase**: JDBC Sink writes to downstream PostgreSQL with upsert logic
5. **Error Handling**: Failed records routed to DLQ for analysis and replay

## Technical Implementation

### Connectors Deployed

**1. jdbc-source-customers**
- **Purpose**: Capture new customer records from PostgreSQL
- **Mode**: Incrementing (timestamp-based)
- **Polling**: 5-second intervals
- **Column**: `created_at` for incremental detection
- **Topic**: `postgres-customers`

**2. jdbc-sink-customers-dlq**
- **Purpose**: Write customer data to downstream database
- **Mode**: Upsert (INSERT or UPDATE based on primary key)
- **Error Handling**: DLQ enabled for data type mismatches
- **DLQ Topic**: `dlq-jdbc-sink-customers`
- **Target**: `customers_sink` table

**3. jdbc-source-with-smt**
- **Purpose**: Source with Single Message Transform
- **Transform**: Add `processing_timestamp` field
- **Use Case**: Audit trail and latency monitoring
- **Topic**: `postgres-smt-customers`

**4. mock-source-connector**
- **Purpose**: Testing and validation
- **Use Case**: Development and connector health checks

### Key Features

**Incremental Data Capture**
- Timestamp-based change detection
- Avoids full table scans
- Minimal database load
- Configurable polling intervals

**Upsert Pattern**
- Primary key-based merge logic
- INSERT new records
- UPDATE existing records
- Idempotent operations

**Dead Letter Queue**
- Isolates problematic records
- Prevents pipeline stalls
- Enables async error handling
- Maintains data integrity

**Single Message Transforms (SMT)**
- Add fields (timestamps, metadata)
- Mask sensitive data
- Route to different topics
- Transform data types
- No code deployment required

**REST API Management**
- Deploy/update/delete connectors
- Check connector status
- View task details
- Pause/resume operations

## Configuration Examples

### JDBC Source Connector
```json
{
  "name": "jdbc-source-customers",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/customerdb",
    "connection.user": "admin",
    "connection.password": "password",
    "table.whitelist": "customers",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "postgres-",
    "poll.interval.ms": "5000"
  }
}
```

### JDBC Sink Connector with DLQ
```json
{
  "name": "jdbc-sink-customers-dlq",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/customerdb",
    "connection.user": "admin",
    "connection.password": "password",
    "topics": "postgres-customers",
    "auto.create": "true",
    "insert.mode": "upsert",
    "pk.mode": "record_key",
    "pk.fields": "id",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq-jdbc-sink-customers",
    "errors.deadletterqueue.topic.replication.factor": "1"
  }
}
```

### Source with SMT
```json
{
  "name": "jdbc-source-with-smt",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/customerdb",
    "table.whitelist": "customers",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "postgres-smt-",
    "transforms": "AddTimestamp",
    "transforms.AddTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.AddTimestamp.timestamp.field": "processing_timestamp"
  }
}
```

## Project Structure
```
kafka-connect-demo/
├── customer_data.txt        # Sample customer data for testing
├── README.md
└── .gitignore
```

## Technologies

- **Apache Kafka** - Distributed streaming platform
- **Kafka Connect** - Integration framework
- **PostgreSQL** - Source and sink database
- **JDBC Connectors** - Database integration
- **REST API** - Connector management
- **Docker** - Container orchestration

## Setup & Execution

### Prerequisites
```bash
# Kafka running on localhost:9092
# Kafka Connect on localhost:8083
# PostgreSQL on localhost:5432
```

### Create Source Table
```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO customers (name, email) VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Charlie Brown', 'charlie@example.com');
```

### Deploy Source Connector
```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @jdbc-source-config.json
```

### Verify Data Flow
```bash
# Check connector status
curl http://localhost:8083/connectors/jdbc-source-customers/status

# Consume from topic
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic postgres-customers --from-beginning
```

### Deploy Sink Connector
```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @jdbc-sink-config.json
```

### Check Results
```sql
-- Verify data in sink table
SELECT * FROM customers_sink;
```

## Monitoring & Operations

### Connector Health Checks
```bash
# List all connectors
curl http://localhost:8083/connectors

# Check specific connector status
curl http://localhost:8083/connectors/jdbc-source-customers/status

# View connector configuration
curl http://localhost:8083/connectors/jdbc-source-customers
```

### Consumer Group Lag
```bash
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group connect-jdbc-sink-customers-dlq
```

### DLQ Monitoring
```bash
# Check for failed records
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic dlq-jdbc-sink-customers --from-beginning
```

## Error Handling Strategies

**Tolerance Levels**
- `none`: Stop connector on first error (default)
- `all`: Route errors to DLQ, continue processing

**DLQ Configuration**
- Separate topic for failed records
- Includes error metadata (stack trace, original message)
- Enables async replay after fixes

**Retry Policies**
- `errors.retry.timeout`: Max retry duration
- `errors.retry.delay.max.ms`: Backoff between retries

## Performance Tuning

**Source Connector**
- `poll.interval.ms`: Balance freshness vs load
- `batch.max.rows`: Records per poll (default: 100)
- `tasks.max`: Parallel tasks for throughput

**Sink Connector**
- `batch.size`: Records per batch write
- `tasks.max`: Parallel sink tasks
- `max.retries`: Retry failed writes

## Key Learnings

**Connector Architecture**
- Source vs Sink connector patterns
- Task distribution and parallelism
- Offset management and exactly-once semantics

**Configuration Best Practices**
- Connection pooling for databases
- Error tolerance strategies
- Transform application order

**Operational Excellence**
- REST API for lifecycle management
- Monitoring connector health
- DLQ analysis and replay procedures

**Production Considerations**
- Connector worker high availability (minimum 2)
- Database connection limits
- Network bandwidth planning
- Schema evolution compatibility

## Use Cases

- **Data Lake Ingestion**: Stream OLTP data to analytical stores
- **Database Replication**: Real-time multi-region sync
- **Event Sourcing**: Capture database changes as events
- **Legacy Modernization**: Bridge old systems to event-driven architecture
- **Microservices Integration**: Share data without tight coupling

## Related Projects

Enterprise-grade Kafka implementations demonstrating end-to-end streaming architectures:

- [CDR Fraud Detection](https://github.com/ezechimere/cdr-fraud-detection) - Real-time telecom fraud detection
- [E-Commerce Analytics](https://github.com/ezechimere/ecommerce-analytics) - ksqlDB-based sales analytics
- [Clickstream Analytics](https://github.com/ezechimere/clickstream-analytics) - Kafka Streams session tracking
- [Schema Registry Demo](https://github.com/ezechimere/schema-registry-demo) - Avro serialization with type safety

## Author

**Conrad Mba** - Software Engineer | Data Engineering & Real-Time Systems | Kafka Specialist

- LinkedIn: [https://www.linkedin.com/in/conrad-mba/]
- Email: [mbaconrad@gmail.com]
- GitHub: [https://github.com/ezechimere]
