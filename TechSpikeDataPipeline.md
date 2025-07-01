# Tech Spike: Multi-Machine Vibration Data Streaming Pipeline

## Current State

### Existing Setup
- **Current Implementation**: `BoschDataServer.py` streams single HDF5 file (`M01_Aug_2019_OP00_000.h5`)
- **OPC UA Endpoint**: `opc.tcp://0.0.0.0:4840/`
- **Streaming Rate**: 1 sample per second
- **Node-RED Integration**: Flattens data using `node-red.js` to extract RMS values

### Current Data Structure
```json
{
    "VibrationXRMS": 37.41122826104484,
    "VibrationYRMS": 87.14413348011443,
    "VibrationZRMS": 1024.0242672905754,
    "VibrationXBatch": [9, 19, -68, ...],
    "VibrationYBatch": [130, 0, -11, ...],
    "VibrationZBatch": [-991, -1032, -1065, ...],
    "ServerTimestamp": "2025-06-27T02:54:30.436Z",
    "SourceTimestamp": "2025-06-27T02:54:30.436Z",
    "StatusCode": "Good"
}
```

## New Multi-Machine Setup

### Data Sources
- **Machines**: M01, M02 (M03 excluded for major version)
- **Operations**: OP00-OP14 
- **Exclusions**: OP00, OP06, OP09, OP13 for M01 & M02
- **Data Format**: HDF5 files located under `/data` folder
- **Target Location**: CNC Bosch data folder

### Architecture Components

#### Sidecar Container Pattern
- **Sidecar Container**: Manages data placement into shared folder
- **Shared Storage**: Kubernetes mount path for data sharing
- **Main OPC UA Container**: Streams data from shared folder
- **Node-RED Container**: Reads and flattens OPC UA data

#### Data Flow Pipeline

1. **Data Ingestion**
   - Sidecar container → Shared K8s mount path
   - OPC UA container → Streams HDF5 data
   - Node-RED → Reads and flattens data

2. **Data Reduction**
   - **Input**: 10 data points from Node-RED
   - **Processing**: Reduce to 1 data point per second
   - **Output**: VibrationXRMS, VibrationYRMS, VibrationZRMS

3. **Data Aggregation**
   - **Collection Period**: 60 records (1 per second = 1 minute)
   - **Target**: Kafka topic via flattened.json format

4. **Feature Extraction**
   - **Source**: 60-second aggregated data
   - **Features**: 
     - `vibration_x_rms`, `vibration_y_rms`, `vibration_z_rms`
     - `vibration_x_peak`, `vibration_y_peak`, `vibration_z_peak`
     - `vibration_x_kurtosis`, `vibration_y_kurtosis`, `vibration_z_kurtosis`

5. **Storage & Visualization**
   - **Data Warehouse**: Store extracted features
   - **Visualization**: Grafana dashboards

### Data Processing Flow

```
HDF5 Files (M01/M02) → Sidecar Container → Shared K8s Mount
                                               ↓
Node-RED ← OPC UA Container ← Shared K8s Mount
    ↓
Kafka (flattened.json)
    ↓
Feature Extraction (60 records/minute)
    ↓
Data Warehouse
    ↓
Grafana Visualization
```

### Operational Scope

#### Included Operations
- **M01**: OP01, OP02, OP03, OP04, OP05, OP07, OP08, OP10, OP11, OP12, OP14
- **M02**: OP01, OP02, OP03, OP04, OP05, OP07, OP08, OP10, OP11, OP12, OP14

#### Excluded Operations
- **M01 & M02**: OP00, OP06, OP09, OP13
- **M03**: Entire machine (deferred to major version)

### Feature Set
**9 Features per Machine/Operation**:
- 3 RMS values (X, Y, Z vibration axes)
- 3 Peak values (X, Y, Z vibration axes)  
- 3 Kurtosis values (X, Y, Z vibration axes) 