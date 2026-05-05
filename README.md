# NICE CXone Connector for Camunda 8

> **Orchestrate your contact center from BPMN.** Zero-code element templates that let Camunda 8 processes call NICE CXone APIs — create work items, monitor agents, route complaints, and pull real-time data — all from Web Modeler.

[![Camunda 8.5+](https://img.shields.io/badge/Camunda-8.5%2B-blue)](https://camunda.com)
[![CXone API v31](https://img.shields.io/badge/CXone%20API-v31.0-002855)](https://developer.niceincontact.com)
[![Marketplace](https://img.shields.io/badge/Camunda%20Marketplace-Published-success)](https://marketplace.camunda.com/en-US/apps/797170)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)

**📦 Camunda Marketplace**: https://marketplace.camunda.com/en-US/apps/797170
**🔄 CXone Exchange**: https://cxexchange.niceincontact.com/en-US/apps/792411

---

## Features

- **Zero infrastructure** — JSON element templates layered on Camunda's built-in REST connector. No JARs, no Docker, no middleware.
- **22+ operations** across 7 CXone API families (Admin, Patron, Agent, Real-Time, Reporting, Quality, Custom).
- **Three templates**: a multi-operation Swiss-army-knife and two focused single-purpose templates.

---

## Architecture

![Architecture diagram](docs/architecture-diagram.png)

**How it works**: Element templates are JSON configuration files that bind to Camunda's built-in REST Connector runtime (`io.camunda:http-json:1`). All communication is **outbound REST calls** from Camunda to CXone — no inbound webhooks, no agents installed on the CXone side, no shared services.

See [`docs/architecture-diagram.txt`](docs/architecture-diagram.txt) for the full ASCII version.

---

## Repository Contents

| Path | Purpose |
|---|---|
| [`element-templates/nicecxone-connector.json`](element-templates/nicecxone-connector.json) | Multi-operation connector covering all 7 CXone API families |
| [`element-templates/nicecxone-create-work-item.json`](element-templates/nicecxone-create-work-item.json) | Focused template — create CXone work items via Patron API |
| [`element-templates/nicecxone-get-agent-status.json`](element-templates/nicecxone-get-agent-status.json) | Focused template — query real-time agent states |

---

## Requirements

| Component | Version | Notes |
|---|---|---|
| Camunda Platform | 8.5+ | For REST connector runtime |
| Camunda Platform | 8.8+ | Required for AI Agent task in the demo |
| CXone API | v31.0 | Current stable version |
| CXone Edition | Enterprise+ | API access required |
| AWS Bedrock | — | Only if using the demo's AI triage (Claude Sonnet) |

---

## Setup Overview

Setup is split into two parts:

1. **[NICE CXone platform setup](#part-1-nice-cxone-platform-setup)** — done once by your CXone admin
2. **[Camunda setup](#part-2-camunda-setup)** — done once by your Camunda admin, then reused across BPMN processes

Total time for a fresh setup: **~30 minutes** (excluding NICE app registration wait time, which can take 2-3 business days).

---

## Part 1: NICE CXone Platform Setup

You'll configure four things in CXone:

1. Application registration (one-time, with NICE)
2. API employee + access keys
3. A Work Item skill
4. A Point of Contact (with optional Studio script)

> 📖 **Full walkthrough**: [`docs/cxone-platform-setup.md`](docs/cxone-platform-setup.md) has step-by-step screenshots and field-by-field guidance. The summary below covers what you need at a high level.

### 1.1 Register Your Application with NICE

CXone application registration is handled by NICE directly via a registration form (no "Applications" menu in the Admin UI).

🔗 **[NICE CXone Application Registration Form](https://forms.microsoft.com/Pages/ResponsePage.aspx?id=vdojcYcOqU2cubfsggEarfHlkRVlgSlMjqsp52ASGttUM1U2VTQ4N0pXNk5XSzROV0VNNkhBU1gzMC4u)**

**Example values** (adapt for your organization):

| Field | Example Value |
|---|---|
| Application Name | `My Company — Camunda Process Orchestration` |
| Application Type | **Back-end** |
| Authentication Method | **`client_secret_basic`** |
| Tenancy | **Single Tenant** |
| Description | Camunda 8 BPMN connector for orchestrating contact center operations against NICE CXone |

**Request these API scopes** (minimum for the demo):
- Admin API (read agents, skills, teams)
- Patron API (create work items)
- Real-Time Data API (agent states)
- Reporting API (contact history) — optional
- Quality Management API — optional

NICE responds in **2-3 business days** with your `client_id` and `client_secret`. **Save these immediately** — they're the app-level credentials.

> 💡 You can complete steps 1.2–1.4 below in parallel while waiting for NICE.

### 1.2 Create an API Employee

In **CXone Admin → Employees → Create Employee**:

| Field | Example Value |
|---|---|
| First Name | `Camunda` |
| Last Name | `API Service` |
| Email | `camunda-api@yourcompany.com` |
| Role | Administrator (or scoped custom role) |
| Team | Any team (required field) |

**Open the new employee → Security tab → Generate New Access Key**. Save both:
- **Access Key ID** — used as `username` in token requests
- **Access Key Secret** — used as `password` in token requests (shown **only once**)

These are the tenant-level credentials that authenticate as this service account.

### 1.3 Create a Work Item Skill

In **CXone Admin → ACD → Skills (or WEM Skills) → Create Skill**:

| Field | Example Value |
|---|---|
| Skill Name | `Complaint_Resolution` |
| Media Type | **Work Item** |
| Direction | **Inbound** |
| Campaign | Your default campaign |

**Assign at least one agent** to the skill so work items can route.

### 1.4 Create a Point of Contact

In **CXone Admin → ACD → Points of Contact → Create**:

| Field | Example Value |
|---|---|
| PoC Name | `CamundaComplaintPoC` |
| Media Type | **Work Item** |
| Skill | `Complaint_Resolution` (from 1.3) |
| Script | (See 1.5 below) |

> ⚠️ The PoC name is **case-sensitive** and is referenced by the demo BPMN. If you use a different name, update the `pointOfContact` field in the "Create Work Item" task. **Use the friendly name, not the GUID** — the GUID will fail validation.

### 1.5 (Recommended) Create a CXone Studio Script

A minimal Studio script ensures work items wait for agent assignment instead of abandoning immediately. The required action sequence is:

```
Begin → Reqagent (skill: Complaint_Resolution) → Wait → End
```

> ⚠️ **The `Wait` action is required.** Without it, contacts abandon with `inQueueSeconds: 0.0` before any agent can pick them up.

A working script is included at [`scripts/CamundaWorkItemScript.json`](scripts/CamundaWorkItemScript.json) — import it into Studio and assign it to your Point of Contact.

### 1.6 Discover Your Regional API Endpoint

CXone uses regional endpoints. Find yours by calling the discovery endpoint:

```bash
curl -s "https://cxone.niceincontact.com/.well-known/cxone-configuration?tenantId=YOUR_TENANT_ID" | jq .
```

Common base URLs:

| Region | API Base URL |
|---|---|
| North America 1 | `https://api-na1.niceincontact.com` |
| North America 2 | `https://api-na2.niceincontact.com` |
| Europe 1 | `https://api-eu1.niceincontact.com` |
| Australia 1 | `https://api-au1.niceincontact.com` |
| Japan 1 | `https://api-jp1.niceincontact.com` |

---

## Part 2: Camunda Setup

### 2.1 Install the Connector Templates

**Option A — Camunda Marketplace (recommended)**:

1. Open **Camunda Console → Web Modeler**
2. Open the [Camunda Marketplace listing](https://marketplace.camunda.com/en-US/apps/797170)
3. Click **Install** — templates appear automatically in your Web Modeler projects

**Option B — Manual Upload (Web Modeler)**:

1. Open Web Modeler → your project → **More → Upload files**
2. Upload all three files from [`element-templates/`](element-templates/)
3. Templates appear when you select a service task

**Option C — Desktop Modeler**:

Copy the three `.json` files into your Modeler's element-templates directory:
- macOS: `~/Library/Application Support/camunda-modeler/resources/element-templates/`
- Windows: `%APPDATA%/camunda-modeler/resources/element-templates/`
- Linux: `~/.config/camunda-modeler/resources/element-templates/`

Restart Modeler.

> 💡 **If you're updating templates**: Delete old cached versions in Web Modeler before re-importing — Web Modeler aggressively caches old template definitions.

### 2.2 Store Credentials as Cluster Secrets

In **Camunda Console → Clusters → [your cluster] → Connector Secrets**, create the following:

| Secret Name | Source | Required For |
|---|---|---|
| `CXONE_BASE_URL` | Your regional API URL (e.g. `https://api-na1.niceincontact.com`) | All CXone calls |
| `CXONE_CLIENT_ID` | NICE app registration (Step 1.1) | Token acquisition |
| `CXONE_CLIENT_SECRET` | NICE app registration (Step 1.1) | Token acquisition |
| `CXONE_ACCESS_KEY_ID` | API Employee Security tab (Step 1.2) | Token acquisition |
| `CXONE_ACCESS_KEY_SECRET` | API Employee Security tab (Step 1.2) | Token acquisition |
| `CXONE_TOKEN` | Bearer token (see [Token Acquisition](#token-acquisition)) | All CXone API calls |

> 🔐 **Secret syntax**: Reference secrets in templates as `{{secrets.CXONE_TOKEN}}` — never `=secrets.X` or `={{secrets.X}}`. They are resolved at runtime by Camunda and never appear in BPMN XML or logs.

### 2.3 Token Acquisition

Bearer tokens expire after **60 minutes**. Two options:

**Option A: Manual refresh (POC / development)**
Run this curl to mint a token, then paste it into the `CXONE_TOKEN` secret:

```bash
# Replace placeholders with your own credentials
CLIENT_ID="your-client-id"
CLIENT_SECRET="your-client-secret"
ACCESS_KEY_ID="your-access-key-id"
ACCESS_KEY_SECRET="your-access-key-secret"

BASIC=$(echo -n "${CLIENT_ID}:${CLIENT_SECRET}" | base64 -w0)

# Note: --http1.1 is required (HTTP/2 fails on this endpoint)
# Note: '=' characters in access keys must be URL-encoded as %3D
curl --http1.1 -X POST "https://cxone.niceincontact.com/auth/token" \
  -H "Authorization: Basic ${BASIC}" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&username=${ACCESS_KEY_ID//=/%3D}&password=${ACCESS_KEY_SECRET//=/%3D}" \
  | jq -r '.access_token'
```

**Option B: Automated refresh (production)**
Add a "Get Token" service task at the start of every BPMN process (or as a reusable call activity) that calls `POST /auth/token` and stores the token as a process variable. This is the recommended pattern for marketplace customers — see the [Roadmap](#roadmap) for the planned reusable token-refresh template.

---

## Connector Templates Reference

### Main Connector (`nicecxone-connector.json`)

Multi-operation template — supports all CXone API families via dropdown:

| API Group | Operations |
|---|---|
| **Admin** | Get Agents, Get Agent by ID, Get Skills, Get Teams, Update Agent Skills, Get Contacts, Get Points of Contact |
| **Patron** | Create Work Item, Request Callback, Start Chat |
| **Agent** | Start Session, Get Session, Set Agent State, End Session |
| **Real-Time** | Get Agent States, Skill Queue Summary, Agent Activity |
| **Reporting** | Contact History, Performance Summary, Custom Report |
| **Quality** | Submit Evaluation, Create Monitoring Schedule |
| **Custom** | Build your own endpoint (any method, any path) |

### Create Work Item (`nicecxone-create-work-item.json`)

Purpose-built for the most common pattern: pushing work from Camunda into CXone.

- Pre-configured for `POST /interactions/work-items`
- FEEL expression auto-builds the payload with the process instance key
- Smart result mapping for acceptance status

### Get Agent Status (`nicecxone-get-agent-status.json`)

Real-time visibility into your contact center.

- Query all agent states, single agent, or skill queue summary
- FEEL result expression calculates `available`/`busy` counts and a `hasAvailableAgents` boolean
- Use in gateways for routing decisions based on live availability

---

## Demo: Intelligent Customer Complaint Resolution

The included demo BPMN showcases an end-to-end flow:

1. **Customer submits complaint** via Camunda Form
2. **Enrich with CXone data** (agent list via Admin API)
3. **AI triage** via AWS Bedrock Claude Sonnet — returns structured JSON `{severity, reasoning, decision}`
4. **Human review** with approve/override controls
5. **Check agent availability** via Real-Time Data API
6. **Create work item** in CXone via Patron API → routes to agent through Studio script
7. **Poll agent state** every 30s (after a 15s initial delay) to detect completion
8. **Audit + close** the case

### Why agent state polling instead of webhooks?

The Patron API's `POST /interactions/work-items` returns 202 with no body, so there's no `contactId` to poll directly. We poll `GET /agents/states` and watch for the assigned agent returning to `Available`. The 15s initial delay prevents a false-positive completion before the work item even reaches the agent.

### Why `Accept: application/json` on every call?

CXone defaults to `Content-Type: application/html` and the REST connector won't parse the body as JSON without an explicit `Accept: application/json` request header. **Every CXone task in the demo sets this header** — without it, response bodies arrive as the literal string `"No known way to render data"`.

### Why hardcoded credentials in tokens?

The demo uses a static `CXONE_TOKEN` secret refreshed manually for simplicity. In production, replace with the automated token-refresh pattern described in [Token Acquisition](#token-acquisition) Option B.

---

## Authentication Reference

| Credential | Type | Source | Scope |
|---|---|---|---|
| `client_id` | Application | NICE Application Registration form | Identifies the connector app — shared across all customer installs |
| `client_secret` | Application | NICE Application Registration form | Same as above |
| `access_key_id` | Tenant | Customer's CXone Admin → Employee → Security | Identifies the API service account — unique per customer |
| `access_key_secret` | Tenant | Customer's CXone Admin → Employee → Security | Same as above |
| `bearer_token` | Runtime | `POST /auth/token` with the above 4 credentials | Used in `Authorization: Bearer` header — expires hourly |

Token request flow:

```
POST https://cxone.niceincontact.com/auth/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=password&username={access_key_id}&password={access_key_secret}
```

---

## Error Handling

All templates include CXone-specific BPMN error mapping. Catch these with error boundary events:

| HTTP | BPMN Error Code | Meaning |
|---|---|---|
| 401 | `CXONE_AUTH_ERROR` | Token expired or invalid — refresh and retry |
| 403 | `CXONE_FORBIDDEN` | API user role lacks permission |
| 404 | `CXONE_NOT_FOUND` | Resource missing (agent, contact, PoC) |
| 429 | `CXONE_RATE_LIMIT` | Backoff and retry |
| 500 | `CXONE_SERVER_ERROR` | CXone-side issue |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Response body is `"No known way to render data"` | Missing `Accept: application/json` header | Add `={Accept: "application/json"}` to the headers input |
| Work items abandon immediately (`inQueueSeconds: 0.0`) | Studio script missing `Wait` action | Use `Begin → Reqagent → Wait → End` |
| `Point of contact is not valid` | Used PoC GUID instead of friendly name | Use the friendly name (e.g. `CamundaComplaintPoC`) |
| Token request fails with HTTP/2 error | curl using HTTP/2 | Add `--http1.1` flag |
| Polling never detects completion | Agent was already `Available` when poll started | Add a 15s initial delay before first poll |
| AI Agent returns markdown-wrapped JSON | Claude wrapping output | Add "JSON only, no markdown, no thinking tags" to system prompt |
| Template not appearing in Modeler | Old cached version | Delete the old template before re-importing |
| `401 invalid_client` | Wrong `client_id`/`client_secret`, or NICE registration not yet approved | Verify credentials, check NICE response email |
| `401 invalid_grant` | Wrong access keys or revoked | Regenerate access keys in Employee Security tab |

---

## Project Structure

```
nicecxone-connector/
├── element-templates/
│   ├── nicecxone-connector.json              # Multi-operation connector
│   ├── nicecxone-create-work-item.json       # Focused: Create Work Item
│   └── nicecxone-get-agent-status.json       # Focused: Get Agent Status
└── README.md                                 # This file
```

---

## Contributing

This is a partnership integration between Camunda and NICE CXone. Contributions welcome:

1. Fork the repository
2. Create a feature branch
3. Follow the existing element template patterns
4. Test against a CXone sandbox tenant
5. Submit a pull request

---

## License

Apache License 2.0 — see [LICENSE](LICENSE).

---

## Links

- [Camunda Marketplace listing](https://marketplace.camunda.com/en-US/apps/797170)
- [CXone Exchange listing](https://cxexchange.niceincontact.com/en-US/apps/792411)
- [NICE CXone Developer Portal](https://developer.niceincontact.com)
- [CXone API Reference](https://developer.niceincontact.com/API)
- [Camunda Connector Templates Documentation](https://docs.camunda.io/docs/components/connectors/custom-built-connectors/connector-templates/)
