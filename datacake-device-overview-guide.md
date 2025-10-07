# Building Device Overviews with Datacake API

> **Complete guide for querying devices and building custom application interfaces**  
> Learn when to use different query methods and how to build efficient, hardcoded frontends for specific use cases.

## Table of Contents

1. [Overview](#overview)
2. [Best Practices: Choosing the Right Approach](#best-practices-choosing-the-right-approach)
3. [Query Methods Comparison](#query-methods-comparison)
4. [Small Workspace Approach: allDevices](#small-workspace-approach-alldevices)
5. [Large Workspace Approach: devicesFiltered](#large-workspace-approach-devicesfiltered)
6. [Product-Based Architecture](#product-based-architecture)
7. [Hardcoded Field Strategy](#hardcoded-field-strategy)
8. [Role Fields for Dynamic Display](#role-fields-for-dynamic-display)
9. [Dynamic Multi-Product Queries with Role Fields](#dynamic-multi-product-queries-with-role-fields)
10. [Dynamic Semantic Queries for KPIs and Cross-Device Compatibility](#dynamic-semantic-queries-for-kpis-and-cross-device-compatibility)
11. [Tag Organization Strategies](#tag-organization-strategies)
12. [Device Folder Structure](#device-folder-structure)
13. [Performance Considerations](#performance-considerations)
14. [Key Performance Patterns](#key-performance-patterns)
15. [Complete Examples](#complete-examples)

---

## Overview

When building custom applications on top of Datacake, you'll typically need to create device overview interfaces. The key decision is choosing the right query method based on your workspace size and use case requirements.

### Two Primary Approaches:

1. **`allDevices`** - For small workspaces with dedicated use cases
2. **`devicesFiltered`** - For larger workspaces requiring pagination and advanced filtering

### Key Concepts:

- **Products**: Devices of the same type with identical field structures
- **Hardcoded Fields**: Pre-defined field identifiers for reliable, fast queries
- **Tag Organization**: Grouping devices for easy filtering and management
- **Role Fields**: User-configured dynamic field assignments

---

## Best Practices: Choosing the Right Approach

### Decision Framework

**The key question: What type of application are you building?**

#### 1. **Single Product, Stable Fields → Use Hardcoded Fields**
✅ **Perfect for:**
- Dedicated applications for one product type
- Stable field structures that rarely change
- Maximum performance requirements
- Specific use cases (e.g., soil moisture monitoring app)

```javascript
// Example: Hardcoded soil sensor fields
const SOIL_FIELDS = {
  MOISTURE: 'SOIL_MOISTURE',
  TEMPERATURE: 'SOIL_TEMPERATURE',
  BATTERY: 'BATTERY'
};
```

#### 2. **Multi-Product Environments → Use Role Fields**
✅ **Perfect for:**
- Manufacturer platforms with multiple product types
- LoRaWAN deployments with different vendors
- Standardized display across diverse devices
- User-configurable device interfaces

```graphql
# Works across any product type
roleFields {
  role  # PRIMARY, SECONDARY, DEVICE_BATTERY
  value
  field { fieldName, unit }
}
```

#### 3. **KPIs and Analytics → ALWAYS Use Semantics**
✅ **Critical for:**
- **Dashboard KPIs and metrics**
- **Cross-device analytics**
- **Dynamic environments with changing products**
- **Any aggregated calculations**

⚠️ **Why semantics are mandatory for KPIs:**
- **Platform calculates** - No frontend math required
- **Instant results** - No need to fetch hundreds of devices
- **Avoids pagination complexity** - KPIs work across entire workspace
- **Performance** - One query vs. fetching 300+ devices

```graphql
# ✅ CORRECT: Platform calculates KPI instantly
devicesFiltered(all: true) {
  avgTemperature: aggregatedNumericSemanticValue(
    semantic: TEMPERATURE
    aggregation: AVG
  )
}

# ❌ WRONG: Fetching devices to calculate frontend KPIs
devicesFiltered(all: true) {
  devices {  # Could return 1000s of devices!
    temperature: currentMeasurements(fieldNames: ["TEMP"]) { value }
  }
}
```

### **The KPI Rule: Semantics Only**

**For any dashboard metrics, overview statistics, or aggregated values:**

| ✅ **Always Use** | ❌ **Never Use** |
|------------------|------------------|
| `aggregatedNumericSemanticValue()` | Fetch all devices + calculate |
| Semantic filtering | Hardcoded field fetching |
| Platform aggregation | Frontend math |

**Why this matters:**
```graphql
# ✅ FAST: One query, instant result
avgTemperature: aggregatedNumericSemanticValue(semantic: TEMPERATURE)

# ❌ SLOW: Fetch 500 devices, calculate average in frontend
devices {  # 500 API calls, massive payload, slow calculation
  currentMeasurements(fieldNames: ["TEMPERATURE"]) { value }
}
```

### **Decision Matrix**

| Need | Field Approach | Query Method | Performance |
|------|---------------|--------------|-------------|
| **Single product app** | Hardcoded | `currentMeasurements(fieldNames: [...])` | ⚡ Fastest |
| **Multi-product UI** | Role Fields | `roleFields { role, value }` | 🚀 Fast |
| **Dashboard KPIs** | **Semantics** | `aggregatedNumericSemanticValue()` | ⚡ Instant |
| **Cross-device analytics** | **Semantics** | `numericSemanticField(semantic: ...)` | 💪 Good |
| **Dynamic environments** | **Semantics** | Both semantic methods | 💪 Flexible |

### **Real-World Examples**

#### Dedicated Soil Moisture App
```javascript
// ✅ GOOD: Hardcoded for single product
const query = `
  currentMeasurements(fieldNames: ["SOIL_MOISTURE", "SOIL_TEMPERATURE"]) {
    value
    field { fieldName, unit }
  }
`;
```

#### Multi-Vendor LoRaWAN Platform
```javascript
// ✅ GOOD: Role fields for standardized display
const query = `
  roleFields {
    role      # PRIMARY, SECONDARY, DEVICE_BATTERY
    value
    field { fieldName, unit }
  }
`;
```

#### Dashboard with KPIs
```javascript
// ✅ GOOD: Semantics for instant KPIs
const query = `
  devicesFiltered(all: true) {
    avgTemperature: aggregatedNumericSemanticValue(semantic: TEMPERATURE)
    avgHumidity: aggregatedNumericSemanticValue(semantic: HUMIDITY)
    totalDevices: total
  }
`;
```

#### Environmental Monitoring Dashboard
```javascript
// ✅ PERFECT: Combined approach
const query = `
  workspace(id: $workspaceId) {
    # 1. SEMANTIC KPIs - Instant metrics
    kpis: devicesFiltered(all: true) {
      avgTemp: aggregatedNumericSemanticValue(semantic: TEMPERATURE)
      maxCO2: aggregatedNumericSemanticValue(semantic: CO2, aggregation: MAX)
    }
    
    # 2. ROLE FIELDS - Multi-product device list
    devices: devicesFiltered(pageSize: 20) {
      devices {
        roleFields { role, value, field { fieldName, unit } }
      }
    }
    
    # 3. HARDCODED - Specific product details
    soilSensors: devicesFiltered(tags: { contains: ["Soil"] }) {
      devices {
        currentMeasurements(fieldNames: ["SOIL_MOISTURE"]) { value }
      }
    }
  }
`;
```

### **Performance Impact**

| Approach | Query Time | Payload Size | Frontend Work |
|----------|------------|--------------|---------------|
| **Semantic KPIs** | ~50ms | Tiny (just numbers) | None |
| **Role Fields** | ~200ms | Medium | Light processing |
| **Hardcoded Fields** | ~150ms | Small | None |
| **❌ Device Fetching for KPIs** | ~5000ms | Huge (1000s devices) | Heavy calculation |

### **Common Mistakes to Avoid**

❌ **Don't do this:**
```graphql
# Fetching all devices to calculate averages
devicesFiltered(all: true) {
  devices {  # Returns 500+ devices
    temperature: currentMeasurements(fieldNames: ["TEMP"]) { value }
  }
}
# Then calculating average in frontend = SLOW
```

✅ **Do this instead:**
```graphql
# Let the platform calculate instantly
devicesFiltered(all: true) {
  avgTemperature: aggregatedNumericSemanticValue(semantic: TEMPERATURE)
}
```

❌ **Don't mix approaches unnecessarily:**
```graphql
# Mixing hardcoded + semantics in same device query
devices {
  currentMeasurements(fieldNames: ["SOIL_TEMP"]) { value }
  temperature: numericSemanticField(semantic: TEMPERATURE) { value }
}
```

✅ **Choose one approach per use case:**
```graphql
# Either hardcoded for specific products
currentMeasurements(fieldNames: ["SOIL_TEMP"]) { value }

# OR semantic for cross-device compatibility
temperature: numericSemanticField(semantic: TEMPERATURE) { value }
```

---

## Query Methods Comparison

| Feature | `allDevices` | `devicesFiltered` |
|---------|--------------|-------------------|
| **Best for** | Small workspaces (<50 devices) | Large workspaces (>50 devices) |
| **Pagination** | ❌ No | ✅ Yes |
| **Performance** | Fast for small sets | Optimized for large sets |
| **Filtering** | Basic tag filtering | Advanced filtering options |
| **History Support** | ⚠️ Not recommended | ❌ Not available |
| **Use Case** | Dedicated hardcoded frontends | General-purpose interfaces |

---

## Small Workspace Approach: allDevices

### When to Use
- Workspaces with **fewer than 50 devices**
- **Dedicated use case frontends** (e.g., soil moisture monitoring)
- When you need **historical data** in the same query
- **Hardcoded applications** with known device types

### Basic Query Structure

```graphql
query GetAllDevicesOverview($workspaceId: String!) {
  allDevices(
    inWorkspace: $workspaceId
    searchTags: ["Soil Moisture"]
    searchTagsAnyAll: any
  ) {
    id
    verboseName
    tags
    metadata
    online
    lastHeard
    lastHeardThreshold
    image
    serialNumber
    
    # Hardcoded field measurements
    currentMeasurements(fieldNames: ["SOIL_MOISTURE", "SOIL_TEMPERATURE"]) {
      value
      field {
        fieldName
        unit
      }
    }
    
    # Role-based measurements
    roleFields {
      field {
        fieldName
        unit
      }
      role
      value
    }
    
    # Historical data (use sparingly!)
    history(
      fields: ["SOIL_MOISTURE", "SOIL_TEMPERATURE"]
      timerangestart: "2025-02-22T05:33:03"
      timerangeend: "2025-02-23T05:33:03"
      resolution: "1h"
    )
  }
}
```

### ⚠️ Important Limitations

- **No pagination** - returns ALL matching devices
- **History queries are expensive** - avoid in production for multiple devices
- **Not suitable for large datasets** - performance degrades significantly

---

## Large Workspace Approach: devicesFiltered

### When to Use
- Workspaces with **more than 50 devices**
- Need **pagination** for performance
- Require **advanced filtering** capabilities
- Building **general-purpose interfaces**

### Basic Query Structure

```graphql
query GetFilteredDevicesOverview($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      tags: { contains: ["Soil Moisture"] }
      page: 0
      pageSize: 20
      orderBy: { verboseName: ASC }
    ) {
      total  # Total count of ALL matching devices
      devices {  # Only devices for current page
        id
        verboseName
        tags
        metadata
        online
        lastHeard
        lastHeardThreshold
        image
        serialNumber
        
        # Product information
        product {
          id
          name
          hardware
        }
        
        # Hardcoded field measurements
        currentMeasurements(fieldNames: ["SOIL_MOISTURE", "SOIL_TEMPERATURE"]) {
          value
          field {
            fieldName
            unit
          }
        }
        
        # Role-based measurements
        roleFields {
          field {
            fieldName
            unit
          }
          role
          value
        }
      }
    }
  }
}
```

### Response Structure

```json
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 150,  // Total devices matching filter
        "devices": [   // Only 20 devices (current page)
          {
            "id": "device-001",
            "verboseName": "Block 1",
            "tags": ["Soil Moisture"],
            "online": true,
            "lastHeard": "2025-02-23T05:33:03.987968+00:00",
            "product": {
              "id": "product-uuid",
              "name": "Seeed SenseCAP S2104/S2105"
            },
            "currentMeasurements": [
              {
                "value": 27.1,
                "field": {
                  "fieldName": "SOIL_MOISTURE",
                  "unit": "%"
                }
              }
            ]
          }
        ]
      }
    }
  }
}
```

### Key Pagination Notes

- **`total`** - Always returns count of ALL devices matching the filter
- **`devices`** - Returns only devices for the current page
- **Page numbering** - Starts at 0
- **Performance** - Consistently fast regardless of total device count

---

## Product-Based Architecture

### Concept
Datacake's **products** ensure all devices of the same type have identical field structures. This is crucial for building scalable, hardcoded frontends.

### Benefits
- **Consistent field identifiers** across all devices of the same product
- **Scalable to thousands** of devices with same structure
- **Predictable data structure** for hardcoded queries
- **Easy management** through workspaces and tags

### Query with Product Information

```graphql
query GetDevicesWithProducts($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      tags: { contains: ["Environmental"] }
      pageSize: 50
    ) {
      total
      devices {
        id
        verboseName
        online
        
        # Product details
        product {
          id
          name
          hardware
          measurementFields {
            fieldName
            verboseFieldName
            unit
            fieldType
            semantic
          }
        }
        
        # Consistent measurements across product
        currentMeasurements(fieldNames: ["TEMPERATURE", "HUMIDITY", "CO2"]) {
          value
          field {
            fieldName
            unit
          }
        }
      }
    }
  }
}
```

### Organization Strategy

```
Organization
├── Workspace: "Farm A"
│   ├── Product: "Soil Moisture Sensor"
│   │   ├── Device: "Block 1" [tags: soil, zone-a]
│   │   ├── Device: "Block 2" [tags: soil, zone-a]
│   │   └── Device: "Block 3" [tags: soil, zone-b]
│   └── Product: "Weather Station"
│       ├── Device: "Station North" [tags: weather, zone-a]
│       └── Device: "Station South" [tags: weather, zone-b]
```

---

## Hardcoded Field Strategy

### Why Hardcode?
- **Performance** - No need for field discovery queries
- **Reliability** - No risk of field name changes breaking your app
- **Simplicity** - Direct, predictable queries
- **Best Practice** for dedicated use case applications

### Discovery Process

1. **Via Web Interface** (Recommended)
   - Go to [app.datacake.de](https://app.datacake.de)
   - Navigate to your device
   - Check device configuration
   - Note field identifiers

2. **Via API** (For documentation)
   ```graphql
   query DiscoverFields($workspaceId: String!, $deviceId: String!) {
     workspace(id: $workspaceId) {
       device(id: $deviceId) {
         product {
           measurementFields {
             fieldName
             verboseFieldName
             unit
             fieldType
           }
         }
       }
     }
   }
   ```

### Implementation Example

```javascript
// Hardcoded field configuration
const SOIL_SENSOR_FIELDS = {
  MOISTURE: 'SOIL_MOISTURE',
  TEMPERATURE: 'SOIL_TEMPERATURE',
  CONDUCTIVITY: 'ELECTRICAL_CONDUCTIVITY',
  BATTERY: 'BATTERY',
  SIGNAL: 'LORA_RSSI'
};

// Use in queries
const query = `
  query GetSoilData($workspaceId: String!) {
    workspace(id: $workspaceId) {
      devicesFiltered(
        tags: { contains: ["Soil"] }
        pageSize: 20
      ) {
        devices {
          id
          verboseName
          currentMeasurements(fieldNames: [
            "${SOIL_SENSOR_FIELDS.MOISTURE}",
            "${SOIL_SENSOR_FIELDS.TEMPERATURE}",
            "${SOIL_SENSOR_FIELDS.CONDUCTIVITY}"
          ]) {
            value
            field {
              fieldName
              unit
            }
          }
        }
      }
    }
  }
`;
```

---

## Role Fields for Dynamic Display

### What are Role Fields?
User-configurable assignments that map device fields to standardized roles for dynamic display purposes.

### Available Roles
- **PRIMARY** - Main measurement value
- **SECONDARY** - Secondary measurement value  
- **DEVICE_BATTERY** - Battery level
- **DEVICE_SIGNAL** - Signal strength
- **DEVICE_LOCATION** - GPS coordinates

### Query Role Fields

```graphql
query GetDeviceRoles($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(pageSize: 20) {
      devices {
        id
        verboseName
        roleFields {
          field {
            fieldName
            unit
          }
          role
          value
        }
      }
    }
  }
}
```

### Response Example

```json
{
  "roleFields": [
    {
      "field": { "fieldName": "SOIL_MOISTURE", "unit": "%" },
      "role": "PRIMARY",
      "value": "27.1"
    },
    {
      "field": { "fieldName": "ELECTRICAL_CONDUCTIVITY", "unit": "dS/m" },
      "role": "SECONDARY", 
      "value": "0.1"
    },
    {
      "field": { "fieldName": "BATTERY", "unit": "%" },
      "role": "DEVICE_BATTERY",
      "value": "85"
    }
  ]
}
```

### Using Role Fields in Frontend

```javascript
// Process role fields for dynamic display
function processRoleFields(roleFields) {
  const roles = {};
  
  roleFields.forEach(field => {
    roles[field.role] = {
      value: field.value,
      unit: field.field.unit,
      fieldName: field.field.fieldName
    };
  });
  
  return {
    primary: roles.PRIMARY,
    secondary: roles.SECONDARY,
    battery: roles.DEVICE_BATTERY,
    signal: roles.DEVICE_SIGNAL,
    location: roles.DEVICE_LOCATION
  };
}
```

---

## Tag Organization Strategies

### Hierarchical Tagging

```graphql
# Example tag structure
tags: [
  "Environmental",     // Device category
  "Indoor",           // Location type
  "Floor-2",          // Specific location
  "Conference-Room",  // Sub-location
  "CO2-Monitor"       // Device function
]
```

### Filtering Strategies

```graphql
# All environmental sensors
devicesFiltered(tags: { contains: ["Environmental"] })

# Indoor sensors on specific floor
devicesFiltered(tags: { contains: ["Indoor", "Floor-2"] })

# Any sensor in conference rooms (across floors)
devicesFiltered(tags: { overlap: ["Conference-Room"] })

# CO2 monitors specifically
devicesFiltered(tags: { contains: ["CO2-Monitor"] })
```

### Best Practices

1. **Consistent naming** - Use standard tag formats
2. **Hierarchical structure** - Location → Sub-location → Function
3. **Avoid spaces** - Use hyphens or underscores
4. **Document your schema** - Keep a tag reference for your team

---

## Device Folder Structure

### Overview

Datacake's folder structure feature allows you to organize devices into hierarchical folders in the sidebar. Folders provide a visual organization layer on top of your device tagging strategy, making it easier for users to navigate and find devices.

### Key Concepts

- **Folder Hierarchy** - Unlimited nesting of folders within folders
- **Tag-Based Assignment** - Devices are automatically included in folders based on their tags
- **Tag Conjunction** - Control whether devices must match ALL tags or ANY tag
- **Multiple Views** - Folders can display devices in different views (OVERVIEW, LIST, GRID, MAP)
- **Custom Icons** - Each folder can have a custom icon for visual identification

### Folder Properties

Each folder contains the following properties:

| Property | Type | Description |
|----------|------|-------------|
| **id** | String | Unique folder identifier (UUID) |
| **name** | String | Display name of the folder |
| **icon** | String | Icon identifier (e.g., "folder", "sun", "chart-bar") |
| **tags** | Array | List of tags used to filter devices |
| **tagsConjunction** | String | "AND" (all tags required) or "OR" (any tag matches) |
| **type** | String | Always "DEVICE_FOLDER" |
| **items** | Array | Nested subfolders (recursive structure) |
| **description** | String | Optional folder description |
| **initialView** | String | Default view when folder is opened |
| **availableViews** | Array | Views available for this folder |
| **online** | Boolean | Computed status (not typically used) |

### Loading Folder Structure

```graphql
query LoadFolderStructure($workspaceId: String!) {
  workspace(id: $workspaceId) {
    deviceFolders
  }
}
```

**Response:**

The `deviceFolders` field returns a JSON string containing an array of folder objects. Note that it's a **stringified JSON**, so you'll need to parse it on the client side.

```json
{
  "data": {
    "workspace": {
      "deviceFolders": "[{\"id\":\"901d6df7-4028-4763-bca4-bac21af049e1\",\"icon\":\"folder\",\"name\":\"Cafeteria\",\"tags\":[\"Cafeteria\"],\"type\":\"DEVICE_FOLDER\",\"items\":[],\"online\":false,\"description\":\"\",\"initialView\":\"OVERVIEW\",\"availableViews\":[\"OVERVIEW\",\"LIST\"],\"tagsConjunction\":\"AND\"}]"
    }
  }
}
```

### Parsed Folder Structure Example

After parsing the JSON string, you'll have a structure like this:

```javascript
const parsedFolders = JSON.parse(response.data.workspace.deviceFolders);

// Example structure:
[
  {
    "id": "901d6df7-4028-4763-bca4-bac21af049e1",
    "icon": "folder",
    "name": "Cafeteria",
    "tags": ["Cafeteria"],
    "type": "DEVICE_FOLDER",
    "items": [],
    "online": false,
    "description": "",
    "initialView": "OVERVIEW",
    "availableViews": ["OVERVIEW", "LIST"],
    "tagsConjunction": "AND"
  },
  {
    "id": "a2ddb645-827d-4bf3-a6f3-a8687458a177",
    "icon": "folder",
    "name": "Second Floor",
    "tags": ["ASS"],
    "type": "DEVICE_FOLDER",
    "items": [
      {
        "id": "ef74c797-c558-40a0-92e6-e2b33d1c662e",
        "icon": "folder",
        "name": "Executive Area",
        "tags": ["GL"],
        "type": "DEVICE_FOLDER",
        "items": [
          {
            "id": "50eedb91-778c-487a-94a8-557db8a9e85f",
            "icon": "folder",
            "name": "Exec 1",
            "tags": [],
            "type": "DEVICE_FOLDER",
            "items": [],
            "online": false,
            "description": "",
            "initialView": "OVERVIEW",
            "availableViews": ["OVERVIEW", "LIST", "GRID", "MAP"],
            "tagsConjunction": "AND"
          }
        ],
        "online": false,
        "description": "",
        "initialView": "OVERVIEW",
        "availableViews": ["OVERVIEW", "LIST"],
        "tagsConjunction": "AND"
      },
      {
        "id": "04f1f948-ffe2-4672-8ef9-feab0e13367c",
        "icon": "sun",
        "name": "Bathrooms",
        "tags": ["WC"],
        "type": "DEVICE_FOLDER",
        "items": [],
        "online": false,
        "description": "",
        "initialView": "LIST",
        "availableViews": ["OVERVIEW", "LIST"],
        "tagsConjunction": "AND"
      }
    ],
    "online": false,
    "description": "",
    "initialView": "OVERVIEW",
    "availableViews": ["OVERVIEW", "LIST"],
    "tagsConjunction": "AND"
  }
]
```

### Tag Conjunction Behavior

The `tagsConjunction` property controls how devices are matched to folders:

#### AND - All Tags Required

```javascript
{
  "name": "Meeting Rooms Floor 2",
  "tags": ["Meeting", "Floor-2"],
  "tagsConjunction": "AND"
}
// Matches devices that have BOTH "Meeting" AND "Floor-2" tags
```

#### OR - Any Tag Matches

```javascript
{
  "name": "All Sensors",
  "tags": ["Temperature", "Humidity", "CO2"],
  "tagsConjunction": "OR"
}
// Matches devices that have Temperature OR Humidity OR CO2 tags
```

### Updating Folder Structure

To update the folder structure, use the `updateDeviceFolders` mutation:

```graphql
mutation UpdateFolderStructure($workspaceId: String!, $foldersJson: String!) {
  updateDeviceFolders(input: {
    workspaceId: $workspaceId
    deviceFolders: $foldersJson
  }) {
    ok
  }
}
```

**Variables:**

```json
{
  "workspaceId": "764cd5ed-520d-4f15-9391-4de66f3e6359",
  "foldersJson": "[{\"id\":\"901d6df7-4028-4763-bca4-bac21af049e1\",\"icon\":\"folder\",\"name\":\"Cafeteria\",\"tags\":[\"Cafeteria\"],\"type\":\"DEVICE_FOLDER\",\"items\":[],\"online\":false,\"description\":\"\",\"initialView\":\"OVERVIEW\",\"availableViews\":[\"OVERVIEW\",\"LIST\"],\"tagsConjunction\":\"AND\"}]"
}
```

**Important:** The `deviceFolders` parameter must be a **stringified JSON array**, not a JavaScript object.

### Processing Folder Structure in Frontend

```javascript
// Fetch and parse folder structure
async function loadFolderStructure(workspaceId) {
  const query = `
    query LoadFolderStructure($workspaceId: String!) {
      workspace(id: $workspaceId) {
        deviceFolders
      }
    }
  `;
  
  const response = await graphqlClient.query({
    query,
    variables: { workspaceId }
  });
  
  // Parse the JSON string
  const folders = JSON.parse(response.data.workspace.deviceFolders);
  return folders;
}

// Create folder structure
function createFolderStructure() {
  const folders = [
    {
      id: generateUUID(),
      icon: "folder",
      name: "Building A",
      tags: ["BuildingA"],
      type: "DEVICE_FOLDER",
      items: [
        {
          id: generateUUID(),
          icon: "folder",
          name: "Floor 1",
          tags: ["BuildingA", "Floor1"],
          type: "DEVICE_FOLDER",
          items: [],
          online: false,
          description: "First floor devices",
          initialView: "OVERVIEW",
          availableViews: ["OVERVIEW", "LIST"],
          tagsConjunction: "AND"
        }
      ],
      online: false,
      description: "All devices in Building A",
      initialView: "OVERVIEW",
      availableViews: ["OVERVIEW", "LIST", "GRID"],
      tagsConjunction: "AND"
    }
  ];
  
  return JSON.stringify(folders);
}

// Update folder structure
async function updateFolderStructure(workspaceId, folders) {
  const mutation = `
    mutation UpdateFolderStructure($workspaceId: String!, $foldersJson: String!) {
      updateDeviceFolders(input: {
        workspaceId: $workspaceId
        deviceFolders: $foldersJson
      }) {
        ok
      }
    }
  `;
  
  const foldersJson = JSON.stringify(folders);
  
  const response = await graphqlClient.mutate({
    mutation,
    variables: {
      workspaceId,
      foldersJson
    }
  });
  
  return response.data.updateDeviceFolders.ok;
}
```

### Recursive Folder Rendering

```javascript
// Render folder tree recursively
function renderFolderTree(folders, level = 0) {
  return folders.map(folder => ({
    id: folder.id,
    name: folder.name,
    icon: folder.icon,
    level: level,
    tags: folder.tags,
    tagsConjunction: folder.tagsConjunction,
    initialView: folder.initialView,
    availableViews: folder.availableViews,
    children: folder.items ? renderFolderTree(folder.items, level + 1) : []
  }));
}

// Get all folders as flat list with parent references
function flattenFolderTree(folders, parentId = null, result = []) {
  folders.forEach(folder => {
    result.push({
      ...folder,
      parentId: parentId,
      hasChildren: folder.items && folder.items.length > 0
    });
    
    if (folder.items && folder.items.length > 0) {
      flattenFolderTree(folder.items, folder.id, result);
    }
  });
  
  return result;
}

// Find folder by ID
function findFolderById(folders, folderId) {
  for (const folder of folders) {
    if (folder.id === folderId) {
      return folder;
    }
    
    if (folder.items && folder.items.length > 0) {
      const found = findFolderById(folder.items, folderId);
      if (found) return found;
    }
  }
  
  return null;
}
```

### Getting Devices for a Folder

Once you have a folder, use its tags to query devices:

```javascript
// Get devices matching folder criteria
async function getDevicesForFolder(workspaceId, folder) {
  const query = `
    query GetFolderDevices(
      $workspaceId: String!
      $tags: [String!]
      $tagsCondition: String!
    ) {
      workspace(id: $workspaceId) {
        devicesFiltered(
          tags: { 
            ${folder.tagsConjunction === "AND" ? "contains" : "overlap"}: $tags 
          }
          pageSize: 50
        ) {
          total
          devices {
            id
            verboseName
            online
            lastHeard
            tags
            roleFields {
              value
              role
              field {
                fieldName
                unit
              }
            }
          }
        }
      }
    }
  `;
  
  const response = await graphqlClient.query({
    query,
    variables: {
      workspaceId,
      tags: folder.tags,
      tagsCondition: folder.tagsConjunction
    }
  });
  
  return response.data.workspace.devicesFiltered;
}
```

### Folder Organization Best Practices

1. **Hierarchical Structure**
   - Use meaningful folder hierarchy that reflects physical or logical organization
   - Example: `Building → Floor → Room Type → Specific Room`

2. **Tag Strategy**
   - Create consistent tag naming conventions
   - Use hierarchical tags that match your folder structure
   - Example: `["BuildingA", "Floor1", "MeetingRoom"]`

3. **Empty Folder Handling**
   - Folders with empty `tags` array won't automatically include any devices
   - Use specific tags to ensure devices are properly categorized

4. **Tag Conjunction Selection**
   - Use `"AND"` for precise filtering (device must have all tags)
   - Use `"OR"` for inclusive filtering (device can have any of the tags)

5. **View Configuration**
   - Set appropriate `initialView` based on folder content
   - `"OVERVIEW"` - Visual cards with key metrics
   - `"LIST"` - Detailed table view
   - `"GRID"` - Compact grid layout
   - `"MAP"` - Geographic map view (requires location data)

### Complete Folder Management Example

```javascript
class FolderManager {
  constructor(workspaceId, graphqlClient) {
    this.workspaceId = workspaceId;
    this.client = graphqlClient;
    this.folders = [];
  }
  
  // Load folders from API
  async load() {
    const query = `
      query LoadFolderStructure($workspaceId: String!) {
        workspace(id: $workspaceId) {
          deviceFolders
        }
      }
    `;
    
    const response = await this.client.query({
      query,
      variables: { workspaceId: this.workspaceId }
    });
    
    this.folders = JSON.parse(response.data.workspace.deviceFolders);
    return this.folders;
  }
  
  // Save folders to API
  async save() {
    const mutation = `
      mutation UpdateFolderStructure($workspaceId: String!, $foldersJson: String!) {
        updateDeviceFolders(input: {
          workspaceId: $workspaceId
          deviceFolders: $foldersJson
        }) {
          ok
        }
      }
    `;
    
    const response = await this.client.mutate({
      mutation,
      variables: {
        workspaceId: this.workspaceId,
        foldersJson: JSON.stringify(this.folders)
      }
    });
    
    return response.data.updateDeviceFolders.ok;
  }
  
  // Add new folder
  addFolder(parentId, folderData) {
    const newFolder = {
      id: this.generateUUID(),
      icon: folderData.icon || "folder",
      name: folderData.name,
      tags: folderData.tags || [],
      type: "DEVICE_FOLDER",
      items: [],
      online: false,
      description: folderData.description || "",
      initialView: folderData.initialView || "OVERVIEW",
      availableViews: folderData.availableViews || ["OVERVIEW", "LIST"],
      tagsConjunction: folderData.tagsConjunction || "AND"
    };
    
    if (parentId) {
      const parent = this.findFolder(this.folders, parentId);
      if (parent) {
        parent.items.push(newFolder);
      }
    } else {
      this.folders.push(newFolder);
    }
    
    return newFolder;
  }
  
  // Update folder
  updateFolder(folderId, updates) {
    const folder = this.findFolder(this.folders, folderId);
    if (folder) {
      Object.assign(folder, updates);
    }
    return folder;
  }
  
  // Delete folder
  deleteFolder(folderId) {
    return this.deleteFolderRecursive(this.folders, folderId);
  }
  
  // Helper: Find folder recursively
  findFolder(folders, folderId) {
    for (const folder of folders) {
      if (folder.id === folderId) {
        return folder;
      }
      if (folder.items && folder.items.length > 0) {
        const found = this.findFolder(folder.items, folderId);
        if (found) return found;
      }
    }
    return null;
  }
  
  // Helper: Delete folder recursively
  deleteFolderRecursive(folders, folderId) {
    for (let i = 0; i < folders.length; i++) {
      if (folders[i].id === folderId) {
        folders.splice(i, 1);
        return true;
      }
      if (folders[i].items && folders[i].items.length > 0) {
        if (this.deleteFolderRecursive(folders[i].items, folderId)) {
          return true;
        }
      }
    }
    return false;
  }
  
  // Helper: Generate UUID
  generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
      const r = Math.random() * 16 | 0;
      const v = c === 'x' ? r : (r & 0x3 | 0x8);
      return v.toString(16);
    });
  }
}

// Usage
const folderManager = new FolderManager(workspaceId, graphqlClient);

// Load existing folders
await folderManager.load();

// Add new folder
folderManager.addFolder(null, {
  name: "Production Floor",
  tags: ["Production"],
  icon: "folder",
  initialView: "LIST",
  availableViews: ["OVERVIEW", "LIST", "GRID"],
  tagsConjunction: "AND"
});

// Save changes
await folderManager.save();
```

### Integration with Device Queries

Combine folder structure with device queries for a complete solution:

```graphql
query FolderWithDevices($workspaceId: String!, $folderTags: [String!]!) {
  workspace(id: $workspaceId) {
    # Get folder structure
    deviceFolders
    
    # Get devices for specific folder
    devices: devicesFiltered(
      tags: { contains: $folderTags }
      pageSize: 20
    ) {
      total
      devices {
        id
        verboseName
        online
        lastHeard
        tags
        product {
          name
        }
        roleFields {
          value
          role
          field {
            fieldName
            unit
          }
        }
      }
    }
  }
}
```

### Folder Structure Validation

When updating folder structure, ensure your JSON is valid:

```javascript
function validateFolderStructure(folders) {
  const errors = [];
  
  function validateFolder(folder, path = "") {
    // Required fields
    if (!folder.id) errors.push(`${path}: Missing id`);
    if (!folder.name) errors.push(`${path}: Missing name`);
    if (!folder.type || folder.type !== "DEVICE_FOLDER") {
      errors.push(`${path}: Invalid type`);
    }
    
    // Validate arrays
    if (!Array.isArray(folder.tags)) {
      errors.push(`${path}: tags must be an array`);
    }
    if (!Array.isArray(folder.items)) {
      errors.push(`${path}: items must be an array`);
    }
    if (!Array.isArray(folder.availableViews)) {
      errors.push(`${path}: availableViews must be an array`);
    }
    
    // Validate tagsConjunction
    if (!["AND", "OR"].includes(folder.tagsConjunction)) {
      errors.push(`${path}: tagsConjunction must be "AND" or "OR"`);
    }
    
    // Validate initialView
    const validViews = ["OVERVIEW", "LIST", "GRID", "MAP"];
    if (!validViews.includes(folder.initialView)) {
      errors.push(`${path}: Invalid initialView`);
    }
    
    // Recursively validate subfolders
    if (folder.items && folder.items.length > 0) {
      folder.items.forEach((subfolder, index) => {
        validateFolder(subfolder, `${path}/${folder.name}[${index}]`);
      });
    }
  }
  
  folders.forEach((folder, index) => {
    validateFolder(folder, `root[${index}]`);
  });
  
  return {
    valid: errors.length === 0,
    errors
  };
}

// Usage
const validation = validateFolderStructure(folders);
if (!validation.valid) {
  console.error("Folder structure validation failed:", validation.errors);
}
```

---

## Performance Considerations

### Query Optimization

| ✅ Do | ❌ Don't |
|-------|----------|
| Use `devicesFiltered` for >50 devices | Use `allDevices` with history for many devices |
| Implement pagination | Request all devices at once |
| Hardcode field identifiers | Discover fields on every request |
| Use specific tag filtering | Query all devices then filter client-side |
| Request only needed fields | Request all available fields |
| **Use pagination even for alerts** | **Request all alert devices without pagination** |

### Critical Alert Query Performance

**⚠️ Important**: Even "alert" queries (low battery, high temperature, etc.) can return hundreds of devices and must use pagination when requesting device details.

```graphql
# ✅ GOOD: Count only (fast)
lowBatteryCount: devicesFiltered(
  battery: { lt: 20.0 }
) {
  total  # Fast - just returns count
}

# ✅ GOOD: With pagination when you need details
lowBatteryDevices: devicesFiltered(
  battery: { lt: 20.0 }
  pageSize: 10  # CRITICAL: Always limit when requesting devices
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

# ❌ BAD: Could return 1000s of devices
lowBatteryDevices: devicesFiltered(
  battery: { lt: 20.0 }
  # No pageSize = potential performance disaster
) {
  devices {
    id
    verboseName
    # ... many fields
  }
}
```

### Memory and Network

```graphql
# GOOD: Specific fields only
currentMeasurements(fieldNames: ["TEMPERATURE", "HUMIDITY"]) {
  value
  field {
    fieldName
    unit
  }
}

# AVOID: All measurements
currentMeasurements {
  value
  field {
    fieldName
    verboseFieldName
    unit
    fieldType
    semantic
  }
}
```

### Pagination Strategy

```javascript
// Efficient pagination implementation
const DEVICES_PER_PAGE = 20;

async function loadDevicesPage(page = 0) {
  const query = `
    query GetDevicesPage($workspaceId: String!, $page: Int!, $pageSize: Int!) {
      workspace(id: $workspaceId) {
        devicesFiltered(
          page: $page
          pageSize: $pageSize
          orderBy: { verboseName: ASC }
        ) {
          total
          devices {
            id
            verboseName
            online
            # ... other fields
          }
        }
      }
    }
  `;
  
  return await graphqlClient.query({
    query,
    variables: {
      workspaceId: WORKSPACE_ID,
      page,
      pageSize: DEVICES_PER_PAGE
    }
  });
}
```

---

## Complete Examples

### Soil Moisture Monitoring Dashboard

```graphql
query SoilMoistureDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # Overview statistics
    allSoilSensors: devicesFiltered(
      tags: { contains: ["Soil"] }
    ) {
      total
    }
    
    # Online sensors
    onlineSensors: devicesFiltered(
      tags: { contains: ["Soil"] }
      online: true
    ) {
      total
    }
    
    # Low moisture alerts - COUNT ONLY (fast)
    lowMoistureCount: devicesFiltered(
      tags: { contains: ["Soil"] }
      soilMoisture: { lt: 20.0 }
    ) {
      total  # Only count - very fast
    }
    
    # Low moisture alerts - WITH PAGINATION (when you need device details)
    lowMoistureDevices: devicesFiltered(
      tags: { contains: ["Soil"] }
      soilMoisture: { lt: 20.0 }
      pageSize: 10  # CRITICAL: Always paginate when requesting devices
      orderBy: { lastHeard: DESC }
    ) {
      total  # Still get total count
      devices {
        id
        verboseName
        lastHeard
        numericSemanticField(semantic: SOIL_MOISTURE) {
          value
        }
      }
    }
    
    # Recent activity
    recentDevices: devicesFiltered(
      tags: { contains: ["Soil"] }
      pageSize: 10
      orderBy: { lastHeard: DESC }
    ) {
      devices {
        id
        verboseName
        lastHeard
        online
        currentMeasurements(fieldNames: ["SOIL_MOISTURE", "SOIL_TEMPERATURE"]) {
          value
          field {
            fieldName
            unit
          }
        }
        roleFields {
          field {
            fieldName
            unit
          }
          role
          value
        }
      }
    }
  }
}
```

### Multi-Product Environment Monitor

```graphql
query EnvironmentOverview($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # CO2 Monitors
    co2Devices: devicesFiltered(
      tags: { contains: ["CO2"] }
      pageSize: 10
    ) {
      total
      devices {
        id
        verboseName
        product {
          name
        }
        currentMeasurements(fieldNames: ["CO2", "TEMPERATURE", "HUMIDITY"]) {
          value
          field {
            fieldName
            unit
          }
        }
      }
    }
    
    # Weather Stations
    weatherStations: devicesFiltered(
      tags: { contains: ["Weather"] }
      pageSize: 5
    ) {
      total
      devices {
        id
        verboseName
        product {
          name
        }
        currentMeasurements(fieldNames: ["TEMPERATURE", "HUMIDITY", "PRESSURE"]) {
          value
          field {
            fieldName
            unit
          }
        }
      }
    }
    
    # All environmental data aggregated
    environmentalOverview: devicesFiltered(
      tags: { overlap: ["Environmental", "Weather", "CO2"] }
      online: true
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: AVG
      )
      avgHumidity: aggregatedNumericSemanticValue(
        semantic: HUMIDITY
        aggregation: AVG
      )
    }
  }
}
```

### Device Status Dashboard

```graphql
query DeviceStatusDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # Alert counts (fast overview)
    offlineCount: devicesFiltered(online: false) {
      total
    }
    
    lowBatteryCount: devicesFiltered(battery: { lt: 20.0 }) {
      total
    }
    
    # Offline devices needing attention (paginated)
    offlineDevices: devicesFiltered(
      online: false
      pageSize: 10  # Always paginate - could be many offline devices
      orderBy: { lastHeard: ASC }
    ) {
      total
      devices {
        id
        verboseName
        lastHeard
        lastHeardThreshold
        product {
          name
        }
        roleFields {
          field {
            fieldName
            unit
          }
          role
          value
        }
      }
    }
    
    # Low battery devices (paginated)
    lowBatteryDevices: devicesFiltered(
      battery: { lt: 20.0 }
      pageSize: 10  # Always paginate - could be many low battery devices
      orderBy: { lastHeard: DESC }
    ) {
      total
      devices {
        id
        verboseName
        lastHeard
        numericSemanticField(semantic: BATTERY) {
          value
        }
      }
    }
    
    # Recently active devices
    recentActivity: devicesFiltered(
      online: true
      pageSize: 15
      orderBy: { lastHeard: DESC }
    ) {
      devices {
        id
        verboseName
        lastHeard
        product {
          name
        }
      }
    }
  }
}
```

---

## Dynamic Multi-Product Queries with Role Fields

### When You Need Role Fields

Role fields are essential when your workspace contains devices from **multiple product types** with different field structures:

- **Manufacturer platforms** offering frontends for customer's own products
- **Mixed LoRaWAN deployments** with devices from different vendors
- **Multi-vendor environments** where each product has unique field names
- **Scalable solutions** that need to work across product types

### Role Fields vs Other Approaches

| Approach | Use Case | Cross-Product | Dynamic |
|----------|----------|---------------|---------|
| **Hardcoded Fields** | Single product type | ❌ No | ❌ No |
| **Semantics** | Cross-device filtering | ✅ Yes | ❌ Limited |
| **Role Fields** | Multi-product overviews | ✅ Yes | ✅ Yes |

### Available Standard Roles

```graphql
enum DeviceRole {
  PRIMARY          # Main measurement value
  SECONDARY        # Secondary measurement value
  DEVICE_BATTERY   # Battery level/voltage
  DEVICE_SIGNAL    # Signal strength (RSSI, etc.)
  DEVICE_LOCATION  # GPS coordinates
}
```

### Dynamic All Devices Query

**For small workspaces with mixed products:**

```graphql
query DynamicAllDevicesOverview($workspaceId: String!) {
  allDevices(inWorkspace: $workspaceId) {
    id
    verboseName
    serialNumber
    online
    lastHeard
    tags
    
    # Product information
    product {
      id
      name
      hardware
    }
    
    # Dynamic role fields - works across different products
    roleFields {
      value
      role
      field {
        fieldName
        unit
      }
    }
  }
}
```

### Dynamic Filtered Devices Query

**For larger workspaces with pagination:**

```graphql
query DynamicFilteredDevicesOverview($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      all: true  # Include all devices
      page: 0
      pageSize: 20
      orderBy: { verboseName: ASC }
    ) {
      total
      devices {
        id
        verboseName
        serialNumber
        online
        lastHeard
        tags
        
        # Product information
        product {
          id
          name
          hardware
        }
        
        # Dynamic role fields - works across different products
        roleFields {
          value
          role
          field {
            fieldName
            unit
          }
        }
      }
    }
  }
}
```

### Real Response Example

```json
{
  "data": {
    "workspace": {
      "devicesFiltered": {
        "total": 39,
        "devices": [
          {
            "id": "5fa87e1f-b4b8-4801-b2e4-11960d300d71",
            "verboseName": "01 - Blumenbeet an der Terrasse",
            "product": {
              "name": "Soil Moisture Sensor Type A"
            },
            "roleFields": [
              {
                "value": "3.1",
                "role": "DEVICE_BATTERY",
                "field": {
                  "fieldName": "BATTERY",
                  "unit": "V"
                }
              },
              {
                "value": "0",
                "role": "PRIMARY",
                "field": {
                  "fieldName": "MOISTURE",
                  "unit": "%"
                }
              },
              {
                "value": "17.83",
                "role": "SECONDARY",
                "field": {
                  "fieldName": "TEMPERATURE",
                  "unit": "°C"
                }
              },
              {
                "value": "-101",
                "role": "DEVICE_SIGNAL",
                "field": {
                  "fieldName": "LORA_RSSI",
                  "unit": "dBm"
                }
              },
              {
                "value": "(52.029,6.812)",
                "role": "DEVICE_LOCATION",
                "field": {
                  "fieldName": "LOCATION",
                  "unit": ""
                }
              }
            ]
          }
        ]
      }
    }
  }
}
```

### Processing Role Fields in Frontend

```javascript
// Process role fields into standardized display format
function processDeviceRoles(device) {
  const roles = {};
  
  device.roleFields.forEach(roleField => {
    const role = roleField.role;
    roles[role] = {
      value: roleField.value,
      unit: roleField.field.unit,
      fieldName: roleField.field.fieldName,
      displayValue: formatValue(roleField.value, roleField.field.unit)
    };
  });
  
  return {
    id: device.id,
    name: device.verboseName,
    online: device.online,
    product: device.product?.name,
    
    // Standardized role-based display
    primaryValue: roles.PRIMARY,
    secondaryValue: roles.SECONDARY,
    batteryLevel: roles.DEVICE_BATTERY,
    signalStrength: roles.DEVICE_SIGNAL,
    location: roles.DEVICE_LOCATION
  };
}

// Format values based on role and unit
function formatValue(value, unit) {
  if (value === null || value === undefined) return 'N/A';
  
  switch (unit) {
    case '%':
      return `${value}%`;
    case '°C':
      return `${value}°C`;
    case 'dBm':
      return `${value} dBm`;
    case 'V':
      return `${value}V`;
    default:
      return `${value} ${unit}`.trim();
  }
}

// Usage in React/Vue component
const processedDevices = devices.map(processDeviceRoles);
```

### Multi-Product Dashboard Example

```graphql
query MultiProductDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    name
    
    # Overview statistics
    totalDevices: devicesFiltered(all: true) {
      total
    }
    
    onlineDevices: devicesFiltered(
      all: true
      online: true
    ) {
      total
    }
    
    # Low battery alerts (using role fields)
    lowBatteryDevices: devicesFiltered(
      all: true
      pageSize: 10
    ) {
      devices {
        id
        verboseName
        online
        product {
          name
        }
        roleFields {
          value
          role
          field {
            fieldName
            unit
          }
        }
      }
    }
    
    # Recent activity across all products
    recentActivity: devicesFiltered(
      all: true
      online: true
      pageSize: 15
      orderBy: { lastHeard: DESC }
    ) {
      devices {
        id
        verboseName
        lastHeard
        product {
          name
        }
        roleFields {
          value
          role
          field {
            fieldName
            unit
          }
        }
      }
    }
  }
}
```

### Role Field Filtering and Display

```javascript
// Create a dashboard component that works across products
class MultiProductDashboard {
  constructor(devices) {
    this.devices = devices.map(this.processDevice);
  }
  
  processDevice(device) {
    const processed = {
      id: device.id,
      name: device.verboseName,
      online: device.online,
      product: device.product?.name || 'Unknown',
      roles: {}
    };
    
    // Process role fields into convenient object
    device.roleFields.forEach(roleField => {
      processed.roles[roleField.role] = {
        value: roleField.value,
        unit: roleField.field.unit,
        fieldName: roleField.field.fieldName
      };
    });
    
    return processed;
  }
  
  // Get devices with low battery across all products
  getLowBatteryDevices(threshold = 20) {
    return this.devices.filter(device => {
      const battery = device.roles.DEVICE_BATTERY;
      if (!battery || battery.value === null) return false;
      return parseFloat(battery.value) < threshold;
    });
  }
  
  // Get devices with weak signal across all products  
  getWeakSignalDevices(threshold = -100) {
    return this.devices.filter(device => {
      const signal = device.roles.DEVICE_SIGNAL;
      if (!signal || signal.value === null) return false;
      return parseFloat(signal.value) < threshold;
    });
  }
  
  // Create standardized display rows
  createDisplayRows() {
    return this.devices.map(device => ({
      id: device.id,
      name: device.name,
      product: device.product,
      status: device.online ? 'Online' : 'Offline',
      primary: this.formatRoleValue(device.roles.PRIMARY),
      secondary: this.formatRoleValue(device.roles.SECONDARY),
      battery: this.formatRoleValue(device.roles.DEVICE_BATTERY),
      signal: this.formatRoleValue(device.roles.DEVICE_SIGNAL)
    }));
  }
  
  formatRoleValue(role) {
    if (!role || role.value === null) return 'N/A';
    return `${role.value} ${role.unit}`.trim();
  }
}
```

### Role Field Configuration Benefits

1. **User-Configurable** - Administrators can assign roles to fields
2. **Product-Agnostic** - Works across different device types
3. **Standardized Display** - Consistent UI regardless of underlying fields
4. **Flexible Mapping** - Same role can map to different fields per product
5. **Dynamic Adaptation** - No code changes needed for new products

### Best Practices for Role Fields

✅ **Do:**
- Use role fields for multi-product dashboards
- Process role fields into standardized display objects
- Handle null/missing role values gracefully
- Use pagination even with `all: true`
- Cache role field mappings for performance

❌ **Don't:**
- Mix role fields with hardcoded field queries
- Assume all devices have all roles assigned
- Skip null checking for role values
- Use role fields for single-product applications

---

## Dynamic Semantic Queries for KPIs and Cross-Device Compatibility

### When You Need Semantic Queries

Semantic queries are perfect for:

- **KPI dashboards** needing metrics across all devices regardless of product type
- **Cross-device analytics** where you want temperature data from all sensors
- **Dynamic frontends** that adapt to backend field changes without code updates
- **Multi-vendor environments** where field names vary but semantics remain consistent

### Semantic Queries vs Other Approaches

| Approach | Performance | Cross-Product | Dynamic | Use Case |
|----------|-------------|---------------|---------|----------|
| **Hardcoded Fields** | ⚡ Fastest | ❌ No | ❌ No | Single product, maximum speed |
| **Role Fields** | 🚀 Fast | ✅ Yes | 🔄 User-config | Multi-product, standardized display |
| **Semantic Queries** | 💪 Good | ✅ Yes | ✅ Automatic | KPIs, cross-device analytics |

### 1. Semantic KPI Overview Query

**Perfect for dashboard metrics and overview panels:**

```graphql
query SemanticKPIOverview($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(all: true) {
      # Soil moisture metrics
      soil_moisture_avg: aggregatedNumericSemanticValue(
        semantic: SOIL_MOISTURE
        aggregation: AVG
      )
      soil_moisture_max: aggregatedNumericSemanticValue(
        semantic: SOIL_MOISTURE
        aggregation: MAX
      )
      soil_moisture_min: aggregatedNumericSemanticValue(
        semantic: SOIL_MOISTURE
        aggregation: MIN
      )
      
      # Environmental metrics
      temperature_avg: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: AVG
      )
      humidity_avg: aggregatedNumericSemanticValue(
        semantic: HUMIDITY
        aggregation: AVG
      )
      co2_avg: aggregatedNumericSemanticValue(
        semantic: CO2
        aggregation: AVG
      )
      
      # Air quality metrics
      air_pollution_max: aggregatedNumericSemanticValue(
        semantic: AIR_POLLUTION
        aggregation: MAX
      )
      
      # Energy metrics
      power_total: aggregatedNumericSemanticValue(
        semantic: POWER
        aggregation: SUM
      )
      energy_consumption_total: aggregatedNumericSemanticValue(
        semantic: ENERGY_CONSUMPTION
        aggregation: SUM
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
        "soil_moisture_avg": 4.428571428571429,
        "soil_moisture_max": 13,
        "soil_moisture_min": 0,
        "temperature_avg": 22.5,
        "humidity_avg": 65.2,
        "co2_avg": 420.5,
        "air_pollution_max": 45.2,
        "power_total": 1250.8,
        "energy_consumption_total": 45670.2
      }
    }
  }
}
```

### 2. Dynamic Semantic Overview Query

**Perfect for flexible device listings that adapt to backend changes:**

```graphql
query DynamicSemanticOverview($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      all: true
      page: 0
      pageSize: 20
      orderBy: { verboseName: ASC }
    ) {
      total
      devices {
        id
        verboseName
        online
        lastHeard
        tags
        
        # Product info
        product {
          name
        }
        
        # Dynamic semantic values - adapts to field changes
        co2: numericSemanticField(semantic: CO2) {
          value
        }
        
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
        
        humidity: numericSemanticField(semantic: HUMIDITY) {
          value
        }
        
        battery: numericSemanticField(semantic: BATTERY) {
          value
        }
        
        # Multiple fields with same semantic (aggregated)
        average_temperature: numericSemanticField(
          semantic: TEMPERATURE
          aggregation: AVG
        ) {
          value
        }
        
        # Advanced: Include field details (use sparingly for performance)
        temperature_details: numericSemanticField(
          semantic: TEMPERATURE
          aggregation: AVG
        ) {
          value
          fields {
            fieldName
            unit
            value
          }
        }
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
        "total": 39,
        "devices": [
          {
            "id": "5fa87e1f-b4b8-4801-b2e4-11960d300d71",
            "verboseName": "01 - Blumenbeet an der Terrasse",
            "online": true,
            "product": {
              "name": "Soil Moisture Sensor"
            },
            "co2": {
              "value": null  // Device doesn't have CO2 sensor
            },
            "temperature": {
              "value": 17.83
            },
            "humidity": {
              "value": null  // Device doesn't have humidity sensor
            },
            "battery": {
              "value": 3.1
            },
            "average_temperature": {
              "value": 17.83
            },
            "temperature_details": {
              "value": 17.83,
              "fields": [
                {
                  "fieldName": "TEMPERATURE",
                  "unit": "°C",
                  "value": 17.83
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```

### 3. Filtered Semantic KPIs

**Combine filtering with semantic aggregation:**

```graphql
query FilteredSemanticKPIs($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # All devices metrics
    allDevicesMetrics: devicesFiltered(all: true) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: AVG
      )
    }
    
    # Online devices only
    onlineMetrics: devicesFiltered(
      online: true
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: AVG
      )
      avgHumidity: aggregatedNumericSemanticValue(
        semantic: HUMIDITY
        aggregation: AVG
      )
    }
    
    # High temperature alerts
    hotDevicesMetrics: devicesFiltered(
      temperature: { gt: 35.0 }
    ) {
      total
      maxTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: MAX
      )
    }
    
    # Devices by tag with metrics
    soilSensorMetrics: devicesFiltered(
      tags: { contains: ["Soil"] }
    ) {
      total
      avgSoilMoisture: aggregatedNumericSemanticValue(
        semantic: SOIL_MOISTURE
        aggregation: AVG
      )
      minSoilMoisture: aggregatedNumericSemanticValue(
        semantic: SOIL_MOISTURE
        aggregation: MIN
      )
    }
  }
}
```

### 4. Processing Semantic Data in Frontend

```javascript
// Process semantic KPI data
function processSemanticKPIs(kpiData) {
  return {
    soilMoisture: {
      average: kpiData.soil_moisture_avg?.toFixed(1) || 'N/A',
      maximum: kpiData.soil_moisture_max || 'N/A',
      minimum: kpiData.soil_moisture_min || 'N/A',
      unit: '%'
    },
    temperature: {
      average: kpiData.temperature_avg?.toFixed(1) || 'N/A',
      unit: '°C'
    },
    humidity: {
      average: kpiData.humidity_avg?.toFixed(1) || 'N/A',
      unit: '%'
    },
    airQuality: {
      maximum: kpiData.air_pollution_max?.toFixed(1) || 'N/A',
      unit: 'μg/m³'
    },
    energy: {
      totalPower: kpiData.power_total?.toFixed(1) || 'N/A',
      totalConsumption: kpiData.energy_consumption_total?.toFixed(1) || 'N/A',
      units: { power: 'W', consumption: 'kWh' }
    }
  };
}

// Process dynamic semantic device data
function processSemanticDevices(devices) {
  return devices.map(device => ({
    id: device.id,
    name: device.verboseName,
    online: device.online,
    product: device.product?.name || 'Unknown',
    
    // Handle null semantic values gracefully
    measurements: {
      co2: formatSemanticValue(device.co2, 'ppm'),
      temperature: formatSemanticValue(device.temperature, '°C'),
      humidity: formatSemanticValue(device.humidity, '%'),
      battery: formatSemanticValue(device.battery, 'V')
    },
    
    // Status indicators based on semantic values
    status: {
      temperatureStatus: getTemperatureStatus(device.temperature?.value),
      batteryStatus: getBatteryStatus(device.battery?.value),
      connectivityStatus: device.online ? 'Online' : 'Offline'
    }
  }));
}

// Helper functions
function formatSemanticValue(semanticField, unit) {
  if (!semanticField || semanticField.value === null) {
    return { value: 'N/A', unit: '', status: 'unavailable' };
  }
  
  return {
    value: semanticField.value,
    unit: unit,
    formatted: `${semanticField.value} ${unit}`,
    status: 'available'
  };
}

function getTemperatureStatus(temp) {
  if (temp === null || temp === undefined) return 'unknown';
  if (temp < 0) return 'freezing';
  if (temp > 35) return 'hot';
  if (temp > 25) return 'warm';
  return 'normal';
}

function getBatteryStatus(battery) {
  if (battery === null || battery === undefined) return 'unknown';
  if (battery < 20) return 'low';
  if (battery < 50) return 'medium';
  return 'good';
}
```

### 5. Complete Dashboard with All Approaches

```graphql
query ComprehensiveDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    name
    
    # 1. SEMANTIC KPIs - Fast overview metrics
    overviewKPIs: devicesFiltered(all: true) {
      total
      online_total: total(online: true)
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: AVG
      )
      maxTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: MAX
      )
      avgHumidity: aggregatedNumericSemanticValue(
        semantic: HUMIDITY
        aggregation: AVG
      )
      avgBattery: aggregatedNumericSemanticValue(
        semantic: BATTERY
        aggregation: AVG
      )
    }
    
    # 2. ROLE FIELDS - Multi-product device display
    multiProductDevices: devicesFiltered(
      pageSize: 10
      orderBy: { lastHeard: DESC }
    ) {
      devices {
        id
        verboseName
        online
        product { name }
        roleFields {
          value
          role
          field {
            fieldName
            unit
          }
        }
      }
    }
    
    # 3. SEMANTIC DEVICES - Cross-device compatibility
    semanticDevices: devicesFiltered(
      temperature: { gt: -50.0 }  # Has temperature data
      pageSize: 10
    ) {
      devices {
        id
        verboseName
        online
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
        humidity: numericSemanticField(semantic: HUMIDITY) {
          value
        }
        battery: numericSemanticField(semantic: BATTERY) {
          value
        }
      }
    }
    
    # 4. HARDCODED - Specific product with known fields
    soilSensors: devicesFiltered(
      tags: { contains: ["Soil"] }
      pageSize: 5
    ) {
      devices {
        id
        verboseName
        currentMeasurements(fieldNames: ["SOIL_MOISTURE", "SOIL_TEMPERATURE"]) {
          value
          field {
            fieldName
            unit
          }
        }
      }
    }
  }
}
```

### 6. Semantic Query Best Practices

✅ **Do:**
- Use semantic KPIs for dashboard metrics and overview panels
- Handle null values gracefully (devices may not have all semantics)
- Use `aggregation: AVG` for multiple fields with same semantic
- Cache semantic KPI results for performance
- Use semantic queries for cross-device analytics

❌ **Don't:**
- Use field details (`fields { ... }`) in overview queries (performance impact)
- Assume all devices have all semantics assigned
- Use semantics for historical data queries (not supported)
- Skip null checking in frontend processing

### 7. Performance Considerations

```graphql
# ✅ FAST: KPIs only
devicesFiltered(all: true) {
  avgTemperature: aggregatedNumericSemanticValue(semantic: TEMPERATURE)
}

# ✅ GOOD: Semantic values with pagination
devicesFiltered(pageSize: 20) {
  devices {
    temperature: numericSemanticField(semantic: TEMPERATURE) {
      value
    }
  }
}

# ⚠️ SLOWER: Field details (use sparingly)
devicesFiltered(pageSize: 5) {
  devices {
    temperature: numericSemanticField(semantic: TEMPERATURE) {
      value
      fields {  # Heavy - avoid in overview queries
        fieldName
        unit
        value
      }
    }
  }
}
```

### Key Benefits of Semantic Queries

1. **Cross-Device Compatibility** - Works regardless of underlying field names
2. **Backend Flexibility** - Administrators can change field mappings without code updates
3. **Standardized Metrics** - Consistent KPIs across different device types
4. **Future-Proof** - New devices automatically work if semantics are assigned
5. **Aggregation Support** - Built-in AVG, MAX, MIN, SUM across all devices

---

## Key Performance Patterns

### The Count vs Details Pattern

**Always separate count queries from detail queries for optimal performance:**

```graphql
query OptimalDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # FAST: Counts only
    totalDevices: devicesFiltered(all: true) { total }
    onlineDevices: devicesFiltered(online: true) { total }
    offlineDevices: devicesFiltered(online: false) { total }
    lowBattery: devicesFiltered(battery: { lt: 20.0 }) { total }
    
    # PAGINATED: Details when needed
    criticalDevices: devicesFiltered(
      battery: { lt: 10.0 }
      pageSize: 5  # Always limit
    ) {
      total
      devices {
        id
        verboseName
        # ... details
      }
    }
  }
}
```

### Why This Matters

- **Count-only queries** return instantly, even with thousands of devices
- **Device detail queries** without pagination can return 100s-1000s of records
- **Alert filters** (low battery, offline, etc.) often match many devices
- **Dashboard performance** depends on separating these concerns

---

## Next Steps

This guide covers the foundational approaches for building device overviews. The next phase will cover:

1. **Dynamic semantic queries** for cross-device compatibility
2. **Advanced role field configurations** 
3. **Real-time updates** and WebSocket integration
4. **Error handling** and offline scenarios
5. **Performance optimization** for large-scale deployments

Use this guide as your foundation for building robust, scalable device overview interfaces with Datacake's API. 