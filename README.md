# NICE CXone Connector for Camunda

Orchestrate your NICE CXone contact center directly from Camunda workflows! The NICE CXone Connector lets you manage agents, create work items, query real-time queue data, pull reporting metrics, and send digital messages — all without writing code.

```
Camunda 8 Cluster
+-----------+     +--------------+     +------------------+
|   BPMN    |---->|    REST      |---->|   NICE CXone     |
|  Process  |     |  Connector   |     |   APIs (v31.0)   |
|  (Zeebe)  |<----|  Runtime     |<----|                  |
+-----------+     +--------------+     +------------------+
      |                                        |
+-----+-------+                  +-------------+--------+
| Element     |                  | CXone Platform       |
| Templates   |                  |  - Admin API         |
| (JSON)      |                  |  - Patron API        |
| - no code!  |                  |  - Agent API         |
+-----------  +                  |  - Real-Time Data    |
                                 |  - Reporting API     |
                                 |  - Digital API       |
                                 +----------------------+
```

## What's Included

| File | Description |
|---|---|
| [`element-templates/nicecxone-connector.json`](../element-templates/nicecxone-connector.json) | **Main connector** — Multi-operation template covering 7 CXone API families with 22+ operations |
| [`element-templates/nicecxone-create-work-item.json`](../element-templates/nicecxone-create-work-item.json) | **Focused template** — Single-purpose: create a CXone work item or callback via the Patron API |
| [`element-templates/nicecxone-get-agent-status.json`](../element-templates/nicecxone-get-agent-status.json) | **Focused template** — Single-purpose: query real-time agent states and skill queue data |

## Prerequisites

1. A [NICE CXone](https://www.nice.com/products/cxone) account with API access (Enterprise edition or above).
2. A registered **backend application** with NICE to obtain your `client_id` and `client_secret`. See [NICE's Application Registration](https://developer.niceincontact.com/Documentation/GettingStarted) for details.
3. An **API Employee** in CXone Admin with generated access keys (`access_key_id` / `access_key_secret`).
4. Camunda Modeler, available in two variants:
   - [Web Modeler](https://modeler.camunda.io/) for an online experience with a Camunda SaaS account (recommended).
   - [Desktop Modeler](https://camunda.com/download/modeler/) for a local installation.

### Configuring Camunda Modeler

1. Install the Camunda Modeler if not already done.
2. Add the element template `.json` files to your Modeler configuration as per the [Element Templates documentation](https://docs.camunda.io/docs/components/modeler/desktop-modeler/element-templates/configuring-templates/).

## Installing the CXone Connector

This connector is a template connector based on the Camunda REST connector, so no compilation is required. To install and use it in your Camunda project:

1. **Download or copy the element templates**
   - Locate the `.json` files in the `element-templates/` directory of this repository.
   - **Web Modeler (recommended)**: Go to a project, click **More** > **Upload files**, and upload the `.json` files. The templates appear when you select a service task in any BPMN diagram.
   - **Desktop Modeler**: Copy the `.json` files to your Modeler's element templates directory:
     - **macOS**: `~/Library/Application Support/camunda-modeler/resources/element-templates/`
     - **Windows**: `%APPDATA%/camunda-modeler/resources/element-templates/`
     - **Linux**: `~/.config/camunda-modeler/resources/element-templates/`

2. **Ensure the REST connector is enabled**
   - For Camunda SaaS, it is available by default. For self-managed, follow the [REST connector installation guide](https://docs.camunda.io/docs/components/connectors/protocol/rest/).

3. **Use the CXone Connector in your BPMN model**
   - In Camunda Modeler, select a service task, choose one of the NICE CXone connector templates from the template catalog, configure it, and deploy your process.

No build or compilation steps are necessary — just add the templates and start automating with NICE CXone!

## Authentication

All templates support two authentication methods:

### Option A — Bearer Token (Quick Start)

Get a token using the CXone OAuth flow, then store it as a Camunda secret:

```
CXone Bearer Token: {{secrets.CXONE_TOKEN}}
```

> Tokens expire (typically 24 hours). For production use, implement a token refresh mechanism or use OAuth 2.0 Client Credentials (Option B).

### Option B — OAuth 2.0 Client Credentials

Select **OAuth 2.0 Client Credentials** in the Authentication Type dropdown and configure:

| Field | Value |
|---|---|
| OAuth Token Endpoint | `{{secrets.CXONE_TOKEN_ENDPOINT}}` |
| Client ID | `{{secrets.CXONE_CLIENT_ID}}` |
| Client Secret | `{{secrets.CXONE_CLIENT_SECRET}}` |
| Client Authentication | Send as Basic Auth header |

Both methods require the **CXone API Base URL**:

| Secret | Example Value |
|---|---|
| `CXONE_BASE_URL` | `https://api-na1.niceincontact.com` |

Store all credentials in **Camunda Console > Clusters > [your cluster] > Connector Secrets**.

## Configuring the Main Connector (`nicecxone-connector.json`)

The main connector supports all CXone API families via dropdown selection:

### API Groups & Operations

| API Group | Operations |
|---|---|
| **Admin** | Get Agents, Get Agent by ID, Get Skills, Get Teams, Update Agent Skills, Get Contacts |
| **Patron** | Create Work Item, Request Callback, Start Chat |
| **Agent** | Start Session, Get Session, Set Agent State, End Session |
| **Real-Time Data** | Get Agent States, Skill Queue Summary, Agent Activity |
| **Reporting** | Contact History, Performance Summary, Custom Report |
| **Digital Engagement** | Send Message, Get Conversation, Create Digital Contact |
| **Custom** | Build your own endpoint (any method, any path) |

### Parameters

- **CXone API Base URL** — Your regional CXone API endpoint (e.g. `https://api-na1.niceincontact.com`). Store as `{{secrets.CXONE_BASE_URL}}`.
- **API Group** — Select the CXone API family from the dropdown.
- **Operation** — Select the specific operation within the chosen API group.
- **API Version** — Defaults to `v31.0`.
- **API Path** — Auto-populated for built-in operations. Enter your own for Custom operations.
- **HTTP Method** — Auto-set for built-in operations, configurable for Custom.
- **Work Item Payload / Body** *(optional)* — JSON payload for POST/PUT operations. Supports FEEL expressions.
- **Query Parameters** *(optional)* — Additional query params as a FEEL context (e.g. `{updatedSince: "2024-01-01"}`).
- **Custom Headers** *(optional)* — Additional HTTP headers as a FEEL context.
- **Connection Timeout** *(optional)* — Connection timeout in seconds. Defaults to 20.
- **Read Timeout** *(optional)* — Read timeout in seconds. Defaults to 20.

## Configuring the Create Work Item Template (`nicecxone-create-work-item.json`)

Purpose-built for pushing work from Camunda into CXone — the most common integration pattern.

### Parameters

- **CXone API Base URL** — Your regional API endpoint. Store as `{{secrets.CXONE_BASE_URL}}`.
- **Point of Contact** — The CXone Point of Contact that receives the work item. Determines skill/queue routing.
- **Work Item Type** — `Work Item` (general task) or `Callback Request`.
- **Work Item Body** — JSON payload. The default FEEL expression auto-builds a payload using the process instance key:
  ```json
  {
    "pointOfContact": "MyWorkItemPoC",
    "workItemId": "camunda-12345",
    "workItemPayload": "Customer complaint about billing",
    "workItemType": "complaint"
  }
  ```

## Configuring the Get Agent Status Template (`nicecxone-get-agent-status.json`)

Real-time visibility into your contact center from within a Camunda process.

### Parameters

- **CXone API Base URL** — Your regional API endpoint. Store as `{{secrets.CXONE_BASE_URL}}`.
- **What to Query** — Select the type of real-time data:
  - `Agent States` — Current state of all agents
  - `Single Agent State` — State of a specific agent (requires Agent ID)
  - `Skill Queue Summary` — Contacts waiting per skill
  - `Agent Activity` — What agents are currently doing
- **Agent ID** *(conditional)* — Required when querying a single agent's state.
- **Skill ID Filter** *(optional)* — Filter results to a specific skill.
- **Query Parameters** *(optional)* — Additional query params as a FEEL context.

## Output

### Main Connector

The response is stored in the `cxoneResult` process variable. Use the **Result Expression** field to map specific fields:

```json
{
  "cxoneResult": {
    "status": 200,
    "body": { }
  }
}
```

### Create Work Item

The default result expression extracts the contact ID and acceptance status:

```json
{
  "contactId": 123456789,
  "workItemAccepted": true
}
```

### Get Agent Status

The default result expression calculates availability from the agent states response:

```json
{
  "agents": [ ],
  "totalAgents": 10,
  "availableAgents": 6,
  "busyAgents": 4,
  "hasAvailableAgents": true
}
```

Use `hasAvailableAgents` in BPMN gateways to make routing decisions based on live agent availability.

You can customise all output mappings via the **Result Expression** field in each template's Output Mapping section.

## Error Codes

All templates include CXone-specific BPMN error mapping. Catch these with error boundary events in your process:

| HTTP Code | BPMN Error Code | Description |
|---|---|---|
| 401 | `CXONE_AUTH_ERROR` | Authentication failed — check token or credentials |
| 403 | `CXONE_FORBIDDEN` | Permission denied — check API user role |
| 404 | `CXONE_NOT_FOUND` | Resource not found (agent, contact, Point of Contact) |
| 429 | `CXONE_RATE_LIMIT` | Rate limit exceeded — back off and retry |
| 500 | `CXONE_SERVER_ERROR` | CXone internal server error |

All templates include automatic retry with configurable retry count (default: 3) and backoff (default: 10 seconds).

## Requirements

| Component | Version | Notes |
|---|---|---|
| Camunda Platform | 8.5+ | For REST connector runtime |
| CXone API | v31.0 | Current stable version |
| CXone Edition | Enterprise+ | API access required |

## Links

- [NICE CXone Developer Portal](https://developer.niceincontact.com)
- [CXone API Reference](https://developer.niceincontact.com/API)
- [CXone Getting Started Guide](https://developer.niceincontact.com/Documentation/GettingStarted)
- [Camunda Connector Templates Documentation](https://docs.camunda.io/docs/components/connectors/custom-built-connectors/connector-templates/)
- [Camunda Marketplace](https://marketplace.camunda.com)
