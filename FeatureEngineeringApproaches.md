# Feature Engineering Approaches for Vibration Data Pipeline

## Overview

This document outlines two approaches for implementing feature engineering in the multi-machine vibration data streaming pipeline, comparing where to calculate RMS, peak, and kurtosis features.

## Approach 1: Hybrid Processing (Node-RED + StarRocks)

### Node-RED: RMS Calculation + Data Flattening

```javascript
// Node-RED Function - Calculate RMS, send raw arrays
const xArr = msg.payload.VibrationStreaming.VibrationXBatch.Value;
const yArr = msg.payload.VibrationStreaming.VibrationYBatch.Value;
const zArr = msg.payload.VibrationStreaming.VibrationZBatch.Value;

const { ServerTimestamp, SourceTimestamp, StatusCode } = msg.payload.VibrationStreaming.VibrationXBatch;

// RMS helper function
function rms(arr) {
    if (!Array.isArray(arr) || arr.length === 0) return null;
    const sumSq = arr.reduce((sum, v) => sum + (v * v), 0);
    return Math.sqrt(sumSq / arr.length);
}

// Calculate RMS in Node-RED
const VibrationXRMS = rms(xArr);
const VibrationYRMS = rms(yArr);
const VibrationZRMS = rms(zArr);

// Send RMS + raw arrays
msg.payload = {
    // Pre-calculated RMS values
    VibrationXRMS,
    VibrationYRMS,
    VibrationZRMS,
    
    // Raw arrays for additional processing
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
    "VibrationXRMS": 37.41122826104484,
    "VibrationYRMS": 87.14413348011443,
    "VibrationZRMS": 1024.0242672905754,
    "VibrationXBatch": [9, 19, -68, -29, 27, 74, -35, -11, 23, 3],
    "VibrationYBatch": [130, 0, -11, 134, 122, -35, -95, 107, 66, 5],
    "VibrationZBatch": [-991, -1032, -1065, -1046, -1136, -1116, -917, -864, -985, -1057],
    "ServerTimestamp": "2025-06-27T02:54:30.436Z",
    "SourceTimestamp": "2025-06-27T02:54:30.436Z",
    "StatusCode": "Good"
}
```

### StarRocks: Peak & Kurtosis Calculation (Approach 1)

```sql
CREATE TABLE vibration_features AS
SELECT 
    ServerTimestamp,
    SourceTimestamp,
    
    -- Use pre-calculated RMS values from Node-RED
    VibrationXRMS as vibration_x_rms,
    VibrationYRMS as vibration_y_rms, 
    VibrationZRMS as vibration_z_rms,
    
    -- Calculate Peak from batch arrays
    array_max(array_map(x -> abs(x), VibrationXBatch)) as vibration_x_peak,
    array_max(array_map(x -> abs(x), VibrationYBatch)) as vibration_y_peak,
    array_max(array_map(x -> abs(x), VibrationZBatch)) as vibration_z_peak,
    
    -- Calculate Kurtosis from batch arrays
    array_kurtosis(VibrationXBatch) as vibration_x_kurtosis,
    array_kurtosis(VibrationYBatch) as vibration_y_kurtosis,
    array_kurtosis(VibrationZBatch) as vibration_z_kurtosis
    
FROM kafka_vibration_data;
```

---

## Approach 2: StarRocks-Only Processing (Recommended)

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

### StarRocks: Complete Feature Engineering (Approach 2)

```sql
CREATE TABLE vibration_features AS
SELECT 
    ServerTimestamp,
    SourceTimestamp,
    
    -- Calculate ALL features from raw arrays in StarRocks
    
    -- RMS calculation
    sqrt(array_avg(array_map(x -> x*x, VibrationXBatch))) as vibration_x_rms,
    sqrt(array_avg(array_map(x -> x*x, VibrationYBatch))) as vibration_y_rms,
    sqrt(array_avg(array_map(x -> x*x, VibrationZBatch))) as vibration_z_rms,
    
    -- Peak calculation  
    array_max(array_map(x -> abs(x), VibrationXBatch)) as vibration_x_peak,
    array_max(array_map(x -> abs(x), VibrationYBatch)) as vibration_y_peak,
    array_max(array_map(x -> abs(x), VibrationZBatch)) as vibration_z_peak,
    
    -- Kurtosis calculation
    array_kurtosis(VibrationXBatch) as vibration_x_kurtosis,
    array_kurtosis(VibrationYBatch) as vibration_y_kurtosis,
    array_kurtosis(VibrationZBatch) as vibration_z_kurtosis
    
FROM kafka_vibration_raw;
```

---

## Comparison Matrix

| Aspect | Approach 1 (Hybrid) | Approach 2 (StarRocks-Only) |
|--------|---------------------|------------------------------|
| **Node-RED Complexity** | Medium (RMS calculation) | Low (data routing only) |
| **Kafka Message Size** | Larger (RMS + arrays) | Smaller (arrays only) |
| **StarRocks Processing** | Medium (2 features) | High (all 3 features) |
| **Feature Consistency** | Split across systems | Centralized |
| **Maintainability** | Moderate | High |
| **Computational Load** | Distributed | Concentrated in StarRocks |
| **Data Lake Content** | RMS + raw data | Raw data only |
| **Future Extensibility** | Requires Node-RED changes | StarRocks SQL changes only |

---

## Data Flow Comparison

### Approach 1 Flow
```
OPC UA → Node-RED (RMS calc) → Kafka (RMS + arrays) → Data Lake (full)
                                    ↓
                              StarRocks (Peak + Kurtosis) → DW (9 features)
```

### Approach 2 Flow  
```
OPC UA → Node-RED (routing) → Kafka (arrays only) → Data Lake (raw)
                                   ↓
                             StarRocks (All features) → DW (9 features)
```

---

## Recommendation

**Approach 2 (StarRocks-Only)** is recommended for:

✅ **Clean Architecture**: Clear separation of concerns  
✅ **Maintainability**: All feature logic in one place  
✅ **Consistency**: Uniform feature calculation methodology  
✅ **Performance**: StarRocks optimized for mathematical operations  
✅ **Flexibility**: Easy to add new features without Node-RED changes  
✅ **Data Integrity**: Raw data preserved in Data Lake for future analysis 