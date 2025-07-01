# Feature Engineering Approaches for Vibration Data Pipeline

## Overview

This document outlines two approaches for implementing feature engineering in the multi-machine vibration data streaming pipeline, comparing complete feature engineering in Node-RED vs StarRocks.

## Approach 1: Complete Feature Engineering in Node-RED

### Node-RED: Complete Feature Engineering

```javascript
// Node-RED Function - Calculate ALL features (RMS, Peak, Kurtosis)
const xArr = msg.payload.VibrationStreaming.VibrationXBatch.Value;
const yArr = msg.payload.VibrationStreaming.VibrationYBatch.Value;
const zArr = msg.payload.VibrationStreaming.VibrationZBatch.Value;

const { ServerTimestamp, SourceTimestamp, StatusCode } = msg.payload.VibrationStreaming.VibrationXBatch;

// Feature calculation helper functions
function rms(arr) {
    if (!Array.isArray(arr) || arr.length === 0) return null;
    const sumSq = arr.reduce((sum, v) => sum + (v * v), 0);
    return Math.sqrt(sumSq / arr.length);
}

function peak(arr) {
    if (!Array.isArray(arr) || arr.length === 0) return null;
    return Math.max(...arr.map(Math.abs));
}

function kurtosis(arr) {
    if (!Array.isArray(arr) || arr.length === 0) return null;
    const n = arr.length;
    const mean = arr.reduce((sum, v) => sum + v, 0) / n;
    const variance = arr.reduce((sum, v) => sum + Math.pow(v - mean, 2), 0) / n;
    const fourthMoment = arr.reduce((sum, v) => sum + Math.pow(v - mean, 4), 0) / n;
    return (fourthMoment / Math.pow(variance, 2)) - 3;
}

// Calculate ALL features in Node-RED + send raw data
msg.payload = {
    // Computed features
    vibration_x_rms: rms(xArr),
    vibration_y_rms: rms(yArr),
    vibration_z_rms: rms(zArr),
    vibration_x_peak: peak(xArr),
    vibration_y_peak: peak(yArr),
    vibration_z_peak: peak(zArr),
    vibration_x_kurtosis: kurtosis(xArr),
    vibration_y_kurtosis: kurtosis(yArr),
    vibration_z_kurtosis: kurtosis(zArr),
    
    // Raw arrays (required for Data Lake)
    VibrationXBatch: xArr,
    VibrationYBatch: yArr,
    VibrationZBatch: zArr,
    
    // Metadata
    ServerTimestamp,
    SourceTimestamp,
    StatusCode: StatusCode.Symbol || StatusCode
};

return msg;
```

### Kafka Message Structure (Approach 1)

```json
{
    "vibration_x_rms": 37.41122826104484,
    "vibration_y_rms": 87.14413348011443,
    "vibration_z_rms": 1024.0242672905754,
    "vibration_x_peak": 74.0,
    "vibration_y_peak": 134.0,
    "vibration_z_peak": 1136.0,
    "vibration_x_kurtosis": -0.512,
    "vibration_y_kurtosis": 1.234,
    "vibration_z_kurtosis": -0.678,
    "VibrationXBatch": [9, 19, -68, -29, 27, 74, -35, -11, 23, 3],
    "VibrationYBatch": [130, 0, -11, 134, 122, -35, -95, 107, 66, 5],
    "VibrationZBatch": [-991, -1032, -1065, -1046, -1136, -1116, -917, -864, -985, -1057],
    "ServerTimestamp": "2025-06-27T02:54:30.436Z",
    "SourceTimestamp": "2025-06-27T02:54:30.436Z",
    "StatusCode": "Good"
}
```

### StarRocks: Direct Storage via Routine Load (Approach 1)

```sql
-- StarRocks table for pre-calculated features from Node-RED
CREATE TABLE bosch_vibration_features (
    SourceTimestamp DATETIME NOT NULL COMMENT 'Timestamp from source',
    ServerTimestamp DATETIME NULL COMMENT 'Server timestamp', 
    StatusCode VARCHAR(16) NULL COMMENT 'Status code',
    vibration_x_rms DOUBLE NULL,
    vibration_y_rms DOUBLE NULL,
    vibration_z_rms DOUBLE NULL,
    vibration_x_peak DOUBLE NULL,
    vibration_y_peak DOUBLE NULL,
    vibration_z_peak DOUBLE NULL,
    vibration_x_kurtosis DOUBLE NULL,
    vibration_y_kurtosis DOUBLE NULL,
    vibration_z_kurtosis DOUBLE NULL
)
ENGINE = OLAP
DUPLICATE KEY(SourceTimestamp)
DISTRIBUTED BY HASH(SourceTimestamp) BUCKETS 8
PROPERTIES ("replication_num" = "1");

-- Routine Load for pre-calculated features (no computation needed)
CREATE ROUTINE LOAD telemetry.vibration_features_load
ON bosch_vibration_features
COLUMNS (
  SourceTimestamp,
  ServerTimestamp,
  StatusCode,
  vibration_x_rms,
  vibration_y_rms,
  vibration_z_rms,
  vibration_x_peak,
  vibration_y_peak,
  vibration_z_peak,
  vibration_x_kurtosis,
  vibration_y_kurtosis,
  vibration_z_kurtosis
)
PROPERTIES (
  "desired_concurrent_number" = "1",
  "format" = "json",
  "strip_outer_array" = "false",
  "timezone" = "Asia/Singapore",
  "jsonpaths" = "[\
\"$.SourceTimestamp\",\
\"$.ServerTimestamp\",\
\"$.StatusCode\",\
\"$.vibration_x_rms\",\
\"$.vibration_y_rms\",\
\"$.vibration_z_rms\",\
\"$.vibration_x_peak\",\
\"$.vibration_y_peak\",\
\"$.vibration_z_peak\",\
\"$.vibration_x_kurtosis\",\
\"$.vibration_y_kurtosis\",\
\"$.vibration_z_kurtosis\"\
]"
)
FROM KAFKA (
  "kafka_broker_list" = "my-release-kafka.p1-kafka.svc.cluster.local:9092",
  "kafka_topic" = "bosch-merged-data",
  "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

---

## Approach 2: Complete Feature Engineering in StarRocks

### Node-RED: Data Routing Only

```javascript
// Simplified Node-RED Function - NO feature calculations
const xArr = msg.payload.VibrationStreaming.VibrationXBatch.Value;
const yArr = msg.payload.VibrationStreaming.VibrationYBatch.Value;  
const zArr = msg.payload.VibrationStreaming.VibrationZBatch.Value;

const { ServerTimestamp, SourceTimestamp, StatusCode } = msg.payload.VibrationStreaming.VibrationXBatch;

// Send only raw arrays + metadata (no feature calculations)
msg.payload = {
    VibrationXBatch: xArr,
    VibrationYBatch: yArr,
    VibrationZBatch: zArr,
    ServerTimestamp,
    SourceTimestamp,
    StatusCode: StatusCode.Symbol || StatusCode
};

return msg;
```

### Kafka Message Structure (Approach 2)

```json
{
    "VibrationXBatch": [9, 19, -68, -29, 27, 74, -35, -11, 23, 3],
    "VibrationYBatch": [130, 0, -11, 134, 122, -35, -95, 107, 66, 5],
    "VibrationZBatch": [-991, -1032, -1065, -1046, -1136, -1116, -917, -864, -985, -1057],
    "ServerTimestamp": "2025-06-27T02:54:30.436Z",
    "SourceTimestamp": "2025-06-27T02:54:30.436Z",
    "StatusCode": "Good"
}
```

### StarRocks: Complete Feature Engineering via Routine Load (Approach 2)

```sql
-- Features table (same as Approach 1)
CREATE TABLE bosch_vibration_features (
    SourceTimestamp DATETIME NOT NULL COMMENT 'Timestamp from source',
    ServerTimestamp DATETIME NULL COMMENT 'Server timestamp',
    StatusCode VARCHAR(16) NULL COMMENT 'Status code',
    vibration_x_rms DOUBLE NULL,
    vibration_y_rms DOUBLE NULL,
    vibration_z_rms DOUBLE NULL,
    vibration_x_peak DOUBLE NULL,
    vibration_y_peak DOUBLE NULL,
    vibration_z_peak DOUBLE NULL,
    vibration_x_kurtosis DOUBLE NULL,
    vibration_y_kurtosis DOUBLE NULL,
    vibration_z_kurtosis DOUBLE NULL
)
ENGINE = OLAP
DUPLICATE KEY(SourceTimestamp)
DISTRIBUTED BY HASH(SourceTimestamp) BUCKETS 8
PROPERTIES ("replication_num" = "1");

-- Routine Load with feature calculations directly in the load process
CREATE ROUTINE LOAD telemetry.vibration_features_load
ON bosch_vibration_features
COLUMNS (
  SourceTimestamp,
  ServerTimestamp,
  StatusCode,
  VibrationXBatch,
  VibrationYBatch,
  VibrationZBatch,
  -- Calculate features during load
  vibration_x_rms = sqrt(array_avg(array_map(x -> x*x, VibrationXBatch))),
  vibration_y_rms = sqrt(array_avg(array_map(x -> x*x, VibrationYBatch))),
  vibration_z_rms = sqrt(array_avg(array_map(x -> x*x, VibrationZBatch))),
  vibration_x_peak = array_max(array_map(x -> abs(x), VibrationXBatch)),
  vibration_y_peak = array_max(array_map(x -> abs(x), VibrationYBatch)),
  vibration_z_peak = array_max(array_map(x -> abs(x), VibrationZBatch)),
  vibration_x_kurtosis = array_kurtosis(VibrationXBatch),
  vibration_y_kurtosis = array_kurtosis(VibrationYBatch),
  vibration_z_kurtosis = array_kurtosis(VibrationZBatch)
)
PROPERTIES (
  "desired_concurrent_number" = "1",
  "format" = "json",
  "strip_outer_array" = "false",
  "timezone" = "Asia/Singapore",
  "jsonpaths" = "[\
\"$.SourceTimestamp\",\
\"$.ServerTimestamp\",\
\"$.StatusCode\",\
\"$.VibrationXBatch\",\
\"$.VibrationYBatch\",\
\"$.VibrationZBatch\"\
]"
)
FROM KAFKA (
  "kafka_broker_list" = "my-release-kafka.p1-kafka.svc.cluster.local:9092",
  "kafka_topic" = "bosch-merged-data",
  "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

---

## Comparison Matrix

| Aspect | Approach 1 (Node-RED) | Approach 2 (StarRocks) |
|--------|----------------------|-------------------------|
| **Node-RED Complexity** | High (all feature calculations) | Low (data routing only) |
| **Kafka Message Size** | Large (features + raw arrays) | Medium (raw arrays only) |
| **StarRocks Processing** | None (direct storage) | High (all feature calculations) |
| **Feature Consistency** | Edge computing | Centralized in DW |
| **Maintainability** | JavaScript functions | SQL-based |
| **Computational Load** | Edge/streaming layer | Data warehouse layer |
| **Data Lake Content** | Features + raw data | Raw vibration arrays |
| **Future Extensibility** | Node-RED function updates | StarRocks SQL changes |
| **Real-time Performance** | Fast (pre-computed) | Batch processing |
| **Resource Efficiency** | Uses edge compute | Uses DW compute power |

---

## StarRocks Implementation Differences

### Approach 1: Pre-calculated Features Load
- **Tables**: 1 table (`bosch_vibration_features`)
- **Routine Load**: Direct feature ingestion from Kafka JSON
- **Processing**: None (features pre-calculated in Node-RED)
- **Complexity**: Simple JSONPath mapping
- **Storage**: Only computed features stored

### Approach 2: Real-time Feature Calculation
- **Tables**: 1 table (`bosch_vibration_features`)
- **Routine Load**: Array ingestion + feature calculation in COLUMNS clause
- **Processing**: SQL array functions during load (RMS, peak, kurtosis)
- **Complexity**: Complex COLUMNS transformations with array operations
- **Storage**: Only computed features stored (raw arrays discarded after calculation)

---

## Data Flow Comparison

### Approach 1 Flow (Edge Computing)
```
OPC UA → Node-RED (ALL features) → Kafka (features + raw arrays) → Data Lake (features + raw)
                                              ↓
                                        StarRocks (direct storage) → DW (9 features)
```

### Approach 2 Flow (Centralized Processing)
```
OPC UA → Node-RED (routing) → Kafka (raw arrays) → Data Lake (raw data)
                                    ↓
                              StarRocks (ALL features) → DW (9 features)
```

---

## Recommendation

### Choose Based on Your Requirements:

**Approach 1 (Node-RED)** is better for:
✅ **Real-time Performance**: Features pre-computed at edge  
✅ **Low Latency**: Immediate feature availability  
✅ **Reduced DW Load**: StarRocks only stores data  
✅ **Edge Computing**: Distribute processing load  
✅ **Feature Availability**: Features immediately available in both systems  

**Approach 2 (StarRocks)** is better for:
✅ **Data Integrity**: Raw data preserved for future analysis  
✅ **Centralized Logic**: All feature engineering in one place  
✅ **SQL-based Maintenance**: Easier for data engineers  
✅ **Computational Power**: Leverage DW resources  
✅ **Feature Consistency**: Uniform calculation methodology  
✅ **Audit Trail**: Full raw data history available  
✅ **Smaller Kafka Messages**: Only raw arrays transmitted  
✅ **Storage Efficiency**: No data duplication in transit  

### Operational Considerations:

**Approach 1** requires:
- Node-RED function maintenance for feature calculations
- Simple routine load job with JSONPath mapping
- Immediate feature availability

**Approach 2** requires:
- StarRocks array function support verification
- Complex routine load job with array transformations
- Real-time feature calculation during ingestion
- No additional storage overhead

**Note**: While calculating features after loading data into staging tables is also a valid approach, for vibration monitoring use cases, calculating features during the routine load process is more efficient and industry-recommended.

**Note**: When implementing , one can debug and load raw data via routine load to ensure data consistency
**Recommended: Approach 2 (StarRocks)** for most enterprise scenarios due to better maintainability, data preservation, and centralized feature management, despite the additional operational complexity. 