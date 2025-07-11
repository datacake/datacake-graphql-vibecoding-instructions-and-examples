# Datacake API Semantics Feature Guide

This guide covers how to use the **Semantics** feature in the Datacake GraphQL API, specifically with the `devicesFiltered` query to filter devices by semantic values and retrieve aggregated measurements.

## Table of Contents

1. [Overview](#overview)
2. [Available Semantics](#available-semantics)  
3. [Getting Workspace Semantics](#getting-workspace-semantics)
4. [Filtering Devices by Semantics](#filtering-devices-by-semantics)
5. [Device Grouping and Organization](#device-grouping-and-organization)
6. [Aggregating Semantic Values](#aggregating-semantic-values)
7. [Practical Working Examples](#practical-working-examples)
8. [Complete Query Examples](#complete-query-examples)
9. [Use Cases](#use-cases)
10. [Best Practices](#best-practices)

## Overview

**Semantics** in Datacake allow you to classify device measurement fields by their meaning rather than their technical field names. For example, instead of working with field names like `temp_01` or `temperature_sensor`, you can work with the semantic type `TEMPERATURE`.

Key benefits:
- **Unified queries** across different device types
- **Standardized filtering** regardless of underlying field names
- **Cross-device aggregation** for similar measurement types
- **Simplified analytics** and dashboard creation

## Available Semantics

The following semantic types are available in the Datacake API:

```graphql
enum FieldSemantic {
  AIR_POLLUTION       # Air quality measurements (PM2.5, PM10, etc.)
  AMBIENT_LIGHT       # Light level measurements (lux, etc.)
  BATTERY             # Battery level/voltage measurements
  CO2                 # Carbon dioxide measurements
  ENERGY_CONSUMPTION  # Energy usage measurements
  FILL_LEVEL          # Tank/container fill level measurements
  HUMIDITY            # Relative humidity measurements
  LOCATION            # GPS/positioning data
  LOUDNESS            # Noise level measurements (dB)
  PEOPLE_COUNT        # People counting/occupancy measurements
  POWER               # Power consumption/output measurements
  SIGNAL              # Signal strength measurements (RSSI, etc.)
  SOIL_MOISTURE       # Soil moisture content measurements
  TEMPERATURE         # Temperature measurements
  VOC                 # Volatile Organic Compounds measurements
  WATER_CONSUMPTION   # Water usage measurements
  WATER_DEPTH         # Water depth/level measurements
}
```

## Getting Workspace Semantics

To discover which semantics are available in your workspace:

```graphql
query GetWorkspaceSemantics {
  workspace(id: "YOUR_WORKSPACE_ID") {
    semantics
  }
}
```

**Response:**
```json
{
  "data": {
    "workspace": {
      "semantics": ["TEMPERATURE", "HUMIDITY", "BATTERY", "CO2"]
    }
  }
}
```

## Filtering Devices by Semantics

### Basic Numeric Filters

Use semantic filters to find devices with measurements in specific ranges:

```graphql
query FilterDevicesByTemperature {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { gt: 20.0, lt: 30.0 }
    ) {
      total
      devices {
        id
        verboseName
        numericSemanticField(semantic: TEMPERATURE) {
          value
          fields {
            fieldName
            verboseFieldName
            value
            unit
          }
        }
      }
    }
  }
}
```

### Available Filter Operators

Each semantic field filter supports these operators:

```graphql
input FilteredDeviceListNumericSemanticFieldFilterInput {
  # Aggregation method for devices with multiple fields of same semantic
  aggregation: NumericSemanticFieldAggregation = AVG
  
  # Range filter (inclusive)
  range: {
    start: Float!
    end: Float!
  }
  
  # Comparison operators
  gt: Float      # Greater than
  gte: Float     # Greater than or equal
  lt: Float      # Less than  
  lte: Float     # Less than or equal
}
```

### Aggregation Options

When a device has multiple measurement fields of the same semantic type, specify how to aggregate them:

```graphql
enum NumericSemanticFieldAggregation {
  AVG    # Average value (default)
  SUM    # Sum of all values
  MAX    # Maximum value
  MIN    # Minimum value
}
```

**Example with aggregation:**

```graphql
query FilterByAggregatedTemperature {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { 
        gt: 25.0,
        aggregation: MAX  # Use highest temperature reading
      }
    ) {
      total
      devices {
        id
        verboseName
      }
    }
  }
}
```

### Multiple Semantic Filters

Combine multiple semantic filters using **AND logic** - devices must have **ALL** specified semantics and match **ALL** filter conditions:

```graphql
query FilterByMultipleSemantics {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { gte: 20.0, lte: 30.0 }
      humidity: { gte: 40.0, lte: 60.0 }
      battery: { gt: 20.0 }
      co2: { lt: 1000.0 }
    ) {
      total
      devices {
        id
        verboseName
        online
      }
    }
  }
}
```

### Combined Semantics with Aggregation

**Important**: When filtering by multiple semantics, devices must have **both semantic types** and meet **all filter conditions**:

```graphql
query FilterByMultipleSemanticTypes {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { contains: ["CO2"] }       # Only CO2-tagged devices
      co2: {
        gt: 400                         # CO2 above 400 ppm
        aggregation: AVG                # Average if multiple CO2 fields
      }
      temperature: { 
        gt: 10.0                        # Temperature above 10°C
        aggregation: AVG                # Average if multiple temp fields
      }
    ) {
      total
      temperature_avg: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE, 
        aggregation: AVG
      )
      co2_avg: aggregatedNumericSemanticValue(
        semantic: CO2, 
        aggregation: AVG
      )
      devices {
        id
        verboseName
        co2: numericSemanticField(semantic: CO2) { 
          value 
        }
        temperature: numericSemanticField(semantic: TEMPERATURE) { 
          value 
        }
      }
    }
  }
}
```

This query **only returns devices that have BOTH temperature AND CO2 semantics** and where both values meet the specified thresholds.

Returns:

```
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 3,
        "temperature_avg": 23.53333333333333,
        "devices": [
          {
            "id": "d01eb653-aea2-4d79-a1fb-2faaab90e8d2",
            "verboseName": "CO2 - Entrance",
            "co2": {
              "value": 479
            },
            "temperature": {
              "value": 23.9
            }
          },
          {
            "id": "6bfc947a-8b63-402c-8eab-36e5909984bd",
            "verboseName": "CO2 - Mid Room",
            "co2": {
              "value": 430
            },
            "temperature": {
              "value": 23.1
            }
          },
          {
            "id": "ce5beb35-673f-4215-bb59-3e0ee925a42e",
            "verboseName": "CO2 - Right Exit",
            "co2": {
              "value": 604
            },
            "temperature": {
              "value": 23.6
            }
          }
        ]
      }
    }
  }
}
```

### Range Filtering

Use range filters for inclusive boundaries:

```graphql
query FilterByTemperatureRange {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { 
        range: { start: 18.0, end: 25.0 }
      }
    ) {
      total
    }
  }
}
```

## Device Grouping and Organization

For large deployments with hundreds of devices, you need efficient ways to group and filter devices by location, type, or function. The `devicesFiltered` query supports powerful grouping through **tags** and **search** parameters.

### Tag-Based Grouping (Recommended)

**Tags are the recommended way to organize devices** into logical groups like floors, rooms, buildings, or device types.

#### Tag Filter Types

```graphql
input FilteredDeviceListTagsFilterInput {
  contains: [String]  # Matches devices that have ALL specified tags
  overlap: [String]   # Matches devices that have AT LEAST ONE of the specified tags
}
```

#### Contains vs Overlap

- **`contains`**: Device must have **ALL** specified tags (AND logic)
- **`overlap`**: Device must have **AT LEAST ONE** of the specified tags (OR logic)

### Basic Tag Filtering

**Find all CO2 sensors:**
```graphql
query CO2Sensors {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { contains: ["CO2"] }
    ) {
      total
      devices {
        id
        verboseName
        tags
      }
    }
  }
}
```

**Find devices on specific floors:**
```graphql
query FloorDevices {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { overlap: ["floor-1", "floor-2"] }  # Devices on floor 1 OR floor 2
    ) {
      total
      devices {
        id
        verboseName
        tags
      }
    }
  }
}
```

### Advanced Tag Combinations

**Find CO2 sensors on ground floor:**
```graphql
query GroundFloorCO2 {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { contains: ["CO2", "ground-floor"] }  # Must have BOTH tags
    ) {
      total
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2,
        aggregation: AVG
      )
      devices {
        id
        verboseName
        tags
      }
    }
  }
}
```

**Find environmental sensors (multiple types):**
```graphql
query EnvironmentalSensors {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { overlap: ["CO2", "temperature", "humidity"] }  # Any environmental sensor
    ) {
      total
      devices {
        id
        verboseName
        tags
      }
    }
  }
}
```

### Search String Filtering

While tags are recommended for grouping, you can also use search strings to filter by device names:

```graphql
query EntranceDevices {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      search: "Entrance"  # Matches device names containing "Entrance"
    ) {
      total
      devices {
        id
        verboseName
      }
    }
  }
}
```

### Combining Tags, Search, and Semantics

**Powerful example combining all filtering options:**
```graphql
query ComplexFiltering {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { contains: ["CO2"] }      # Only CO2 sensors
      search: "Entrance"               # Names containing "Entrance"
      temperature: { gt: 20.0 }        # Temperature above 20°C
      co2: { lt: 1000.0 }             # CO2 below 1000 ppm
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE,
        aggregation: AVG
      )
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2,
        aggregation: AVG
      )
      devices {
        id
        verboseName
        tags
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
        co2: numericSemanticField(semantic: CO2) {
          value
        }
      }
    }
  }
}
```

### Recommended Tagging Strategies

#### 1. **Hierarchical Location Tags**
```
Building: ["building-A", "building-B"]
Floor: ["floor-1", "floor-2", "ground-floor"]
Room: ["room-101", "room-102", "conference-room"]
Zone: ["entrance", "exit", "lobby"]
```

#### 2. **Device Type Tags**
```
Sensor Type: ["CO2", "temperature", "humidity", "motion"]
Device Category: ["environmental", "security", "energy"]
Brand: ["elsys", "milesight", "dragino"]
```

#### 3. **Functional Tags**
```
Purpose: ["monitoring", "alerting", "control"]
Criticality: ["critical", "important", "optional"]
Maintenance: ["scheduled", "needs-attention"]
```

### Real-World Examples

#### Building Management System
```graphql
query BuildingFloorKPIs {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # Ground floor environmental data
    groundFloor: devicesFiltered(
      tags: { contains: ["ground-floor", "environmental"] }
    ) {
      total
      avgTemp: aggregatedNumericSemanticValue(semantic: TEMPERATURE, aggregation: AVG)
      avgHumidity: aggregatedNumericSemanticValue(semantic: HUMIDITY, aggregation: AVG)
    }
    
    # First floor environmental data  
    firstFloor: devicesFiltered(
      tags: { contains: ["floor-1", "environmental"] }
    ) {
      total
      avgTemp: aggregatedNumericSemanticValue(semantic: TEMPERATURE, aggregation: AVG)
      avgHumidity: aggregatedNumericSemanticValue(semantic: HUMIDITY, aggregation: AVG)
    }
  }
}
```

#### Room-by-Room Analysis
```graphql
query RoomAnalysis {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # Conference rooms
    conferenceRooms: devicesFiltered(
      tags: { contains: ["conference-room"] }
      co2: { gt: 0 }  # Only rooms with CO2 data
    ) {
      total
      avgCO2: aggregatedNumericSemanticValue(semantic: CO2, aggregation: AVG)
      maxCO2: aggregatedNumericSemanticValue(semantic: CO2, aggregation: MAX)
    }
    
    # Office spaces
    offices: devicesFiltered(
      tags: { contains: ["office"] }
      co2: { gt: 0 }
    ) {
      total
      avgCO2: aggregatedNumericSemanticValue(semantic: CO2, aggregation: AVG)
    }
  }
}
```

#### Device Health Monitoring
```graphql
query DeviceHealthByType {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # CO2 sensor health
    co2Sensors: devicesFiltered(
      tags: { contains: ["CO2"] }
      online: true
    ) {
      total
      avgBattery: aggregatedNumericSemanticValue(semantic: BATTERY, aggregation: AVG)
      minBattery: aggregatedNumericSemanticValue(semantic: BATTERY, aggregation: MIN)
    }
    
    # Temperature sensor health
    tempSensors: devicesFiltered(
      tags: { contains: ["temperature"] }
      online: true
    ) {
      total
      avgSignal: aggregatedNumericSemanticValue(semantic: SIGNAL, aggregation: AVG)
    }
  }
}
```

### Best Practices for Device Organization

#### 1. **Use Tags for Grouping (Not Search)**
```graphql
# Good: Using tags for grouping
devicesFiltered(tags: { contains: ["floor-1", "CO2"] })

# Avoid: Using search for grouping (less reliable)
devicesFiltered(search: "Floor 1 CO2")
```

#### 2. **Consistent Tagging Convention**
```
# Good: Consistent naming
["floor-1", "floor-2", "ground-floor"]
["room-101", "room-102", "room-103"]

# Avoid: Inconsistent naming  
["Floor1", "floor_2", "Ground Level"]
```

#### 3. **Multiple Filter Levels**
```graphql
# Good: Multiple levels of organization
devicesFiltered(
  tags: { contains: ["building-A", "floor-2", "environmental"] }
  temperature: { gt: 18.0 }
)
```

#### 4. **Performance Considerations**
```graphql
# Fast: Specific tag filtering
devicesFiltered(tags: { contains: ["CO2"] })

# Slower: Broad search strings
devicesFiltered(search: "sensor")
```

## Aggregating Semantic Values

### Workspace-Level Aggregation

Get aggregated values across all filtered devices:

```graphql
query GetAggregatedSemanticValues {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      all: true  # Include all devices
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE,
        aggregation: AVG
      )
      maxTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE,
        aggregation: MAX
      )
      avgHumidity: aggregatedNumericSemanticValue(
        semantic: HUMIDITY,
        aggregation: AVG
      )
      totalPower: aggregatedNumericSemanticValue(
        semantic: POWER,
        aggregation: SUM
      )
    }
  }
}
```

### Filter Aggregation vs Result Aggregation

**Important**: The `aggregation` parameter in filters and `aggregatedNumericSemanticValue` serve different purposes:

- **Filter aggregation**: How to combine multiple fields of the same semantic within each device for filtering
- **Result aggregation**: How to combine values across all filtered devices for the final result

```graphql
query FilterAndAggregateTemperature {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { 
        gt: 10.0,
        aggregation: AVG  # How to combine multiple temp fields per device
      }
    ) {
      total
      temperature_avg: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE, 
        aggregation: AVG  # How to combine across all filtered devices
      )
      devices {
        id
        verboseName
        numericSemanticField(semantic: TEMPERATURE) {
          value
        }
      }
    }
  }
}
```

**Example Response:**
```json
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 6,
        "temperature_avg": 23.101666666666663,
        "devices": [
          {
            "id": "d01eb653-aea2-4d79-a1fb-2faaab90e8d2",
            "verboseName": "CO2 - Entrance",
            "numericSemanticField": {
              "value": 23.7
            }
          },
          {
            "id": "0ac70e3f-e8a3-4379-b632-115359922f40",
            "verboseName": "CO2 - Large Room", 
            "numericSemanticField": {
              "value": 23.2
            }
          }
        ]
      }
    }
  }
}
```

### Performance Optimization: Aggregation-Only Queries

For better performance when you only need aggregated values, omit the `devices` field:

```graphql
query FastAggregationQuery {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { 
        gt: 10.0,
        aggregation: AVG
      }
    ) {
      total
      temperature_avg: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE, 
        aggregation: AVG
      )
      # No devices field = faster query
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 6,
        "temperature_avg": 23.101666666666663
      }
    }
  }
}
```

### Device-Level Semantic Fields

Get semantic field details for individual devices:

```graphql
query GetDeviceSemanticFields {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(all: true) {
      devices {
        id
        verboseName
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
          fields {
            fieldName
            verboseFieldName
            value
            unit
            fieldType
          }
        }
        humidity: numericSemanticField(semantic: HUMIDITY) {
          value
          fields {
            fieldName
            verboseFieldName
            value
            unit
          }
        }
      }
    }
  }
}
```

## Practical Working Examples

### Quick Start: Basic Temperature Filtering

**Query:**
```graphql
query BasicTemperatureFilter {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { gt: 25.0 }
    ) {
      total
      devices {
        id
        verboseName
      }
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 1,
        "devices": [
          {
            "id": "faf34567-5f13-4536-b3f3-a4a1e245ab2a",
            "verboseName": "Server Temperature"
          }
        ]
      }
    }
  }
}
```

### Adding Aggregated Values

**Query:**
```graphql
query TemperatureWithAggregation {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { 
        gt: 25.0,
        aggregation: MAX
      }
    ) {
      total
      temperature_max: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE, 
        aggregation: MAX
      )
      devices {
        id
        verboseName
        numericSemanticField(semantic: TEMPERATURE) {
          value
        }
      }
    }
  }
}
```

### Combining Tags with Semantics

**Query:**
```graphql
query CO2SensorsAtEntrance {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      tags: { contains: ["CO2"] }       # Only CO2 sensors
      search: "Entrance"                # Names containing "Entrance"
      temperature: { gt: 10.0 }         # With temperature data
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE,
        aggregation: AVG
      )
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2,
        aggregation: AVG
      )
      devices {
        id
        verboseName
        tags
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
        co2: numericSemanticField(semantic: CO2) {
          value
        }
      }
    }
  }
}
```

### Performance-Optimized Aggregation

**Query:**
```graphql
query FastTemperatureAverage {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    devicesFiltered(
      temperature: { 
        gt: 10.0,
        aggregation: AVG
      }
    ) {
      total
      temperature_avg: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE, 
        aggregation: AVG
      )
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 6,
        "temperature_avg": 23.101666666666663
      }
    }
  }
}
```

## Complete Query Examples

### Example 1: Environmental Monitoring Dashboard

```graphql
query EnvironmentalDashboard {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # Get all devices with recent temperature readings
    activeDevices: devicesFiltered(
      temperature: { gt: -50.0 }  # Exclude obviously invalid readings
      online: true
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE,
        aggregation: AVG
      )
      avgHumidity: aggregatedNumericSemanticValue(
        semantic: HUMIDITY,
        aggregation: AVG
      )
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2,
        aggregation: AVG
      )
      devices {
        id
        verboseName
        location
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
        humidity: numericSemanticField(semantic: HUMIDITY) {
          value
        }
        co2: numericSemanticField(semantic: CO2) {
          value
        }
      }
    }
  }
}
```

### Example 2: Alert Conditions

```graphql
query HighTemperatureAlert {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # Find devices with high temperature
    hotDevices: devicesFiltered(
      temperature: { gt: 35.0 }
      online: true
    ) {
      total
      maxTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE,
        aggregation: MAX
      )
      devices {
        id
        verboseName
        location
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
          fields {
            verboseFieldName
            value
            unit
          }
        }
      }
    }
    
    # Find devices with low battery
    lowBatteryDevices: devicesFiltered(
      battery: { lt: 20.0 }
    ) {
      total
      devices {
        id
        verboseName
        battery: numericSemanticField(semantic: BATTERY) {
          value
        }
      }
    }
  }
}
```

### Example 3: Energy Management

```graphql
query EnergyConsumptionAnalysis {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # High energy consumers
    highConsumers: devicesFiltered(
      energyConsumption: { gt: 100.0 }
    ) {
      total
      totalConsumption: aggregatedNumericSemanticValue(
        semantic: ENERGY_CONSUMPTION,
        aggregation: SUM
      )
    }
    
    # All devices with power data
    powerDevices: devicesFiltered(
      power: { gt: 0.0 }
    ) {
      total
      avgPower: aggregatedNumericSemanticValue(
        semantic: POWER,
        aggregation: AVG
      )
      totalPower: aggregatedNumericSemanticValue(
        semantic: POWER,
        aggregation: SUM
      )
      devices {
        id
        verboseName
        power: numericSemanticField(semantic: POWER) {
          value
        }
        energyConsumption: numericSemanticField(semantic: ENERGY_CONSUMPTION) {
          value
        }
      }
    }
  }
}
```

### Example 4: Air Quality Monitoring

```graphql
query AirQualityReport {
  workspace(id: "f6331019-8978-4a86-b1bc-3522546f67d5") {
    # Poor air quality locations
    poorAirQuality: devicesFiltered(
      airPollution: { gt: 50.0 }
      co2: { gt: 1000.0 }
    ) {
      total
      maxAirPollution: aggregatedNumericSemanticValue(
        semantic: AIR_POLLUTION,
        aggregation: MAX
      )
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2,
        aggregation: AVG
      )
    }
    
    # All air quality devices
    airQualityDevices: devicesFiltered(
      all: true
    ) {
      devices {
        id
        verboseName
        location
        airPollution: numericSemanticField(semantic: AIR_POLLUTION) {
          value
          fields {
            fieldName
            verboseFieldName
            value
            unit
          }
        }
        co2: numericSemanticField(semantic: CO2) {
          value
        }
        voc: numericSemanticField(semantic: VOC) {
          value
        }
      }
    }
  }
}
```

## Use Cases

### 1. **Multi-Device Dashboards**
Create unified dashboards showing data from different device types using semantic classification.

### 2. **Fleet Monitoring**
Monitor large deployments by semantic types (battery levels, signal strength, etc.) regardless of device manufacturer.

### 3. **Environmental Analytics**
Aggregate environmental data (temperature, humidity, air quality) across locations and device types.

### 4. **Building Management Systems**
Use tag-based grouping to monitor KPIs per floor, room, or zone. Get averaged environmental conditions for each area of your building.

### 5. **Alerting Systems**
Set up alerts based on semantic thresholds that work across your entire device fleet, filtered by location tags.

### 6. **Energy Management**
Track power consumption and efficiency across different types of devices and installations, grouped by building areas.

### 7. **Predictive Maintenance**
Monitor device health indicators (battery, signal) to predict maintenance needs, organized by device type and location.

### 8. **Multi-Location Deployments**
Manage devices across multiple buildings, floors, and rooms using hierarchical tagging combined with semantic filtering.

### 9. **Device Type Analytics**
Compare performance metrics between different sensor types (CO2 vs temperature sensors) using tag-based grouping.

### 10. **Room-Level Insights**
Get detailed analytics for specific rooms or zones by combining location tags with semantic measurements.

## Best Practices

### 1. **Use Appropriate Aggregation**
- `AVG` for typical readings (temperature, humidity)
- `MAX` for peak detection (signal strength, pollution)
- `SUM` for cumulative values (energy consumption)
- `MIN` for minimum thresholds (battery levels)

### 2. **Filter Efficiently**
```graphql
# Good: Specific filters
devicesFiltered(
  temperature: { gte: 20.0, lte: 30.0 }
  online: true
)

# Avoid: Very broad ranges that return too many devices
devicesFiltered(
  temperature: { gt: -100.0 }
)
```

### 3. **Combine with Other Filters**
```graphql
devicesFiltered(
  temperature: { gt: 25.0 }
  online: true
  tags: { contains: ["outdoor"] }
)
```

### 4. **Request Only Needed Data**
```graphql
# Good: Specific fields
devices {
  id
  verboseName
  temperature: numericSemanticField(semantic: TEMPERATURE) {
    value
  }
}

# Avoid: Requesting all fields when not needed
devices {
  id
  verboseName
  lastHeard
  metadata
  # ... many unnecessary fields
}
```

### 5. **Performance: Omit Devices for Aggregation-Only Queries**
```graphql
# Fast: Only aggregated data
devicesFiltered(temperature: { gt: 20.0 }) {
  total
  avgTemp: aggregatedNumericSemanticValue(semantic: TEMPERATURE, aggregation: AVG)
}

# Slower: Includes all device details
devicesFiltered(temperature: { gt: 20.0 }) {
  total
  avgTemp: aggregatedNumericSemanticValue(semantic: TEMPERATURE, aggregation: AVG)
  devices {
    id
    verboseName
    # ... device fields slow down the query
  }
}
```

### 6. **Use Consistent Tagging Strategies**
```graphql
# Good: Hierarchical and consistent tags
devicesFiltered(
  tags: { contains: ["building-A", "floor-2", "environmental"] }
)

# Good: Clear device type tags
devicesFiltered(
  tags: { overlap: ["CO2", "temperature", "humidity"] }
)

# Avoid: Inconsistent or unclear tags
devicesFiltered(
  tags: { contains: ["Floor2", "temp_sensor", "BuildingA"] }
)
```

### 7. **Prefer Tags Over Search for Grouping**
```graphql
# Good: Reliable tag-based grouping
devicesFiltered(
  tags: { contains: ["entrance", "CO2"] }
  co2: { gt: 400 }
)

# Less reliable: Search-based grouping
devicesFiltered(
  search: "Entrance CO2"
  co2: { gt: 400 }
)
```

### 8. **Handle Missing Semantics**
```graphql
query SafeSemanticQuery {
  workspace(id: "YOUR_ID") {
    semantics  # Check available semantics first
    devicesFiltered(
      temperature: { gt: 20.0 }
    ) {
      total
      devices {
        id
        verboseName
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
          fields {
            fieldName
            value
            unit
          }
        }
      }
    }
  }
}
```

## Error Handling

When working with semantics, be prepared for:

1. **Missing Semantics**: Not all workspaces have all semantic types
2. **No Matching Devices**: Filters might return zero devices
3. **Null Values**: Devices might not have data for requested semantics

Always check the `total` count and handle null values appropriately in your applications. 