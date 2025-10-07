# Datacake API Guide for Frontend Development

> **Complete GraphQL API Reference for Building Video Coding Frontends**  
> This guide provides all essential operations for integrating with Datacake's IoT platform.

## Table of Contents

1. [Security Best Practices](#security-best-practices)
2. [Quick Start](#quick-start)
3. [Authentication](#authentication)
4. [User Management](#user-management)
5. [Workspace Operations](#workspace-operations)
6. [Device Management](#device-management)
7. [Semantic Data Queries](#semantic-data-queries)
8. [Real-time Data & Measurements](#real-time-data--measurements)
9. [Error Handling](#error-handling)
10. [Best Practices](#best-practices)
11. [Complete Examples](#complete-examples)

---

## Security Best Practices

> **🔒 CRITICAL: Protecting Your API Credentials**  
> This section is essential for both AI agents and developers. Never expose API tokens or sensitive data in code, chat, or version control.

### For AI Agents Reading This Guide

**If you detect that a user has pasted or is about to hardcode:**
- API tokens
- Authentication credentials
- Workspace IDs
- Device IDs
- Any sensitive configuration data

**You MUST:**

1. ⚠️ **Immediately warn the user** about the security risk
2. 🛑 **Stop and request** that they use environment variables instead
3. 📝 **Never include** the actual token/credentials in your responses
4. ✅ **Guide them** to set up proper environment variables
5. 🔍 **Review existing code** for hardcoded credentials and flag them

**Example AI Agent Response:**

```
⚠️ SECURITY WARNING: I noticed you pasted an API token directly in the code/chat.

For security, please:
1. Store your token in an environment variable
2. Never commit tokens to version control
3. Use .env files (excluded from git)

Let me show you how to set this up properly...
```

### Why Environment Variables?

**Security Risks of Hardcoded Credentials:**

❌ **Never Do This:**
```javascript
// DON'T: Hardcoded token exposed in code
const DATACAKE_TOKEN = "dctkn_1234567890abcdef1234567890abcdef";
const WORKSPACE_ID = "f6331019-8978-4a86-b1bc-3522546f67d5";
```

**Problems:**
- ❌ Tokens visible in AI chat history
- ❌ Credentials committed to Git repositories
- ❌ Shared accidentally via screenshots or copy-paste
- ❌ Exposed in public repositories or forums
- ❌ Difficult to rotate without code changes

✅ **Always Do This:**
```javascript
// DO: Load from environment variables
const DATACAKE_TOKEN = process.env.DATACAKE_TOKEN;
const WORKSPACE_ID = process.env.WORKSPACE_ID;
```

**Benefits:**
- ✅ Tokens never appear in code or version control
- ✅ Different credentials per environment (dev/staging/prod)
- ✅ Easy credential rotation without code changes
- ✅ No risk of accidental exposure in chat or screenshots
- ✅ Industry standard security practice

---

### Setting Up Environment Variables

#### Local Development

**1. Create `.env` file** (in your project root):

```bash
# .env
DATACAKE_TOKEN=dctkn_your_actual_token_here
WORKSPACE_ID=your-workspace-id-here
DEVICE_ID=your-device-id-here
```

**2. Add `.env` to `.gitignore`:**

```bash
# .gitignore
.env
.env.local
.env.*.local
*.env
node_modules/
```

**3. Load environment variables in your code:**

**Node.js / JavaScript:**
```javascript
// Install dotenv package first: npm install dotenv
require('dotenv').config();

const DATACAKE_TOKEN = process.env.DATACAKE_TOKEN;
const WORKSPACE_ID = process.env.WORKSPACE_ID;

// Validate that required env vars are set
if (!DATACAKE_TOKEN || !WORKSPACE_ID) {
  console.error('ERROR: Missing required environment variables');
  console.error('Please create a .env file with:');
  console.error('  DATACAKE_TOKEN=your_token');
  console.error('  WORKSPACE_ID=your_workspace_id');
  process.exit(1);
}

// Use in API calls
const headers = {
  'Authorization': `Token ${DATACAKE_TOKEN}`,
  'Content-Type': 'application/json'
};
```

**Python:**
```python
# Install python-dotenv: pip install python-dotenv
from dotenv import load_dotenv
import os

load_dotenv()

DATACAKE_TOKEN = os.getenv('DATACAKE_TOKEN')
WORKSPACE_ID = os.getenv('WORKSPACE_ID')

# Validate
if not DATACAKE_TOKEN or not WORKSPACE_ID:
    raise ValueError('Missing required environment variables. Check .env file.')

# Use in API calls
headers = {
    'Authorization': f'Token {DATACAKE_TOKEN}',
    'Content-Type': 'application/json'
}
```

---

#### Replit Environment

**1. Use Replit Secrets (NOT .env files):**

Replit provides a built-in secure secrets manager:

1. Click the **🔒 Secrets** tab in the left sidebar (or Tools → Secrets)
2. Add your secrets:
   - Key: `DATACAKE_TOKEN` → Value: `dctkn_your_actual_token`
   - Key: `WORKSPACE_ID` → Value: `your-workspace-id`
3. Click **Add new secret** for each variable

**2. Access in code:**

```javascript
// Node.js on Replit
const DATACAKE_TOKEN = process.env.DATACAKE_TOKEN;
const WORKSPACE_ID = process.env.WORKSPACE_ID;
```

```python
# Python on Replit
import os

DATACAKE_TOKEN = os.environ['DATACAKE_TOKEN']
WORKSPACE_ID = os.environ['WORKSPACE_ID']
```

**Important:** Replit Secrets are:
- ✅ Encrypted and secure
- ✅ Never exposed in the editor
- ✅ Not committed to version control
- ✅ Private to you (not shared with Replit forks)

---

#### Docker Environment

**1. Using `.env` file with Docker Compose:**

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    env_file:
      - .env
    environment:
      - DATACAKE_TOKEN=${DATACAKE_TOKEN}
      - WORKSPACE_ID=${WORKSPACE_ID}
```

**2. Using environment variables directly:**

```bash
# Run with environment variables
docker run -e DATACAKE_TOKEN=${DATACAKE_TOKEN} \
           -e WORKSPACE_ID=${WORKSPACE_ID} \
           your-app
```

---

#### Production Deployment

**Never use `.env` files in production.** Use your platform's secret management:

| Platform | Secret Management |
|----------|-------------------|
| **Vercel** | Environment Variables in project settings |
| **Netlify** | Environment Variables in site settings |
| **AWS** | AWS Secrets Manager or Parameter Store |
| **Google Cloud** | Secret Manager |
| **Azure** | Key Vault |
| **Heroku** | Config Vars in app settings |
| **Railway** | Environment Variables in project settings |

---

### Complete Secure Implementation Example

**Secure Node.js Setup:**

```javascript
// config.js - Environment configuration
require('dotenv').config();

const config = {
  datacake: {
    apiUrl: 'https://api.datacake.co/graphql/',
    token: process.env.DATACAKE_TOKEN,
    workspaceId: process.env.WORKSPACE_ID
  }
};

// Validation
const requiredEnvVars = ['DATACAKE_TOKEN', 'WORKSPACE_ID'];
const missingVars = requiredEnvVars.filter(varName => !process.env[varName]);

if (missingVars.length > 0) {
  console.error('❌ Missing required environment variables:');
  missingVars.forEach(varName => console.error(`   - ${varName}`));
  console.error('\n📝 Create a .env file with:');
  requiredEnvVars.forEach(varName => console.error(`   ${varName}=your_value_here`));
  process.exit(1);
}

module.exports = config;
```

```javascript
// datacakeClient.js - API client
const config = require('./config');

async function queryDatacake(query, variables = {}) {
  const response = await fetch(config.datacake.apiUrl, {
    method: 'POST',
    headers: {
      'Authorization': `Token ${config.datacake.token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ query, variables })
  });
  
  return response.json();
}

module.exports = { queryDatacake };
```

```javascript
// app.js - Your application
const { queryDatacake } = require('./datacakeClient');
const config = require('./config');

async function getDevices() {
  const query = `
    query GetDevices($workspaceId: String!) {
      workspace(id: $workspaceId) {
        devices {
          id
          verboseName
          online
        }
      }
    }
  `;
  
  const result = await queryDatacake(query, {
    workspaceId: config.datacake.workspaceId
  });
  
  return result.data.workspace.devices;
}

// Use it
getDevices().then(devices => {
  console.log('Devices:', devices);
});
```

---

### Security Checklist

Before deploying or sharing your code, verify:

- [ ] **No hardcoded tokens** in any source files
- [ ] **`.env` file in `.gitignore`** (check with `git status`)
- [ ] **Environment variables documented** in README
- [ ] **Validation logic** for missing environment variables
- [ ] **Separate credentials** for dev/staging/production
- [ ] **Token rotation plan** in place
- [ ] **No credentials in** commit history (check with `git log`)
- [ ] **No credentials shared** in AI chat or screenshots

---

### Creating a `.env.example` Template

Provide a template for other developers (without actual credentials):

```bash
# .env.example
# Copy this file to .env and fill in your actual values
# DO NOT commit .env to version control

# Datacake API Configuration
DATACAKE_TOKEN=dctkn_your_token_here
WORKSPACE_ID=your-workspace-id-here

# Optional: Device IDs for testing
DEVICE_ID=your-device-id-here
```

**Usage instructions in README:**

```markdown
## Setup

1. Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and add your Datacake credentials:
   - Get your token from: https://app.datacake.de
   - Navigate to Settings → API → Generate Token

3. Never commit `.env` to version control
```

---

### What to Do If Credentials Are Exposed

If you accidentally expose your API token:

1. **🚨 Immediately revoke the token:**
   - Go to https://app.datacake.de
   - Navigate to Settings → API
   - Delete the exposed token

2. **🔑 Generate a new token:**
   - Create a new API token
   - Update your `.env` file with the new token

3. **🧹 Clean up exposure:**
   - Remove from chat history if possible
   - If committed to Git, consider rewriting history or rotating all credentials
   - If posted publicly, delete immediately

4. **📝 Review security practices:**
   - Ensure `.env` is in `.gitignore`
   - Check for other exposed credentials
   - Implement the practices in this guide

---

### For AI Agents: Detection Patterns

Watch for these patterns and warn users:

**Hardcoded Token Patterns:**
```javascript
// Pattern 1: Direct string
const token = "dctkn_[a-f0-9]{32}";

// Pattern 2: In fetch/axios calls
headers: { 'Authorization': 'Token dctkn_...' }

// Pattern 3: In config objects
config = { token: "dctkn_..." }
```

**Hardcoded ID Patterns:**
```javascript
// Pattern 1: UUID format
const workspaceId = "f6331019-8978-4a86-b1bc-3522546f67d5";

// Pattern 2: In GraphQL variables
variables: { workspaceId: "f6331019-..." }
```

**When detected, respond with:**
```
⚠️ SECURITY WARNING: I detected a hardcoded credential in your code.

This is a security risk. Let's fix it by using environment variables:

1. Create a .env file (I'll show you how)
2. Store your credentials there
3. Load them using process.env or os.getenv()
4. Add .env to .gitignore

Would you like me to help you set this up properly?
```

---

## Quick Start

### API Endpoint
```
https://api.datacake.co/graphql/
```

### Authentication Method
HTTP Header: `Authorization: Token YOUR_TOKEN_HERE`

### Essential Flow
1. **Login** → Get authentication token
2. **Get User** → Verify authentication  
3. **List Workspaces** → Choose workspace to work with
4. **Query Devices** → Get device data from workspace
5. **Fetch Measurements** → Get real-time or historical data

---

## Authentication

### Login Mutation

**Request:**
```graphql
mutation LoginUser {
  login(
    email: "youruser@email.com"
    password: "yourpassword"
  ) {
    ok
    token
    user {
      id
      firstName
      lastName
      email
      fullName
    }
    error
  }
}
```

**Successful Response:**
```json
{
  "data": {
    "login": {
      "ok": true,
      "token": "your-authentication-token-here",
      "user": {
        "id": "e9f090c1-1d10-4705-b840-34352b966916",
        "firstName": "John",
        "lastName": "Doe",
        "email": "john.doe@example.com",
        "fullName": "John Doe"
      },
      "error": null
    }
  }
}
```

**Error Response:**
```json
{
  "data": {
    "login": {
      "ok": false,
      "token": null,
      "user": null,
      "error": "Invalid credentials"
    }
  }
}
```

### Setting Authentication Header

After successful login, include the token in all subsequent requests:

```javascript
// HTTP Headers
{
  "Authorization": "Token your-authentication-token-here",
  "Content-Type": "application/json"
}
```

---

## User Management

### Get Current User

**Query:**
```graphql
query GetCurrentUser {
  user {
    id
    firstName
    lastName
    email
    language
    phoneNumber
    fullName
    primaryWorkspace {
      id
      name
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "id": "e9f090c1-1d10-4705-b840-34352b966916",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com",
      "language": "EN",
      "phoneNumber": "+1234567890",
      "fullName": "John Doe",
      "primaryWorkspace": {
        "id": "f6331019-8978-4a86-b1bc-3522546f67d5",
        "name": "Main Workspace"
      }
    }
  }
}
```

---

## Workspace Operations

### List All Workspaces

**Query:**
```graphql
query GetAllWorkspaces {
  allWorkspaces {
    id
    name
    slug
    deviceCount
    memberCount
    logo
    myPermissions
  }
}
```

**Response:**
```json
{
  "data": {
    "allWorkspaces": [
      {
        "id": "f6331019-8978-4a86-b1bc-3522546f67d5",
        "name": "Main Workspace",
        "slug": "main-workspace",
        "deviceCount": 42,
        "memberCount": 5,
        "logo": null,
        "myPermissions": ["basics", "devices", "rules"]
      },
      {
        "id": "67e01a42-8c0d-4f12-aa06-e935727e5b14",
        "name": "Test Environment",
        "slug": "test-environment",
        "deviceCount": 3,
        "memberCount": 2,
        "logo": null,
        "myPermissions": ["basics", "devices"]
      }
    ]
  }
}
```

### Get Single Workspace Details

**Query:**
```graphql
query GetWorkspaceDetails($workspaceId: String!) {
  workspace(id: $workspaceId) {
    id
    name
    slug
    deviceCount
    memberCount
    users {
      id
      firstName
      lastName
      email
      fullName
    }
    myPermissions
    features
    allTags
  }
}
```

**Variables:**
```json
{
  "workspaceId": "f6331019-8978-4a86-b1bc-3522546f67d5"
}
```

---

## Device Management

### List All Devices in Workspace

**Query:**
```graphql
query GetWorkspaceDevices($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devices {
      id
      verboseName
      serialNumber
      online
      lastHeard
      location
      tags
      icon
      product {
        id
        name
        hardware
      }
    }
  }
}
```

### Advanced Device Filtering

**Query:**
```graphql
query GetFilteredDevices($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      online: true
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
        location
        tags
        currentLocation {
          lat
          lng
        }
        product {
          name
          hardware
        }
      }
    }
  }
}
```

### Get Single Device Details

**Query:**
```graphql
query GetDeviceDetails($workspaceId: String!, $deviceId: String!) {
  workspace(id: $workspaceId) {
    device(id: $deviceId) {
      id
      verboseName
      serialNumber
      online
      lastHeard
      location
      tags
      metadata
      softwareVersion
      icon
      image
      features
      currentLocation {
        lat
        lng
      }
      product {
        id
        name
        hardware
        measurementFields {
          id
          fieldName
          verboseFieldName
          unit
          fieldType
          active
        }
      }
    }
  }
}
```

---

## Semantic Data Queries

### Available Semantics

```graphql
enum FieldSemantic {
  AIR_POLLUTION
  AMBIENT_LIGHT
  BATTERY
  CO2
  ENERGY_CONSUMPTION
  FILL_LEVEL
  HUMIDITY
  LOCATION
  LOUDNESS
  PEOPLE_COUNT
  POWER
  SIGNAL
  SOIL_MOISTURE
  TEMPERATURE
  VOC
  WATER_CONSUMPTION
  WATER_DEPTH
}
```

### Get Workspace Semantics

**Query:**
```graphql
query GetWorkspaceSemantics($workspaceId: String!) {
  workspace(id: $workspaceId) {
    semantics
  }
}
```

### Filter Devices by Semantic Values

**Query:**
```graphql
query GetDevicesByTemperature($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      temperature: { gt: 20.0, lt: 30.0 }
      online: true
    ) {
      total
      avgTemperature: aggregatedNumericSemanticValue(
        semantic: TEMPERATURE
        aggregation: AVG
      )
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
      }
    }
  }
}
```

### Multi-Semantic Environmental Monitoring

**Query:**
```graphql
query GetEnvironmentalData($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      temperature: { gt: -50.0 }
      humidity: { gt: 0.0 }
      co2: { gt: 300.0 }
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
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2
        aggregation: AVG
      )
      devices {
        id
        verboseName
        location
        tags
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
          fields {
            verboseFieldName
            value
            unit
          }
        }
        humidity: numericSemanticField(semantic: HUMIDITY) {
          value
          fields {
            verboseFieldName
            value
            unit
          }
        }
        co2: numericSemanticField(semantic: CO2) {
          value
          fields {
            verboseFieldName
            value
            unit
          }
        }
        battery: numericSemanticField(semantic: BATTERY) {
          value
          fields {
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

---

## Real-time Data & Measurements

### Get Field Identifiers

**Important**: Before querying specific measurements or history, you need to know the exact field names available for a device.

**Best Practice for Frontend Development**: **HARDCODE field identifiers** in your application rather than fetching them dynamically. This avoids extra queries and potential issues when building frontends for specific use cases.

**Ways to discover field identifiers:**

1. **Via Datacake Web Frontend** (recommended): Go to [app.datacake.de](https://app.datacake.de), navigate to your device, and check the device configuration to see all available field identifiers, then hardcode them in your app
2. **Via GraphQL API** (for initial discovery or dynamic systems):

**Query:**
```graphql
query GetFieldIdentifiers($workspaceId: String!, $deviceId: String!) {
  workspace(id: $workspaceId) {
    device(id: $deviceId) {
      id
      verboseName
      product {
        measurementFields {
          id
          fieldName
          verboseFieldName
          unit
          fieldType
          semantic
          active
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
      "device": {
        "id": "d01eb653-aea2-4d79-a1fb-2faaab90e8d2",
        "verboseName": "CO2 - Entrance",
        "product": {
          "measurementFields": [
            {
              "id": "349eca8d-569d-4c66-ad36-a14fa20f11dd",
              "fieldName": "TEMPERATURE",
              "verboseFieldName": "Temperature",
              "unit": "°C",
              "fieldType": "FLOAT",
              "semantic": "TEMPERATURE",
              "active": true
            },
            {
              "id": "14f9a4db-290a-454e-b5ce-48a1ed42a9b5",
              "fieldName": "HUMIDITY",
              "verboseFieldName": "Humidity",
              "unit": "%",
              "fieldType": "FLOAT",
              "semantic": "HUMIDITY",
              "active": true
            },
            {
              "id": "e3c26b8f-06fa-4080-af68-d4b01d9ff6d9",
              "fieldName": "CO2",
              "verboseFieldName": "CO2",
              "unit": "ppm",
              "fieldType": "FLOAT",
              "semantic": "CO2",
              "active": true
            }
          ]
        }
      }
    }
  }
}
```

### Get Current Device Measurements

**Query:**
```graphql
query GetDeviceMeasurements($workspaceId: String!, $deviceId: String!) {
  workspace(id: $workspaceId) {
    device(id: $deviceId) {
      id
      verboseName
      # Get all current measurements
      currentMeasurements {
        fieldName
        verboseFieldName
        value
        unit
        fieldType
        timestamp
      }
      # Get specific measurements only
      currentMeasurements(fieldNames: ["TEMPERATURE", "HUMIDITY", "CO2"]) {
        fieldName
        verboseFieldName
        value
        unit
        fieldType
        timestamp
      }
    }
  }
}
```

**Important Notes:**
- Use **field names** (like "TEMPERATURE", "CO2") for `currentMeasurements(fieldNames: [...])` - **HARDCODE these in your app**
- **Semantics limitation**: `currentMeasurements()` does NOT support semantic field names, only exact field identifiers
- Use **semantic names** (like `TEMPERATURE`, `CO2`) only for cross-device filtering in `devicesFiltered()`
- For frontend development: Discover field names via the Datacake web interface, then hardcode them

### Get Historical Data

**Query:**
```graphql
query GetDeviceHistory($workspaceId: String!, $deviceId: String!) {
  workspace(id: $workspaceId) {
    device(id: $deviceId) {
      id
      verboseName
      history(
        fields: ["CO2", "TEMPERATURE", "HUMIDITY"]  # Optional: filter specific fields
        timerangestart: "2025-01-01T00:00:00Z"
        timerangeend: "2025-01-02T00:00:00Z"
        resolution: "1h"  # Bucketing size - crucial for performance
      )
    }
  }
}
```

**Important Notes:**
- **Fields filtering**: Use exact **field names** like `fields: ["TEMPERATURE", "CO2"]` - **HARDCODE these in your frontend app**
- **Semantics limitation**: `history()` and `currentMeasurements()` do NOT support semantic field names, only exact field identifiers
- **Resolution is crucial**: This determines the bucketing size and affects performance:
  - For 1 day: use `"15m"` or `"1h"`
  - For 1 week: use `"1h"` or `"6h"`
  - For 1 month: use `"24h"` (1 day buckets)
  - Higher resolution = less data points = better performance
- **Aggregation**: Buckets are aggregated using **AVERAGE only** (no sum, min, max options)
- **Field Names vs Semantics**: Use field names for `history()` and `currentMeasurements()`, use semantics only for filtering in `devicesFiltered()`

**Examples by Time Range:**

```graphql
# Last 24 hours - high resolution
history(
  fields: ["CO2"]
  timerangestart: "2025-01-01T00:00:00Z"
  timerangeend: "2025-01-02T00:00:00Z"
  resolution: "15m"
)

# Last week - medium resolution
history(
  fields: ["TEMPERATURE", "HUMIDITY"]
  timerangestart: "2025-01-01T00:00:00Z"
  timerangeend: "2025-01-08T00:00:00Z"
  resolution: "1h"
)

# Last month - low resolution for performance
history(
  fields: ["CO2", "TEMPERATURE"]
  timerangestart: "2024-12-01T00:00:00Z"
  timerangeend: "2025-01-01T00:00:00Z"
  resolution: "24h"
)
```

### Get Device Dashboard Data

**Query:**
```graphql
query GetDeviceDashboard($workspaceId: String!, $deviceId: String!) {
  workspace(id: $workspaceId) {
    device(id: $deviceId) {
      id
      verboseName
      dashboardData
      product {
        dashboardConfig
      }
    }
  }
}
```

---

## Error Handling

### Common Error Types

```graphql
enum ErrorCode {
  NOT_AUTHENTICATED
  NOT_AUTHORIZED
  NOT_FOUND
  VALIDATION_ERROR
  CANNOT_PERFORM_OPERATION
  INVALID_BILLING_INFORMATION
  DEVICE_NOT_FOUND
  FIELD_DOES_NOT_EXIST
  INSUFFICIENT_QUOTA
  INVALID_AUTHENTICATION_CODE
  GATEWAY_ALREADY_CLAIMED
}
```

### Error Response Format

**Standard Error Response:**
```json
{
  "errors": [
    {
      "message": "Authentication required",
      "extensions": {
        "code": "NOT_AUTHENTICATED"
      }
    }
  ],
  "data": null
}
```

### Handling Authentication Errors

```javascript
// Example error handling
if (response.errors) {
  const authError = response.errors.find(
    error => error.extensions?.code === 'NOT_AUTHENTICATED'
  );
  
  if (authError) {
    // Redirect to login or refresh token
    console.log('Authentication required');
  }
}
```

---

## Best Practices

### 1. Authentication Management

```javascript
// Store token securely
localStorage.setItem('datacake_token', token);

// Include in all requests
const headers = {
  'Authorization': `Token ${localStorage.getItem('datacake_token')}`,
  'Content-Type': 'application/json'
};
```

### 2. Efficient Data Fetching

```graphql
# Good: Request only needed fields
query GetDeviceBasics {
  workspace(id: "workspace-id") {
    devices {
      id
      verboseName
      online
    }
  }
}

# Avoid: Requesting all fields when not needed
query GetDeviceEverything {
  workspace(id: "workspace-id") {
    devices {
      id
      verboseName
      serialNumber
      metadata
      lastHeard
      location
      # ... many fields slow down the query
    }
  }
}
```

### 3. Pagination for Large Datasets

```graphql
query GetDevicesPaginated($workspaceId: String!) {
  workspace(id: $workspaceId) {
    devicesFiltered(
      pageSize: 20
      page: 1
      orderBy: { verboseName: ASC }
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

### 4. Use Semantic Filters for Cross-Device Queries

```graphql
# Good: Semantic filtering works across device types
query GetAllTemperatureSensors {
  workspace(id: "workspace-id") {
    devicesFiltered(
      temperature: { gt: -100.0 }  # Filter by semantic
    ) {
      devices {
        id
        verboseName
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
      }
    }
  }
}
```

### 5. Efficient Tag-Based Filtering

```graphql
# Good: Use tags for device organization
query GetCO2Sensors {
  workspace(id: "workspace-id") {
    devicesFiltered(
      tags: { contains: ["CO2"] }
    ) {
      devices {
        id
        verboseName
        tags
      }
    }
  }
}
```

### 6. Field Names vs Semantics - Critical Distinction

**Field Names** (REQUIRED for specific data queries):
- For `history(fields: ["TEMPERATURE", "CO2"])` - **semantics NOT supported**
- For `currentMeasurements(fieldNames: ["HUMIDITY"])` - **semantics NOT supported**  
- **Best practice**: Hardcode these in your frontend application
- Device-specific, exact field identifiers

**Semantics** (use for cross-device filtering and generic overviews):
- For `devicesFiltered(temperature: { gt: 20.0 })`
- For `numericSemanticField(semantic: TEMPERATURE)`
- Cross-device compatible filtering
- Standardized meaning-based queries
- **Limitation**: NOT supported for `history()` or `currentMeasurements()`

```graphql
# CORRECT: Mixed usage example
query CorrectFieldUsage {
  workspace(id: "workspace-id") {
    # Use semantics for filtering
    devicesFiltered(temperature: { gt: 20.0 }) {
      devices {
        id
        verboseName
        # Use field names for specific measurements
        currentMeasurements(fieldNames: ["TEMPERATURE", "HUMIDITY"]) {
          fieldName
          value
        }
        # Use field names for history
        history(
          fields: ["CO2", "TEMPERATURE"]
          timerangestart: "2025-01-01T00:00:00Z"
          timerangeend: "2025-01-02T00:00:00Z"
          resolution: "1h"
        )
      }
    }
  }
}
```

---

## Complete Examples

### Building a Device Dashboard

```graphql
query BuildDeviceDashboard($workspaceId: String!) {
  workspace(id: $workspaceId) {
    name
    
    # Get all online devices
    onlineDevices: devicesFiltered(online: true) {
      total
      devices {
        id
        verboseName
        lastHeard
        location
        tags
      }
    }
    
    # Get environmental overview
    environmentalData: devicesFiltered(
      temperature: { gt: -50.0 }
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
      avgCO2: aggregatedNumericSemanticValue(
        semantic: CO2
        aggregation: AVG
      )
    }
    
    # Get devices needing attention
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
    
    # Get recent activity
    recentDevices: devicesFiltered(
      pageSize: 10
      orderBy: { lastHeard: DESC }
    ) {
      devices {
        id
        verboseName
        lastHeard
        online
      }
    }
  }
}
```

### Real-time Monitoring Query

```graphql
query RealTimeMonitoring($workspaceId: String!) {
  workspace(id: $workspaceId) {
    # Critical alerts
    highTemperature: devicesFiltered(
      temperature: { gt: 35.0 }
      online: true
    ) {
      total
      devices {
        id
        verboseName
        location
        temperature: numericSemanticField(semantic: TEMPERATURE) {
          value
        }
      }
    }
    
    # Poor air quality
    poorAirQuality: devicesFiltered(
      co2: { gt: 1000.0 }
      online: true
    ) {
      total
      devices {
        id
        verboseName
        location
        co2: numericSemanticField(semantic: CO2) {
          value
        }
      }
    }
    
    # Offline devices
    offlineDevices: devicesFiltered(
      online: false
    ) {
      total
      devices {
        id
        verboseName
        lastHeard
        location
      }
    }
  }
}
```

### Device Management Interface

```graphql
query DeviceManagementInterface($workspaceId: String!) {
  workspace(id: $workspaceId) {
    name
    deviceCount
    memberCount
    allTags
    
    # All devices with full details
    devices {
      id
      verboseName
      serialNumber
      online
      lastHeard
      location
      tags
      metadata
      softwareVersion
      features
      
      # Device capabilities
      product {
        id
        name
        hardware
        features
        measurementFields(active: true) {
          id
          fieldName
          verboseFieldName
          unit
          fieldType
          semantic
        }
      }
      
      # Current status
      currentMeasurements {
        fieldName
        verboseFieldName
        value
        unit
        timestamp
      }
      
      # Location data
      currentLocation {
        lat
        lng
      }
    }
  }
}
```

---

## Integration Tips for Video Coding Frontends

### 1. State Management
- Store authentication token in secure storage
- Cache workspace and device lists
- Use reactive queries for real-time updates

### 2. Performance Optimization
- Implement pagination for large device lists
- Use semantic filtering for efficient queries
- Request only needed fields to reduce payload size

### 3. Error Handling
- Implement retry logic for network failures
- Handle authentication errors gracefully
- Provide meaningful error messages to users

### 4. Real-time Updates
- Poll critical data every 30-60 seconds
- Use WebSocket connections for live dashboards
- Implement offline mode for network interruptions

### 5. User Experience
- Show loading states during data fetching
- Implement search and filter capabilities
- Provide clear device status indicators

---

This guide provides all essential operations for building comprehensive IoT frontend applications with Datacake. Use the semantic queries for cross-device compatibility and follow the best practices for optimal performance.