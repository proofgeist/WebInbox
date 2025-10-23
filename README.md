# WebInbox Module

## Overview

**WebInbox** is a FileMaker module designed to act as a **logging and replay interface** for incoming **FileMaker Data API calls**.  
It provides a transparent and auditable way to:

- Receive API payloads as **records** in the `WebInbox` table  
- Automatically trigger a FileMaker **script** to handle each incoming payload  
- Log both the **request** and the **result** for later review or reprocessing  

This module makes your Data API integrations **traceable**, **rerunnable**, and **debuggable**—ideal for complex integrations or asynchronous workflows.

---

## Architecture

### Table: `WebInbox`

The `WebInbox` table serves as a **queue** and **log** of all inbound API requests.

| Field | Type | Description |
|-------|------|--------------|
| **PrimaryKey** | Text | Auto-enter UUID. Primary unique identifier for the inbox record. |
| **CreationTimestamp** | Timestamp | Auto-enter creation timestamp. |
| **CreatedBy** | Text | Auto-enter account name at record creation. |
| **ModificationTimestamp** | Timestamp | Auto-enter modification timestamp. |
| **ModifiedBy** | Text | Auto-enter account name at modification. |
| **inboxPayload** | Text | The JSON or text payload sent by the external API request. |
| **scriptName** | Text | The name of the FileMaker script to execute using the payload. |
| **scriptResult** | Text | The result returned by the executed script. |
| **ONE** | Number | Constant (auto-enter = 1) used for global relationships. |
| **RunTime** | Number | Runtime of the script execution, in seconds. |

---

## Script: `Run Script from Inbox`

**Do not edit this script.**  
This script is automatically called by the Data API after a record is created in `WebInbox`.

### Purpose
To:
1. Run the specified script listed in the inbox record (`Inbox::scriptName`)  
2. Pass the API payload (`Inbox::inboxPayload`) as a script parameter  
3. Log the script’s result and execution time

### Script Definition
```filemaker
# DO NOT EDIT THIS SCRIPT
# PURPOSE: Web API will create a record in the Inbox, then call this script to process the API call
Allow User Abort [ Off ]
Set Error Capture [ On ]

# Run script safely in its own context
New Window [ Style: Document ; Using layout: <Current Layout> ]
Set Variable [ $startTime ; Value: Get ( CurrentTimeUTCMilliseconds ) ]

Perform Script [ Specified: By name ; Inbox::scriptName ; Parameter: Inbox::inboxPayload ]

Close Window [ Current Window ]

# Capture timing and results
Set Variable [ $runTime ; Value: ( Get ( CurrentTimeUTCMilliseconds ) - $startTime ) / 1000 ]
Set Variable [ $result ; Value: Get ( ScriptResult ) ]

# Log the result
Set Field [ Inbox::scriptResult ; $result ]
Set Field [ Inbox::RunTime ; $runTime ]
Commit Records/Requests [ With dialog: Off ]

Exit Script [ Text Result: $result ]
```

---

## How It Works

### Step 1: API Request Creates an Inbox Record

External clients (such as webhooks or other systems) create a new record in `WebInbox` via the **FileMaker Data API** using the `create record` endpoint.

- API reference:  
  - [Create Record (Data API Guide)](https://help.claris.com/en/data-api-guide/content/create-record.html)

Example (simplified `curl` syntax):
```bash
curl -X POST https://<server>/fmi/data/v2/databases/<dbname>/layouts/WebInbox/records   -H "Authorization: Bearer <token>"   -H "Content-Type: application/json"   -d '{
    "fieldData": {
      "inboxPayload": "{ "customerId": 42, "status": "active" }",
      "scriptName": "Process Customer Update"
    },
    "script": "Run Script from Inbox"
  }'
```

### Step 2: FileMaker Executes the Script

Once the record is created, FileMaker immediately calls the `Run Script from Inbox` script (as specified in the request).

That script:
1. Runs the named script stored in `scriptName`
2. Passes the `inboxPayload` as the parameter
3. Captures the script’s output and runtime
4. Logs the result back into the record

---

## User Interface

The layout shown in this module (`Web Inbox`) provides an operator-friendly view of the inbox:

- **Payload Received Timestamp:** shows when the API payload arrived  
- **Inbox Payload:** raw request data  
- **Script to Run / Script Result:** clear before-and-after logging  
- **Re-Run Script** button: allows manual reprocessing of a previous request  
- **Open in VS Code** buttons: open payloads or results in a text editor for analysis

---

## Usage Guidelines

- Treat each API payload as **immutable input**—do not overwrite `inboxPayload` after creation.
- Use **Re-Run Script** only for replay or debugging scenarios.
- Ensure all target scripts are **idempotent** (i.e., safe to run multiple times) since reruns can occur.
- Limit payload size and avoid storing binary data directly.

---

## Benefits

- **Traceability:** Each inbound API request and its result are permanently logged.
- **Resilience:** Errors don’t interrupt the API; they’re captured for review.
- **Replayability:** Requests can be rerun at any time for troubleshooting.
- **Simplicity:** Uses standard FileMaker Data API endpoints—no plugins or middleware required.

---

## References

- [FileMaker Data API – Run Script with Another Request](https://help.claris.com/en/data-api-guide/content/run-script-with-another-request.html)  
- [FileMaker Data API – Create Record](https://help.claris.com/en/data-api-guide/content/create-record.html)
