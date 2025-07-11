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