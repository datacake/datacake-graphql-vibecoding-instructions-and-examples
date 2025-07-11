# Working with Counters, Consumption and Meter Devices

> **Complete guide for analyzing energy, water, gas, and counting devices with cumulative meter values**  
> Learn how to calculate consumption changes, compare time periods, and build consumption analytics dashboards.

## Table of Contents

1. [Overview](#overview)
2. [Understanding Meter Devices](#understanding-meter-devices)
3. [The Change Operation](#the-change-operation)
4. [Time Period Comparisons](#time-period-comparisons)
5. [Historical Consumption Profiles](#historical-consumption-profiles)
6. [Timezone Handling](#timezone-handling)
7. [Performance Considerations](#performance-considerations)
8. [Frontend Processing](#frontend-processing)
9. [Real-World Use Cases](#real-world-use-cases)
10. [Complete Examples](#complete-examples)

---

## Overview

**Consumption and meter devices** track cumulative values that continuously increase over time. Unlike regular sensor readings that can fluctuate up and down, meter devices store **absolute totals** that only grow.

### Common Device Types

- **⚡ Energy Meters** - Track kilowatt-hours (kWh) consumed
- **💧 Water Meters** - Monitor cubic meters or gallons used
- **🔥 Gas Meters** - Measure gas consumption volume
- **🌡️ Heating Meters** - Track thermal energy consumption
- **👥 People Counters** - Count total entries/exits
- **🚗 Vehicle Counters** - Track traffic or parking occupancy
- **📦 Production Counters** - Monitor manufacturing output

### Key Characteristics

1. **Cumulative Values** - Always increasing totals
2. **Never Reset** - Values don't go back to zero (unless device is replaced)
3. **Consumption = Delta** - Usage is calculated as difference between two timestamps
4. **Time-Based Analysis** - Compare different periods for insights

---

## Understanding Meter Devices

### Meter Value vs Consumption

```
Meter Reading Timeline:
Day 1: 1000 kWh (total since installation)
Day 2: 1025 kWh (total since installation)
Day 3: 1055 kWh (total since installation)

Consumption Calculations:
Day 1-2: 1025 - 1000 = 25 kWh consumed
Day 2-3: 1055 - 1025 = 30 kWh consumed
Day 1-3: 1055 - 1000 = 55 kWh total consumed
```

### Field Name Examples

Different device types use various field naming conventions:

```javascript
// Energy/Power Meters
"ACTIVE_ENERGY_IMPORT_T1_KWH"
"TOTAL_ENERGY_KWH"
"ENERGY_CONSUMPTION"

// Water Meters  
"WATER_CONSUMPTION_M3"
"TOTAL_WATER_VOLUME"
"WATER_METER_READING"

// Gas Meters
"GAS_CONSUMPTION_M3"
"TOTAL_GAS_VOLUME"
"GAS_METER_VALUE"

// Counters
"PEOPLE_COUNT_TOTAL"
"VISITOR_COUNT"
"PRODUCTION_COUNT"
```

---

## The Change Operation

### Basic Syntax

The `change()` operation calculates the difference between meter values at two timestamps:

```graphql
currentMeasurement(fieldName: "METER_FIELD_NAME") {
  total_meter_value: value
  consumption_period: change(
    timeRangeStart: "2025-07-01T00:00:00"  # Inclusive start
    timeRangeEnd: "2025-08-01T00:00:00"    # Exclusive end (start of next period)
  )
}
```

**Time Range Convention:**
- **Start**: Inclusive timestamp (00:00:00 of the period)
- **End**: Exclusive timestamp (00:00:00 of the next period)
- **Example**: For July 2025, use `timeRangeEnd: "2025-08-01T00:00:00"` (not `"2025-07-31T23:59:59"`)

### Complete Query Structure

```graphql
query ConsumptionAnalysis($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    lastHeard
    online
    
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      # Current total meter reading
      total_meter_value: value
      
      # Today's consumption (since midnight)
      consumption_today: change(
        timeRangeStart: "2025-07-11T00:00:00"
        timeRangeEnd: "2025-07-12T00:00:00"
      )
      
      # Yesterday's consumption  
      consumption_yesterday: change(
        timeRangeStart: "2025-07-10T00:00:00"
        timeRangeEnd: "2025-07-11T00:00:00"
      )
      
      # This week (Monday to now)
      consumption_this_week: change(
        timeRangeStart: "2025-07-07T00:00:00"
        timeRangeEnd: "2025-07-14T00:00:00"
      )
      
      # Last week (Monday to Sunday)
      consumption_last_week: change(
        timeRangeStart: "2025-06-30T00:00:00"
        timeRangeEnd: "2025-07-07T00:00:00"
      )
      
      # This month
      consumption_this_month: change(
        timeRangeStart: "2025-07-01T00:00:00"
        timeRangeEnd: "2025-08-01T00:00:00"
      )
      
      # Last month
      consumption_last_month: change(
        timeRangeStart: "2025-06-01T00:00:00"
        timeRangeEnd: "2025-07-01T00:00:00"
      )
    }
  }
}
```

### Response Example

```json
{
  "data": {
    "device": {
      "id": "db8286e3-306d-4e25-96a2-f8b68f05ce35",
      "verboseName": "Main Building Energy Meter",
      "lastHeard": "2025-07-11T09:48:33.844096+00:00",
      "online": true,
      "currentMeasurement": {
        "total_meter_value": 31823.075,
        "consumption_today": 6.399,
        "consumption_yesterday": 92.264,
        "consumption_this_week": 166.746,
        "consumption_last_week": 234.123,
        "consumption_this_month": 364.409,
        "consumption_last_month": 914.303
      }
    }
  }
}
```

---

## Time Period Comparisons

### Daily Comparisons

```graphql
query DailyEnergyComparison($deviceId: String!) {
  device(deviceId: $deviceId) {
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      # Compare today vs yesterday
      today: change(
        timeRangeStart: "2025-07-11T00:00:00"
        timeRangeEnd: "2025-07-12T00:00:00"
      )
      yesterday: change(
        timeRangeStart: "2025-07-10T00:00:00"
        timeRangeEnd: "2025-07-11T00:00:00"
      )
      
      # Last 7 days individual
      day_1: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      day_2: change(timeRangeStart: "2025-07-10T00:00:00", timeRangeEnd: "2025-07-11T00:00:00")
      day_3: change(timeRangeStart: "2025-07-09T00:00:00", timeRangeEnd: "2025-07-10T00:00:00")
      day_4: change(timeRangeStart: "2025-07-08T00:00:00", timeRangeEnd: "2025-07-09T00:00:00")
      day_5: change(timeRangeStart: "2025-07-07T00:00:00", timeRangeEnd: "2025-07-08T00:00:00")
      day_6: change(timeRangeStart: "2025-07-06T00:00:00", timeRangeEnd: "2025-07-07T00:00:00")
      day_7: change(timeRangeStart: "2025-07-05T00:00:00", timeRangeEnd: "2025-07-06T00:00:00")
    }
  }
}
```

### Weekly Comparisons

```graphql
query WeeklyEnergyComparison($deviceId: String!) {
  device(deviceId: $deviceId) {
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      # Current week vs last week
      this_week: change(
        timeRangeStart: "2025-07-07T00:00:00"  # Monday
        timeRangeEnd: "2025-07-14T00:00:00"    # Next Monday
      )
      last_week: change(
        timeRangeStart: "2025-06-30T00:00:00"  # Monday
        timeRangeEnd: "2025-07-07T00:00:00"    # Next Monday
      )
      
      # Last 4 weeks
      week_1: change(timeRangeStart: "2025-07-07T00:00:00", timeRangeEnd: "2025-07-14T00:00:00")
      week_2: change(timeRangeStart: "2025-06-30T00:00:00", timeRangeEnd: "2025-07-07T00:00:00")
      week_3: change(timeRangeStart: "2025-06-23T00:00:00", timeRangeEnd: "2025-06-30T00:00:00")
      week_4: change(timeRangeStart: "2025-06-16T00:00:00", timeRangeEnd: "2025-06-23T00:00:00")
    }
  }
}
```

### Monthly Comparisons

```graphql
query MonthlyEnergyComparison($deviceId: String!) {
  device(deviceId: $deviceId) {
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      # Year-over-year comparison
      july_2025: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      july_2024: change(timeRangeStart: "2024-07-01T00:00:00", timeRangeEnd: "2024-08-01T00:00:00")
      
      # Last 12 months
      month_01: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      month_02: change(timeRangeStart: "2025-06-01T00:00:00", timeRangeEnd: "2025-07-01T00:00:00")
      month_03: change(timeRangeStart: "2025-05-01T00:00:00", timeRangeEnd: "2025-06-01T00:00:00")
      month_04: change(timeRangeStart: "2025-04-01T00:00:00", timeRangeEnd: "2025-05-01T00:00:00")
      month_05: change(timeRangeStart: "2025-03-01T00:00:00", timeRangeEnd: "2025-04-01T00:00:00")
      month_06: change(timeRangeStart: "2025-02-01T00:00:00", timeRangeEnd: "2025-03-01T00:00:00")
      month_07: change(timeRangeStart: "2025-01-01T00:00:00", timeRangeEnd: "2025-02-01T00:00:00")
      month_08: change(timeRangeStart: "2024-12-01T00:00:00", timeRangeEnd: "2025-01-01T00:00:00")
      month_09: change(timeRangeStart: "2024-11-01T00:00:00", timeRangeEnd: "2024-12-01T00:00:00")
      month_10: change(timeRangeStart: "2024-10-01T00:00:00", timeRangeEnd: "2024-11-01T00:00:00")
      month_11: change(timeRangeStart: "2024-09-01T00:00:00", timeRangeEnd: "2024-10-01T00:00:00")
      month_12: change(timeRangeStart: "2024-08-01T00:00:00", timeRangeEnd: "2024-09-01T00:00:00")
    }
  }
}
```

---

## Historical Consumption Profiles

### Hourly Profile (24-hour view)

```graphql
query HourlyConsumptionProfile($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    
    # Get 24-hour meter values to calculate hourly consumption
    daily_profile: history(
      fields: ["ACTIVE_ENERGY_IMPORT_T1_KWH"]
      timerangestart: "2025-07-11T00:00:00"
      timerangeend: "2025-07-12T00:00:00"
      resolution: "1h"
    )
  }
}
```

### Daily Profile (30-day view)

```graphql
query DailyConsumptionProfile($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    
    # Get daily meter values for the last 30 days
    monthly_profile: history(
      fields: ["ACTIVE_ENERGY_IMPORT_T1_KWH"]
      timerangestart: "2025-06-11T00:00:00"
      timerangeend: "2025-07-11T00:00:00"
      resolution: "24h"
    )
  }
}
```

### Weekly Profile (12-week view)

```graphql
query WeeklyConsumptionProfile($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    
    # Get weekly meter values for the last 12 weeks
    quarterly_profile: history(
      fields: ["ACTIVE_ENERGY_IMPORT_T1_KWH"]
      timerangestart: "2025-04-11T00:00:00"
      timerangeend: "2025-07-11T00:00:00"
      resolution: "168h"  # 7 days = 168 hours
    )
  }
}
```

### Historical Response Example

```json
{
  "data": {
    "device": {
      "id": "db8286e3-306d-4e25-96a2-f8b68f05ce35",
      "verboseName": "Main Building Energy Meter",
      "daily_profile": "[{\"time\": \"2025-06-01T00:00:00Z\", \"ACTIVE_ENERGY_IMPORT_T1_KWH\": 30525.073}, {\"time\": \"2025-06-02T00:00:00Z\", \"ACTIVE_ENERGY_IMPORT_T1_KWH\": 30541.845}, {\"time\": \"2025-06-03T00:00:00Z\", \"ACTIVE_ENERGY_IMPORT_T1_KWH\": 30560.712}]"
    }
  }
}
```

---

## Timezone Handling

### Critical Important Notes

⚠️ **All timestamps in GraphQL queries use UTC timezone**  
⚠️ **Frontend is responsible for timezone conversions**  
⚠️ **Day boundaries depend on local timezone, not UTC**  
⚠️ **Time ranges use inclusive start, exclusive end pattern** (e.g., `timeRangeEnd: "2025-07-12T00:00:00"` for July 11th)

### Timezone Conversion Examples

```javascript
// JavaScript timezone handling
function getLocalDayBoundaries(date, timezone = 'Europe/Berlin') {
  const startOfDay = new Date(date);
  startOfDay.setHours(0, 0, 0, 0);
  
  const endOfDay = new Date(date);
  endOfDay.setDate(endOfDay.getDate() + 1); // Next day
  endOfDay.setHours(0, 0, 0, 0);
  
  // Convert to UTC for GraphQL query
  return {
    start: startOfDay.toISOString(),
    end: endOfDay.toISOString()
  };
}

// Usage for German timezone (CET/CEST)
const today = getLocalDayBoundaries(new Date(), 'Europe/Berlin');
const query = `
  consumption_today: change(
    timeRangeStart: "${today.start}"
    timeRangeEnd: "${today.end}"
  )
`;
```

### Week Boundary Calculation

```javascript
function getWeekBoundaries(date, timezone = 'Europe/Berlin') {
  const startOfWeek = new Date(date);
  const day = startOfWeek.getDay();
  const diff = startOfWeek.getDate() - day + (day === 0 ? -6 : 1); // Monday
  startOfWeek.setDate(diff);
  startOfWeek.setHours(0, 0, 0, 0);
  
  const endOfWeek = new Date(startOfWeek);
  endOfWeek.setDate(startOfWeek.getDate() + 7); // Next Monday
  endOfWeek.setHours(0, 0, 0, 0);
  
  return {
    start: startOfWeek.toISOString(),
    end: endOfWeek.toISOString()
  };
}
```

### Month Boundary Calculation

```javascript
function getMonthBoundaries(date, timezone = 'Europe/Berlin') {
  const startOfMonth = new Date(date.getFullYear(), date.getMonth(), 1);
  startOfMonth.setHours(0, 0, 0, 0);
  
  const endOfMonth = new Date(date.getFullYear(), date.getMonth() + 1, 1);
  endOfMonth.setHours(0, 0, 0, 0);
  
  return {
    start: startOfMonth.toISOString(),
    end: endOfMonth.toISOString()
  };
}
```

---

## Performance Considerations

### Separate Queries for Optimal Performance

⚠️ **Important**: Decouple `change()` operations from `history()` queries for better performance.

```graphql
# ✅ GOOD: Separate fast consumption query
query ConsumptionStats($deviceId: String!) {
  device(deviceId: $deviceId) {
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      total_meter_value: value
      consumption_today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      consumption_yesterday: change(timeRangeStart: "2025-07-10T00:00:00", timeRangeEnd: "2025-07-11T00:00:00")
    }
  }
}

# ✅ GOOD: Separate history query (load asynchronously)
query ConsumptionHistory($deviceId: String!) {
  device(deviceId: $deviceId) {
    history(
      fields: ["ACTIVE_ENERGY_IMPORT_T1_KWH"]
      timerangestart: "2025-06-01T00:00:00"
      timerangeend: "2025-07-01T00:00:00"
      resolution: "24h"
    )
  }
}
```

### Performance Timing

| Operation | Response Time | Use Case |
|-----------|---------------|----------|
| **Change operations** | ~100-200ms | Real-time consumption stats |
| **Current meter value** | ~50-100ms | Live meter reading |
| **History (24h resolution)** | ~500-2000ms | Daily consumption charts |
| **History (1h resolution)** | ~1000-5000ms | Hourly consumption profiles |

### Best Practices

✅ **Do:**
- Load consumption stats first (fast)
- Load history charts asynchronously
- Use appropriate resolution for time range
- Cache consumption data for dashboard updates
- Use local timezone calculations

❌ **Don't:**
- Combine many change operations with history in one query
- Use 1-hour resolution for year-long periods
- Forget timezone conversions
- Query consumption data on every page refresh

---

## Frontend Processing

### Processing Historical Data

```javascript
// Process history JSON string into consumption deltas
function processConsumptionHistory(historyJson) {
  const data = JSON.parse(historyJson);
  const consumption = [];
  
  for (let i = 1; i < data.length; i++) {
    const current = data[i];
    const previous = data[i - 1];
    
    // Calculate consumption for this period
    const periodConsumption = current.ACTIVE_ENERGY_IMPORT_T1_KWH - previous.ACTIVE_ENERGY_IMPORT_T1_KWH;
    
    consumption.push({
      time: current.time,
      consumption: Math.max(0, periodConsumption), // Handle counter resets
      meterReading: current.ACTIVE_ENERGY_IMPORT_T1_KWH
    });
  }
  
  return consumption;
}

// Usage
const historyData = processConsumptionHistory(response.data.device.history);
console.log(historyData);
// [
//   { time: "2025-06-02T00:00:00Z", consumption: 16.77, meterReading: 30541.845 },
//   { time: "2025-06-03T00:00:00Z", consumption: 18.87, meterReading: 30560.712 },
//   ...
// ]
```

### Building Consumption Dashboard

```javascript
function buildConsumptionDashboard(consumptionData) {
  return {
    // Current status
    currentReading: consumptionData.total_meter_value,
    lastUpdate: consumptionData.lastHeard,
    
    // Today's performance
    today: {
      consumption: consumptionData.consumption_today,
      unit: 'kWh'
    },
    
    // Comparisons
    comparisons: {
      vsYesterday: {
        consumption: consumptionData.consumption_yesterday,
        change: calculatePercentageChange(
          consumptionData.consumption_today,
          consumptionData.consumption_yesterday
        )
      },
      vsLastWeek: {
        consumption: consumptionData.consumption_last_week,
        change: calculatePercentageChange(
          consumptionData.consumption_this_week,
          consumptionData.consumption_last_week
        )
      },
      vsLastMonth: {
        consumption: consumptionData.consumption_last_month,
        change: calculatePercentageChange(
          consumptionData.consumption_this_month,
          consumptionData.consumption_last_month
        )
      }
    },
    
    // Trends
    trends: {
      daily: calculateTrend([
        consumptionData.day_7,
        consumptionData.day_6,
        consumptionData.day_5,
        consumptionData.day_4,
        consumptionData.day_3,
        consumptionData.day_2,
        consumptionData.day_1
      ]),
      weekly: calculateTrend([
        consumptionData.week_4,
        consumptionData.week_3,
        consumptionData.week_2,
        consumptionData.week_1
      ])
    }
  };
}

function calculatePercentageChange(current, previous) {
  if (previous === 0) return current > 0 ? 100 : 0;
  return ((current - previous) / previous) * 100;
}

function calculateTrend(values) {
  // Simple linear regression for trend
  const n = values.length;
  const sumX = n * (n + 1) / 2;
  const sumY = values.reduce((sum, val) => sum + val, 0);
  const sumXY = values.reduce((sum, val, idx) => sum + val * (idx + 1), 0);
  const sumX2 = n * (n + 1) * (2 * n + 1) / 6;
  
  const slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
  return slope > 0 ? 'increasing' : slope < 0 ? 'decreasing' : 'stable';
}
```

### Chart Data Formatting

```javascript
// Format data for Chart.js or similar charting library
function formatChartData(historyData, type = 'consumption') {
  const processed = processConsumptionHistory(historyData);
  
  return {
    labels: processed.map(item => new Date(item.time).toLocaleDateString()),
    datasets: [{
      label: type === 'consumption' ? 'Daily Consumption (kWh)' : 'Meter Reading (kWh)',
      data: processed.map(item => 
        type === 'consumption' ? item.consumption : item.meterReading
      ),
      borderColor: type === 'consumption' ? '#3498db' : '#e74c3c',
      backgroundColor: type === 'consumption' ? '#3498db20' : '#e74c3c20',
      fill: true
    }]
  };
}
```

---

## Real-World Use Cases

### Energy Management Dashboard

```graphql
query EnergyManagementDashboard($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    online
    lastHeard
    
    # Current meter reading and consumption stats
    energyMeter: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      current_reading: value
      
      # Real-time consumption tracking
      consumption_today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      consumption_yesterday: change(timeRangeStart: "2025-07-10T00:00:00", timeRangeEnd: "2025-07-11T00:00:00")
      
      # Weekly performance
      consumption_this_week: change(timeRangeStart: "2025-07-07T00:00:00", timeRangeEnd: "2025-07-14T00:00:00")
      consumption_last_week: change(timeRangeStart: "2025-06-30T00:00:00", timeRangeEnd: "2025-07-07T00:00:00")
      
      # Monthly billing
      consumption_this_month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      consumption_last_month: change(timeRangeStart: "2025-06-01T00:00:00", timeRangeEnd: "2025-07-01T00:00:00")
      
      # Year-over-year comparison
      consumption_this_year: change(timeRangeStart: "2025-01-01T00:00:00", timeRangeEnd: "2026-01-01T00:00:00")
      consumption_last_year: change(timeRangeStart: "2024-01-01T00:00:00", timeRangeEnd: "2025-01-01T00:00:00")
    }
  }
}
```

### Water Usage Monitoring

```graphql
query WaterUsageMonitoring($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    
    waterMeter: currentMeasurement(fieldName: "WATER_CONSUMPTION_M3") {
      total_volume: value
      
      # Daily water usage
      usage_today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      usage_yesterday: change(timeRangeStart: "2025-07-10T00:00:00", timeRangeEnd: "2025-07-11T00:00:00")
      
      # Weekly patterns
      usage_this_week: change(timeRangeStart: "2025-07-07T00:00:00", timeRangeEnd: "2025-07-14T00:00:00")
      
      # Monthly billing cycle
      usage_this_month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      
      # Leak detection (comparing similar periods)
      usage_same_week_last_month: change(timeRangeStart: "2025-06-09T00:00:00", timeRangeEnd: "2025-06-16T00:00:00")
    }
  }
}
```

### People Counter Analytics

```graphql
query PeopleCounterAnalytics($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    
    peopleCounter: currentMeasurement(fieldName: "PEOPLE_COUNT_TOTAL") {
      total_count: value
      
      # Daily visitor counts
      visitors_today: change(timeRangeStart: "2025-07-11T08:00:00", timeRangeEnd: "2025-07-11T18:00:00")
      visitors_yesterday: change(timeRangeStart: "2025-07-10T08:00:00", timeRangeEnd: "2025-07-10T18:00:00")
      
      # Business hours vs off-hours
      business_hours_today: change(timeRangeStart: "2025-07-11T08:00:00", timeRangeEnd: "2025-07-11T18:00:00")
      off_hours_today: change(timeRangeStart: "2025-07-11T18:00:00", timeRangeEnd: "2025-07-12T08:00:00")
      
      # Weekly foot traffic
      visitors_this_week: change(timeRangeStart: "2025-07-07T00:00:00", timeRangeEnd: "2025-07-14T00:00:00")
      visitors_last_week: change(timeRangeStart: "2025-06-30T00:00:00", timeRangeEnd: "2025-07-07T00:00:00")
      
      # Monthly trends
      visitors_this_month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
    }
  }
}
```

### Gas Consumption Tracking

```graphql
query GasConsumptionTracking($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    
    gasMeter: currentMeasurement(fieldName: "GAS_CONSUMPTION_M3") {
      total_volume: value
      
      # Daily consumption (heating patterns)
      consumption_today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      consumption_yesterday: change(timeRangeStart: "2025-07-10T00:00:00", timeRangeEnd: "2025-07-11T00:00:00")
      
      # Seasonal comparisons
      consumption_this_month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      consumption_same_month_last_year: change(timeRangeStart: "2024-07-01T00:00:00", timeRangeEnd: "2024-08-01T00:00:00")
      
      # Heating season analysis
      consumption_winter_2025: change(timeRangeStart: "2024-12-01T00:00:00", timeRangeEnd: "2025-04-01T00:00:00")
      consumption_winter_2024: change(timeRangeStart: "2023-12-01T00:00:00", timeRangeEnd: "2024-04-01T00:00:00")
    }
  }
}
```

---

## Working with Multiple Meters and Submetering or Counters

### Overview

In real-world scenarios, you often need to aggregate consumption data across multiple meters for submetering applications:

- **🏢 Building Energy Management** - Sum consumption across floor meters, HVAC systems, and lighting circuits
- **🏭 Industrial Facilities** - Track total energy usage across production lines, equipment, and support systems
- **🏠 Smart Home/Office** - Aggregate consumption from multiple smart meters and outlets
- **👥 Multi-Location Counting** - Sum people counters across office floors, building entrances, or retail locations
- **💧 Water Distribution** - Monitor total water usage across building zones, apartments, or facilities

### Future Semantic Solution

**🔮 Coming Soon: Counter Semantics**

The current approach of batch fetching for KPI aggregation is a temporary solution. We are working on implementing **counter semantics** that will allow direct aggregation of consumption data across multiple meters, similar to how current semantics work for temperature, humidity, and other sensor values.

**Future capability (in development):**
```graphql
# This will work in the future with counter semantics
devicesFiltered(tags: { contains: ["Energy"] }) {
  totalEnergyConsumptionToday: aggregatedNumericSemanticValue(
    semantic: ENERGY_CONSUMPTION_CHANGE
    timeRangeStart: "2025-07-11T00:00:00"
    timeRangeEnd: "2025-07-12T00:00:00"
    aggregation: SUM
  )
}
```

**Until then:** Use the batch fetching approach described in this guide for reliable KPI aggregation across multiple meters.

### Key Considerations

**⚠️ Critical Understanding for KPI Aggregation:**
- **Semantics don't currently support consumption aggregation** - You must calculate sums in the frontend
- **Single page queries ≠ KPI aggregation** - A single `devicesFiltered` query only returns devices for that page
- **KPI widgets require ALL devices** - Must use asynchronous batch fetching to get complete totals
- **Pagination is for device lists** - Use for browsing/displaying individual devices
- **Batch processing is for KPIs** - Use for calculating totals across entire fleet

### ⚠️ CRITICAL: Understanding the Pagination Limitation

**This is the most important concept to understand:**

```javascript
// ❌ WRONG: This will only sum 25 devices out of 100!
const response = await client.query({
  query: `
    devicesFiltered(tags: { contains: ["Energy"] }, pageSize: 25) {
      devices {
        energyConsumption: currentMeasurement(fieldName: "ENERGY_KWH") {
          consumption_today: change(...)
        }
      }
    }
  `
});

// This only calculates the sum for 25 devices, not all 100!
const totalConsumption = response.data.devices.reduce((sum, device) => 
  sum + device.energyConsumption.consumption_today, 0
);
```

```javascript
// ✅ CORRECT: Batch fetch ALL devices for complete KPI
const manager = new LargeScaleSubmeteringManager(client, workspaceId);
const aggregatedData = await manager.loadAllMeters(["Energy"]);

// This calculates the sum for ALL devices
const totalConsumption = aggregatedData.consumption_today.total;
```

### When to Use Each Approach

| Use Case | Approach | Why |
|----------|----------|-----|
| **📊 KPI Widgets** (Total consumption, fleet averages) | **Batch fetch ALL devices** | Must include every device in calculation |
| **📋 Device Lists** (Browse/manage devices) | **Single page query** | Only showing subset of devices |
| **🔍 Search Results** (Find specific devices) | **Single page query** | User browsing through results |
| **📈 Dashboard Totals** (Sum across entire fleet) | **Batch fetch ALL devices** | Incomplete data = wrong KPIs |

### Query Strategies by Scale

**⚠️ Remember: The strategies below are for different use cases:**
- **For KPI widgets** (total consumption, fleet averages): Always use batch fetching
- **For device lists** (browsing/managing devices): Use single page queries

#### Small Scale (5-20 meters)

**Option A: Single Page Query (for device lists only)**
*⚠️ Use only for displaying/browsing devices, NOT for KPIs*

```graphql
query SmallScaleDeviceList($workspaceId: String!) {
  workspace(id: $workspaceId) {
    energyMeters: devicesFiltered(
      tags: { contains: ["Energy", "Submeter"] }
      pageSize: 20  # Shows only 20 devices for browsing
      orderBy: { verboseName: ASC }
    ) {
      total  # Total count of ALL devices (e.g., 100)
      devices {  # But only returns 20 devices for this page
        id
        verboseName
        tags
        online
        lastHeard
        
        # Get consumption for display purposes
        energyConsumption: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
          current_reading: value
          consumption_today: change(
            timeRangeStart: "2025-07-11T00:00:00"
            timeRangeEnd: "2025-07-12T00:00:00"
          )
          consumption_this_month: change(
            timeRangeStart: "2025-07-01T00:00:00"
            timeRangeEnd: "2025-08-01T00:00:00"
          )
          # add other timespans as needed
        }
      }
    }
  }
}
```

**Option B: Batch Fetching for KPIs (REQUIRED for totals)**
*✅ Use for KPI widgets showing total consumption across ALL devices*

```javascript
// For KPI widgets, always use batch fetching
async function getFleetConsumptionKPIs(workspaceId, tags = ["Energy", "Submeter"]) {
  const manager = new LargeScaleSubmeteringManager(graphqlClient, workspaceId);
  
  // This fetches ALL devices, not just one page
  const aggregatedData = await manager.loadAllMeters(tags, (progress) => {
    console.log(`Loading meters: ${progress.devicesLoaded}/${progress.totalDevices}`);
  });
  
  return {
    // These are totals across ALL devices
    totalConsumptionToday: aggregatedData.consumption_today.total,
    totalConsumptionThisMonth: aggregatedData.consumption_this_month.total,
    averageConsumptionToday: aggregatedData.consumption_today.average,
    activeMetersCount: aggregatedData.consumption_today.count
  };
}
```

#### Medium Scale (20-100 meters)
Use pagination with multiple queries:

```graphql
query MediumScaleSubmetering($workspaceId: String!, $page: Int!) {
  workspace(id: $workspaceId) {
    energyMeters: devicesFiltered(
      tags: { contains: ["Energy", "Submeter"] }
      page: $page
      pageSize: 25  # Manageable batch size
      orderBy: { verboseName: ASC }
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        # Focus on key consumption periods
        energyConsumption: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
          current_reading: value
          consumption_today: change(
            timeRangeStart: "2025-07-11T00:00:00"
            timeRangeEnd: "2025-07-12T00:00:00"
          )
          consumption_this_month: change(
            timeRangeStart: "2025-07-01T00:00:00"
            timeRangeEnd: "2025-08-01T00:00:00"
          )
          # add other timespans as needed
        }
      }
    }
  }
}
```

#### Large Scale (100+ meters)
Use asynchronous batching with smaller page sizes:

```graphql
query LargeScaleSubmetering($workspaceId: String!, $page: Int!) {
  workspace(id: $workspaceId) {
    energyMeters: devicesFiltered(
      tags: { contains: ["Energy", "Submeter"] }
      page: $page
      pageSize: 10  # Smaller batches for better performance
      orderBy: { verboseName: ASC }
    ) {
      total
      devices {
        id
        verboseName
        online
        
        # Minimal consumption data to reduce payload
        energyConsumption: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
          consumption_today: change(
            timeRangeStart: "2025-07-11T00:00:00"
            timeRangeEnd: "2025-07-12T00:00:00"
          )
          # add other timespans as needed
        }
      }
    }
  }
}
```

### Frontend Aggregation Strategies

#### Processing Multiple Meters

```javascript
// Process consumption data from multiple meters
class SubmeteringProcessor {
  constructor() {
    this.meters = [];
    this.aggregatedData = {};
  }
  
  // Add meter data from API responses
  addMeterData(metersResponse) {
    metersResponse.devices.forEach(device => {
      const meter = {
        id: device.id,
        name: device.verboseName,
        tags: device.tags,
        online: device.online,
        consumption: device.energyConsumption
      };
      
      this.meters.push(meter);
    });
  }
  
  // Calculate aggregated consumption totals
  calculateAggregatedConsumption() {
    const periods = ['consumption_today', 'consumption_yesterday', 'consumption_this_week', 'consumption_this_month'];
    
    periods.forEach(period => {
      this.aggregatedData[period] = {
        total: 0,
        count: 0,
        average: 0,
        meters: []
      };
      
      this.meters.forEach(meter => {
        const consumption = meter.consumption[period];
        if (consumption !== null && consumption !== undefined) {
          this.aggregatedData[period].total += consumption;
          this.aggregatedData[period].count += 1;
          this.aggregatedData[period].meters.push({
            id: meter.id,
            name: meter.name,
            consumption: consumption
          });
        }
      });
      
      // Calculate average
      if (this.aggregatedData[period].count > 0) {
        this.aggregatedData[period].average = 
          this.aggregatedData[period].total / this.aggregatedData[period].count;
      }
    });
  }
  
  // Get aggregated results
  getAggregatedData() {
    return this.aggregatedData;
  }
  
  // Get top consumers for a period
  getTopConsumers(period, limit = 5) {
    const periodData = this.aggregatedData[period];
    if (!periodData || !periodData.meters) return [];
    
    return periodData.meters
      .sort((a, b) => b.consumption - a.consumption)
      .slice(0, limit);
  }
  
  // Get consumption by tag groups
  getConsumptionByTags() {
    const tagGroups = {};
    
    this.meters.forEach(meter => {
      meter.tags.forEach(tag => {
        if (!tagGroups[tag]) {
          tagGroups[tag] = {
            total_today: 0,
            total_month: 0,
            meter_count: 0,
            meters: []
          };
        }
        
        tagGroups[tag].total_today += meter.consumption.consumption_today || 0;
        tagGroups[tag].total_month += meter.consumption.consumption_this_month || 0;
        tagGroups[tag].meter_count += 1;
        tagGroups[tag].meters.push(meter.id);
      });
    });
    
    return tagGroups;
  }
}
```

#### Asynchronous Batching for Large Scale

```javascript
// Handle large numbers of meters with batching
class LargeScaleSubmeteringManager {
  constructor(graphqlClient, workspaceId) {
    this.client = graphqlClient;
    this.workspaceId = workspaceId;
    this.batchSize = 10;
    this.processor = new SubmeteringProcessor();
  }
  
  // Load all meters in batches
  async loadAllMeters(tags = ["Energy", "Submeter"], onProgress = null) {
    let page = 0;
    let totalPages = 0;
    let allLoaded = false;
    
    while (!allLoaded) {
      try {
        const response = await this.loadMeterBatch(tags, page);
        const data = response.data.workspace.energyMeters;
        
        // Calculate total pages on first request
        if (page === 0) {
          totalPages = Math.ceil(data.total / this.batchSize);
        }
        
        // Process this batch
        this.processor.addMeterData(data);
        
        // Report progress
        if (onProgress) {
          onProgress({
            page: page + 1,
            totalPages: totalPages,
            devicesLoaded: this.processor.meters.length,
            totalDevices: data.total
          });
        }
        
        // Check if we've loaded all pages
        allLoaded = (page + 1) >= totalPages || data.devices.length === 0;
        page++;
        
        // Small delay between batches to avoid overwhelming the API
        await new Promise(resolve => setTimeout(resolve, 100));
        
      } catch (error) {
        console.error(`Error loading meter batch ${page}:`, error);
        throw error;
      }
    }
    
    // Calculate final aggregated data
    this.processor.calculateAggregatedConsumption();
    return this.processor.getAggregatedData();
  }
  
  // Load a single batch of meters
  async loadMeterBatch(tags, page) {
    const query = `
      query LoadMeterBatch($workspaceId: String!, $tags: [String!]!, $page: Int!) {
        workspace(id: $workspaceId) {
          energyMeters: devicesFiltered(
            tags: { contains: $tags }
            page: $page
            pageSize: ${this.batchSize}
            orderBy: { verboseName: ASC }
          ) {
            total
            devices {
              id
              verboseName
              tags
              online
              energyConsumption: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
                consumption_today: change(
                  timeRangeStart: "2025-07-11T00:00:00"
                  timeRangeEnd: "2025-07-12T00:00:00"
                )
                consumption_this_month: change(
                  timeRangeStart: "2025-07-01T00:00:00"
                  timeRangeEnd: "2025-08-01T00:00:00"
                )
              }
            }
          }
        }
      }
    `;
    
    return await this.client.query({
      query,
      variables: {
        workspaceId: this.workspaceId,
        tags,
        page
      }
    });
  }
}
```

### Complete Submetering Examples

#### Office Building Energy Dashboard

```graphql
query OfficeBuildingEnergyDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # Floor-level energy meters
    floorMeters: devicesFiltered(
      tags: { contains: ["Energy", "Floor"] }
      pageSize: 20
      orderBy: { verboseName: ASC }
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        energyMeter: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
          current_reading: value
          consumption_today: change(
            timeRangeStart: "2025-07-11T00:00:00"
            timeRangeEnd: "2025-07-12T00:00:00"
          )
          consumption_this_month: change(
            timeRangeStart: "2025-07-01T00:00:00"
            timeRangeEnd: "2025-08-01T00:00:00"
          )
        }
      }
    }
    
    # HVAC system meters
    hvacMeters: devicesFiltered(
      tags: { contains: ["Energy", "HVAC"] }
      pageSize: 10
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        energyMeter: currentMeasurement(fieldName: "ENERGY_CONSUMPTION_KWH") {
          current_reading: value
          consumption_today: change(
            timeRangeStart: "2025-07-11T00:00:00"
            timeRangeEnd: "2025-07-12T00:00:00"
          )
          consumption_this_month: change(
            timeRangeStart: "2025-07-01T00:00:00"
            timeRangeEnd: "2025-08-01T00:00:00"
          )
        }
      }
    }
    
    # Lighting circuit meters
    lightingMeters: devicesFiltered(
      tags: { contains: ["Energy", "Lighting"] }
      pageSize: 15
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        energyMeter: currentMeasurement(fieldName: "LIGHTING_ENERGY_KWH") {
          current_reading: value
          consumption_today: change(
            timeRangeStart: "2025-07-11T00:00:00"
            timeRangeEnd: "2025-07-12T00:00:00"
          )
          consumption_this_month: change(
            timeRangeStart: "2025-07-01T00:00:00"
            timeRangeEnd: "2025-08-01T00:00:00"
          )
        }
      }
    }
  }
}
```

#### Multi-Location People Counter Aggregation

```graphql
query MultiLocationPeopleCounters($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # Main building entrances
    mainEntrances: devicesFiltered(
      tags: { contains: ["People", "Main-Entrance"] }
      pageSize: 10
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        peopleCounter: currentMeasurement(fieldName: "PEOPLE_COUNT_TOTAL") {
          total_count: value
          
          # Business hours today
          visitors_today: change(
            timeRangeStart: "2025-07-11T08:00:00"
            timeRangeEnd: "2025-07-11T18:00:00"
          )
          
          # This week's traffic
          visitors_this_week: change(
            timeRangeStart: "2025-07-07T08:00:00"
            timeRangeEnd: "2025-07-14T18:00:00"
          )
          
          # Monthly totals
          visitors_this_month: change(
            timeRangeStart: "2025-07-01T08:00:00"
            timeRangeEnd: "2025-08-01T18:00:00"
          )
        }
      }
    }
    
    # Floor-level counters
    floorCounters: devicesFiltered(
      tags: { contains: ["People", "Floor"] }
      pageSize: 15
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        peopleCounter: currentMeasurement(fieldName: "PEOPLE_COUNT_TOTAL") {
          total_count: value
          visitors_today: change(
            timeRangeStart: "2025-07-11T08:00:00"
            timeRangeEnd: "2025-07-11T18:00:00"
          )
          visitors_this_week: change(
            timeRangeStart: "2025-07-07T08:00:00"
            timeRangeEnd: "2025-07-14T18:00:00"
          )
        }
      }
    }
    
    # Conference room counters
    conferenceRoomCounters: devicesFiltered(
      tags: { contains: ["People", "Conference-Room"] }
      pageSize: 20
    ) {
      total
      devices {
        id
        verboseName
        tags
        online
        
        peopleCounter: currentMeasurement(fieldName: "PEOPLE_COUNT_TOTAL") {
          total_count: value
          visitors_today: change(
            timeRangeStart: "2025-07-11T08:00:00"
            timeRangeEnd: "2025-07-11T18:00:00"
          )
        }
      }
    }
  }
}
```

### Usage Examples

#### Complete Office Building Dashboard Implementation

```javascript
// Complete implementation for office building energy management
class OfficeBuildingEnergyManager {
  constructor(graphqlClient, workspaceId) {
    this.client = graphqlClient;
    this.workspaceId = workspaceId;
    this.processor = new SubmeteringProcessor();
  }
  
  // Load all building energy data
  async loadBuildingEnergyData() {
    const response = await this.client.query({
      query: `
        query OfficeBuildingEnergyDashboard($workspaceId: String!) {
          workspace(id: $workspaceId) {
            floorMeters: devicesFiltered(
              tags: { contains: ["Energy", "Floor"] }
              pageSize: 20
            ) {
              total
              devices {
                id
                verboseName
                tags
                online
                energyMeter: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
                  current_reading: value
                  consumption_today: change(
                    timeRangeStart: "2025-07-11T00:00:00"
                    timeRangeEnd: "2025-07-12T00:00:00"
                  )
                  consumption_this_month: change(
                    timeRangeStart: "2025-07-01T00:00:00"
                    timeRangeEnd: "2025-08-01T00:00:00"
                  )
                }
              }
            }
            hvacMeters: devicesFiltered(
              tags: { contains: ["Energy", "HVAC"] }
              pageSize: 10
            ) {
              total
              devices {
                id
                verboseName
                tags
                online
                energyMeter: currentMeasurement(fieldName: "ENERGY_CONSUMPTION_KWH") {
                  current_reading: value
                  consumption_today: change(
                    timeRangeStart: "2025-07-11T00:00:00"
                    timeRangeEnd: "2025-07-12T00:00:00"
                  )
                  consumption_this_month: change(
                    timeRangeStart: "2025-07-01T00:00:00"
                    timeRangeEnd: "2025-08-01T00:00:00"
                  )
                }
              }
            }
            lightingMeters: devicesFiltered(
              tags: { contains: ["Energy", "Lighting"] }
              pageSize: 15
            ) {
              total
              devices {
                id
                verboseName
                tags
                online
                energyMeter: currentMeasurement(fieldName: "LIGHTING_ENERGY_KWH") {
                  current_reading: value
                  consumption_today: change(
                    timeRangeStart: "2025-07-11T00:00:00"
                    timeRangeEnd: "2025-07-12T00:00:00"
                  )
                  consumption_this_month: change(
                    timeRangeStart: "2025-07-01T00:00:00"
                    timeRangeEnd: "2025-08-01T00:00:00"
                  )
                }
              }
            }
          }
        }
      `,
      variables: { workspaceId: this.workspaceId }
    });
    
    // Process all meter groups
    const data = response.data.workspace;
    this.processor.addMeterData(data.floorMeters);
    this.processor.addMeterData(data.hvacMeters);
    this.processor.addMeterData(data.lightingMeters);
    
    // Calculate aggregated consumption
    this.processor.calculateAggregatedConsumption();
    
    return this.generateBuildingReport();
  }
  
  // Generate comprehensive building energy report
  generateBuildingReport() {
    const aggregated = this.processor.getAggregatedData();
    const byTags = this.processor.getConsumptionByTags();
    
    return {
      overview: {
        totalConsumptionToday: aggregated.consumption_today.total,
        totalConsumptionThisMonth: aggregated.consumption_this_month.total,
        totalActiveMeters: aggregated.consumption_today.count,
        averageConsumptionToday: aggregated.consumption_today.average
      },
      
      bySystem: {
        floors: {
          totalToday: byTags['Floor']?.total_today || 0,
          totalMonth: byTags['Floor']?.total_month || 0,
          meterCount: byTags['Floor']?.meter_count || 0
        },
        hvac: {
          totalToday: byTags['HVAC']?.total_today || 0,
          totalMonth: byTags['HVAC']?.total_month || 0,
          meterCount: byTags['HVAC']?.meter_count || 0
        },
        lighting: {
          totalToday: byTags['Lighting']?.total_today || 0,
          totalMonth: byTags['Lighting']?.total_month || 0,
          meterCount: byTags['Lighting']?.meter_count || 0
        }
      },
      
      topConsumers: {
        today: this.processor.getTopConsumers('consumption_today', 10),
        thisMonth: this.processor.getTopConsumers('consumption_this_month', 10)
      },
      
      efficiency: {
        consumptionPerMeter: aggregated.consumption_today.total / aggregated.consumption_today.count,
        monthlyGrowth: this.calculateMonthlyGrowth(aggregated)
      }
    };
  }
  
  // Calculate monthly growth trend
  calculateMonthlyGrowth(aggregated) {
    const todayConsumption = aggregated.consumption_today.total;
    const monthlyConsumption = aggregated.consumption_this_month.total;
    
    // Estimate daily average for the month
    const currentDay = new Date().getDate();
    const monthlyAverage = monthlyConsumption / currentDay;
    
    return {
      todayVsMonthlyAverage: ((todayConsumption - monthlyAverage) / monthlyAverage) * 100,
      projectedMonthlyTotal: monthlyAverage * 31
    };
  }
}

// Usage example
async function displayBuildingEnergyDashboard() {
  const manager = new OfficeBuildingEnergyManager(graphqlClient, workspaceId);
  
  try {
    const report = await manager.loadBuildingEnergyData();
    
    console.log('Building Energy Report:');
    console.log('======================');
    console.log(`Total Energy Today: ${report.overview.totalConsumptionToday.toFixed(2)} kWh`);
    console.log(`Total Energy This Month: ${report.overview.totalConsumptionThisMonth.toFixed(2)} kWh`);
    console.log(`Active Meters: ${report.overview.totalActiveMeters}`);
    console.log('');
    
    console.log('By System:');
    console.log(`- Floors: ${report.bySystem.floors.totalToday.toFixed(2)} kWh today`);
    console.log(`- HVAC: ${report.bySystem.hvac.totalToday.toFixed(2)} kWh today`);
    console.log(`- Lighting: ${report.bySystem.lighting.totalToday.toFixed(2)} kWh today`);
    console.log('');
    
    console.log('Top Consumers Today:');
    report.topConsumers.today.forEach((consumer, index) => {
      console.log(`${index + 1}. ${consumer.name}: ${consumer.consumption.toFixed(2)} kWh`);
    });
    
  } catch (error) {
    console.error('Error loading building energy data:', error);
  }
}
```

### Real-World Scenario: Building the Right Dashboard

**Scenario**: You have 100 energy meters and need to build a dashboard with:
1. **KPI widget** showing "Total Energy Consumption Today: 1,250 kWh"
2. **Device list** showing 25 meters per page for management

#### ❌ Common Mistake (Wrong Approach)

```javascript
// This is WRONG - will only show consumption for 25 meters!
async function buildDashboard() {
  const response = await client.query({
    query: `
      devicesFiltered(tags: { contains: ["Energy"] }, pageSize: 25) {
        total  # Returns 100 (correct total count)
        devices {  # But only returns 25 devices!
          energyConsumption: currentMeasurement(fieldName: "ENERGY_KWH") {
            consumption_today: change(...)
          }
        }
      }
    `
  });
  
  // This calculation is WRONG - only sums 25 devices out of 100!
  const totalConsumption = response.data.devices.reduce((sum, device) => 
    sum + device.energyConsumption.consumption_today, 0
  );
  
  return {
    kpiWidget: `Total Energy Today: ${totalConsumption} kWh`,  // WRONG: Shows ~250 kWh instead of 1,250 kWh
    deviceList: response.data.devices  // Correct: Shows 25 devices for browsing
  };
}
```

#### ✅ Correct Approach

```javascript
async function buildDashboard() {
  // 1. KPI Widget - Batch fetch ALL devices
  const kpiManager = new LargeScaleSubmeteringManager(client, workspaceId);
  const aggregatedData = await kpiManager.loadAllMeters(["Energy"]);
  
  // 2. Device List - Single page query 
  const deviceListResponse = await client.query({
    query: `
      devicesFiltered(tags: { contains: ["Energy"] }, pageSize: 25, page: 0) {
        total
        devices {
          id
          verboseName
          online
          energyConsumption: currentMeasurement(fieldName: "ENERGY_KWH") {
            consumption_today: change(...)
          }
        }
      }
    `
  });
  
  return {
    kpiWidget: `Total Energy Today: ${aggregatedData.consumption_today.total} kWh`,  // CORRECT: Shows 1,250 kWh
    deviceList: deviceListResponse.data.devices,  // CORRECT: Shows 25 devices for browsing
    totalDevices: deviceListResponse.data.total   // CORRECT: Shows 100 total devices
  };
}
```

### Performance Best Practices for Submetering

#### Query Optimization

✅ **Do:**
- **Use batch fetching for KPIs** - Must include ALL devices in totals
- **Use single page queries for device lists** - Only showing subset of devices
- Batch large meter sets into manageable chunks (10-25 devices per query)
- Cache aggregated results for dashboard updates
- Use specific tags to filter relevant meters only
- Process data asynchronously to avoid blocking the UI

❌ **Don't:**
- **Use single page queries for KPIs** - Will give incorrect totals
- Query all meters without pagination (for device lists)
- Mix consumption queries with history queries
- Calculate aggregations in real-time without caching
- Ignore error handling for individual meter failures
- Query more time periods than needed

#### Frontend Processing

✅ **Do:**
- **Clearly separate KPI logic from device list logic** - Use different query strategies
- Validate consumption values (handle nulls and counter resets)
- Aggregate data in organized classes/modules
- Provide progress indicators for large batch operations
- Cache processed results for subsequent views
- Handle offline meters gracefully

❌ **Don't:**
- **Confuse device list pagination with KPI aggregation** - They serve different purposes
- Assume all meters have valid consumption data
- Process large datasets on the main thread
- Recalculate aggregations on every UI update
- Ignore meter online/offline status
- Skip error handling for individual meter processing

---

## Complete Examples

### Multi-Device Energy Dashboard

```graphql
query MultiDeviceEnergyDashboard($mainMeterId: String!, $hvacMeterId: String!, $lightingMeterId: String!) {
  # Main building meter
  mainBuilding: device(deviceId: $mainMeterId) {
    id
    verboseName
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      total: value
      today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
    }
  }
  
  # HVAC system meter
  hvacSystem: device(deviceId: $hvacMeterId) {
    id
    verboseName
    currentMeasurement(fieldName: "ENERGY_CONSUMPTION_KWH") {
      total: value
      today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
    }
  }
  
  # Lighting circuit meter
  lightingCircuit: device(deviceId: $lightingMeterId) {
    id
    verboseName
    currentMeasurement(fieldName: "TOTAL_ENERGY_KWH") {
      total: value
      today: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      month: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
    }
  }
}
```

### Comprehensive Utility Monitoring

```graphql
query UtilityMonitoringDashboard(
  $electricityMeterId: String!
  $waterMeterId: String!
  $gasMeterId: String!
) {
  electricity: device(deviceId: $electricityMeterId) {
    id
    verboseName
    currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      currentReading: value
      consumptionToday: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      consumptionThisMonth: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      consumptionLastMonth: change(timeRangeStart: "2025-06-01T00:00:00", timeRangeEnd: "2025-07-01T00:00:00")
    }
  }
  
  water: device(deviceId: $waterMeterId) {
    id
    verboseName
    currentMeasurement(fieldName: "WATER_CONSUMPTION_M3") {
      currentReading: value
      consumptionToday: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      consumptionThisMonth: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      consumptionLastMonth: change(timeRangeStart: "2025-06-01T00:00:00", timeRangeEnd: "2025-07-01T00:00:00")
    }
  }
  
  gas: device(deviceId: $gasMeterId) {
    id
    verboseName
    currentMeasurement(fieldName: "GAS_CONSUMPTION_M3") {
      currentReading: value
      consumptionToday: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      consumptionThisMonth: change(timeRangeStart: "2025-07-01T00:00:00", timeRangeEnd: "2025-08-01T00:00:00")
      consumptionLastMonth: change(timeRangeStart: "2025-06-01T00:00:00", timeRangeEnd: "2025-07-01T00:00:00")
    }
  }
}
```

### Detailed Consumption Analysis

```graphql
query DetailedConsumptionAnalysis($deviceId: String!) {
  device(deviceId: $deviceId) {
    id
    verboseName
    online
    lastHeard
    
    # Current meter state
    meter: currentMeasurement(fieldName: "ACTIVE_ENERGY_IMPORT_T1_KWH") {
      currentReading: value
      
      # Hourly breakdown for today
      hour_00_01: change(timeRangeStart: "2025-07-11T00:00:00", timeRangeEnd: "2025-07-11T01:00:00")
      hour_01_02: change(timeRangeStart: "2025-07-11T01:00:00", timeRangeEnd: "2025-07-11T02:00:00")
      hour_02_03: change(timeRangeStart: "2025-07-11T02:00:00", timeRangeEnd: "2025-07-11T03:00:00")
      hour_03_04: change(timeRangeStart: "2025-07-11T03:00:00", timeRangeEnd: "2025-07-11T04:00:00")
      hour_04_05: change(timeRangeStart: "2025-07-11T04:00:00", timeRangeEnd: "2025-07-11T05:00:00")
      hour_05_06: change(timeRangeStart: "2025-07-11T05:00:00", timeRangeEnd: "2025-07-11T06:00:00")
      hour_06_07: change(timeRangeStart: "2025-07-11T06:00:00", timeRangeEnd: "2025-07-11T07:00:00")
      hour_07_08: change(timeRangeStart: "2025-07-11T07:00:00", timeRangeEnd: "2025-07-11T08:00:00")
      hour_08_09: change(timeRangeStart: "2025-07-11T08:00:00", timeRangeEnd: "2025-07-11T09:00:00")
      hour_09_10: change(timeRangeStart: "2025-07-11T09:00:00", timeRangeEnd: "2025-07-11T10:00:00")
      
      # Peak vs off-peak consumption
      peak_hours: change(timeRangeStart: "2025-07-11T08:00:00", timeRangeEnd: "2025-07-11T20:00:00")
      off_peak_hours: change(timeRangeStart: "2025-07-11T20:00:00", timeRangeEnd: "2025-07-12T08:00:00")
      
      # Business patterns
      business_days_this_week: change(timeRangeStart: "2025-07-07T00:00:00", timeRangeEnd: "2025-07-12T00:00:00")
      weekend_this_week: change(timeRangeStart: "2025-07-12T00:00:00", timeRangeEnd: "2025-07-14T00:00:00")
    }
  }
}
```

---

## Key Takeaways

### Essential Concepts

1. **Meter Values are Cumulative** - Always increasing totals
2. **Consumption = Delta** - Use `change()` operation for usage calculations
3. **Timezone Matters** - Convert local time boundaries to UTC
4. **Performance Optimization** - Separate fast stats from slow history queries
5. **Frontend Responsibility** - Calculate consumption deltas from historical data

### Best Practices Summary

✅ **Always Do:**
- Use `change()` operation for consumption calculations
- Handle timezone conversions properly
- Separate fast consumption queries from history queries
- Cache consumption data for dashboard performance
- Validate consumption values (handle counter resets)

❌ **Never Do:**
- Combine many change operations with history in one query
- Forget timezone boundaries for daily/weekly/monthly periods
- Use raw meter values as consumption (always calculate deltas)
- Query history data for real-time consumption stats
- Ignore counter resets or negative consumption values

This guide provides all the tools needed to build comprehensive consumption monitoring and analytics applications with Datacake's meter device capabilities. 