# agent_poc — MuleSoft MCP Server (Customer Churn & Retention POC)

> **Purpose:** A Proof of Concept MuleSoft application that acts as an **MCP (Model Context Protocol) Server**, exposing customer churn analysis and retention tools to an AI Agent (e.g., Agentforce / Claude) via the `mule-mcp-connector`.

---

## 📦 Project Identity

| Field | Value |
|---|---|
| **Artifact ID** | `agent_poc` |
| **Group ID** | `com.mycompany` |
| **Version** | `1.0.0-SNAPSHOT` |
| **Packaging** | `mule-application` |
| **Mule Runtime** | `4.11.0` |
| **Java Version** | `17` |
| **MCP Server Name** | `mule-salesforce-integration` |
| **MCP Server Version** | `1.0.0` |
| **HTTP Endpoint** | `localhost:8081` |

---

## 🏛️ Architecture

```
AI Agent (LLM / Agentforce / Claude)
              │
              │  MCP Protocol — Streamable HTTP (JSON)
              ▼
┌──────────────────────────────────────────────────────┐
│        MuleSoft MCP Server                           │
│        agent_poc @ localhost:8081                    │
│                                                      │
│  MCP Tools:                                          │
│  ├── get-customer-with-expired-or-expiring-plans     │
│  ├── get-customer-by-churn-level                     │
│  ├── get-customer-by-churn-score                     │
│  ├── get-customer-details-with-highest-churn-score   │
│  ├── get-campaign-details                            │
│  ├── send-email-to-admin                             │
│  ├── send-bulk-retention-email                       │
│  └── get-top-customer-by-revenue                    │
└───────────────┬──────────────────────────────────────┘
                │
     ┌──────────┼──────────────────────┐
     ▼          ▼                      ▼
Salesforce CRM  Google Gemini LLM    AWS SES (Email)
(Basic Auth)   (gemini-2.0-flash)  ap-southeast-2:587
```

---

## 🔌 Connectors & Dependencies

| Connector | Version | Role |
|---|---|---|
| `mule-http-connector` | `1.11.1` | HTTP listener / MCP transport |
| `mule-mcp-connector` | `1.6.0` | **MCP Server** — exposes tools to AI agents |
| `mule-salesforce-connector` | `11.4.0` | Salesforce CRM data access (Basic Auth) |
| `mule-email-connector` | `1.8.0` | Email via AWS SES (SMTP) |
| `mule-objectstore-connector` | `1.2.2` | State / session persistence |
| `mule-sockets-connector` | `1.2.7` | TCP/UDP transport support |

---

## ⚙️ Global Configuration (`agent_poc.xml`)

| Config Name | Type | Details |
|---|---|---|
| `HTTP_Listener_config` | HTTP Listener | `${https.host}:${https.port}` → `localhost:8081` |
| `Salesforce_Config` | Salesforce Basic Auth | Username/password/URL authentication against `https://login.salesforce.com` |
| `MCP_Server` | MCP Server | Streamable HTTP, JSON response; name from `${mcp.name}` |
| `Email_SMTP` | Email SMTP | AWS SES `${aws.host}:${aws.port}` with user/password auth |
| `Gemini_Request_Config` | HTTP Request | HTTPS to `${gemini.api.url}:443` — Google Gemini 2.0 Flash LLM API |

---

## 🗂️ Project Structure

```
agent_poc/
├── README.md                                                    ← This file
├── pom.xml                                                      ← Maven build + all connector dependencies
├── mule-artifact.json                                           ← Runtime 4.11.0, Java 17
├── exchange-docs/
│   └── home.md                                                  ← Anypoint Exchange documentation
├── services/                                                    ← (currently empty)
└── src/
    ├── main/
    │   ├── java/                                                ← Custom Java classes (none yet)
    │   ├── mule/
    │   │   ├── agent_poc.xml                                    ← Global configs (HTTP, MCP, Salesforce, Email)
    │   │   ├── expired-plan-customer.xml                        ← MCP Tool: expired/expiring plan customers
    │   │   ├── get-customer-by-churn-level.xml                  ← MCP Tool: customers by churn risk level
    │   │   ├── get-customer-by-churn-score.xml                  ← MCP Tool: customers by churn score threshold
    │   │   ├── get-customer-details-with-highest-churn-score.xml ← MCP Tool: top N churn customers
    │   │   ├── get-campaign-details.xml                         ← MCP Tool: active Salesforce campaigns
    │   │   ├── sent-email-to-customer.xml                       ← MCP Tool: send email to admin via AWS SES
    │   │   ├── send-bulk-retention-email.xml                    ← MCP Tool: bulk personalized email via Gemini LLM + AWS SES
    │   │   └── top-customer-by-revenue.xml                      ← MCP Tool: top N customers by revenue
    │   └── resources/
    │       ├── properties.yaml                                  ← All environment config & credentials
    │       ├── application-types.xml                            ← Custom metadata type catalog (empty)
    │       ├── log4j2.xml                                       ← Logging configuration
    │       └── api/                                             ← API spec folder (currently empty)
    └── test/
        ├── java/
        ├── munit/                                               ← MUnit test files (currently EMPTY)
        └── resources/
            └── log4j2-test.xml
```

---

## 🔄 Existing MCP Tools (Flows)

### 1. `expired-plan-customer.xml`
- **MCP Tool Name:** `get-customer-with-expired-or-expiring-plans`
- **Flow Name:** `expired-plan-customer-flow`
- **Description:** Returns customers with expired or expiring subscription plans within a given number of days.
- **Input Parameter:**
  - `days` (number, required) — number of days from now to look ahead
- **SOQL:** Queries `Account` WHERE `Valid_Till__c <= now() + days(input.days)`, ordered by `Valid_Till__c ASC`
- **Returns:** JSON array of Account records

---

### 2. `get-customer-by-churn-level.xml`
- **MCP Tool Name:** `get-customer-by-churn-level`
- **Flow Name:** `get-customer-by-churn-level-flow`
- **Description:** Returns customers filtered by their churn risk classification.
- **Input Parameter:**
  - `riskLevel` (string, required) — enum: `"High Risk"` | `"Medium Risk"` | `"Low Risk"`
- **SOQL:** Queries `Account` WHERE `Churn_Risk_Level__c = :riskLevel`, ordered by `Churn_Score__c DESC`
- **Returns:** JSON array of Account records

---

### 3. `get-customer-by-churn-score.xml`
- **MCP Tool Name:** `get-customer-by-churn-score`
- **Flow Name:** `get-customer-by-churn-score-flow`
- **Description:** Returns customers whose churn score meets or exceeds the given threshold.
- **Input Parameter:**
  - `score` (number, required) — minimum churn score threshold
- **SOQL:** Queries `Account` WHERE `Total_Revenue__c != NULL AND Churn_Score__c >= :score`, ordered by `Churn_Score__c DESC`
- **Returns:** JSON array of Account records

---

### 4. `get-customer-details-with-highest-churn-score.xml`
- **MCP Tool Name:** `get-customer-details-with-highest-churn-score`
- **Flow Name:** `get-customer-details-with-highest-churn-score-flow`
- **Description:** Returns the top N customers with the highest churn scores.
- **Input Parameter:**
  - `num` (number, required) — number of records to return (SOQL LIMIT)
- **SOQL:** Queries `Account` WHERE `Total_Revenue__c != NULL`, ordered by `Churn_Score__c DESC LIMIT :num`
- **Returns:** JSON array of Account records

---

### 5. `get-campaign-details.xml`
- **MCP Tool Name:** `get-campaign-details`
- **Flow Name:** `get-campaign-details-flow`
- **Description:** Retrieves currently active Salesforce campaigns.
- **Input Parameter:**
  - `active` (boolean, optional) — not used in query (query always filters `IsActive = true`)
- **SOQL:** `SELECT Id, Name, IsActive, Description FROM Campaign WHERE IsActive = true`
- **Returns:** JSON array of Campaign records

---

### 6. `sent-email-to-customer.xml`
- **MCP Tool Name:** `send-email-to-admin`
- **Flow Name:** `send-email-to-customer-flow`
- **Description:** Sends an email to the configured admin address with customer and campaign data.
- **Input Parameters:**
  - `customers` (array, required) — array of customer objects `{Id, LastName, Email__c, ...}`
  - `campaign` (object, required) — campaign object `{Id, Name, IsActive, Description}`
- **Email:** Sends to `aws.to_email` (admin), FROM `aws.from_email`, subject "Details"
- **Note:** ⚠️ Does NOT send to actual customer email — sends only to admin address. Email body is empty.
- **Returns:** Static string `"email sent successfully.."`

---

### 7. `top-customer-by-revenue.xml`
- **MCP Tool Name:** `get-top-customer-by-revenue`
- **Flow Name:** `top-customer-by-revenue-flow`
- **Description:** Returns the top N customers ranked by total revenue.
- **Input Parameter:**
  - `num` (number, required) — number of records to return (SOQL LIMIT)
- **SOQL:** Queries `Account` WHERE `Total_Revenue__c != NULL`, ordered by `Total_Revenue__c DESC LIMIT :num`
- **Returns:** JSON array of Account records

---

### 8. `send-bulk-retention-email.xml`
- **MCP Tool Name:** `send-bulk-retention-email`
- **Flow Name:** `send-bulk-retention-email-flow`
- **Description:** Accepts a list of customers and a campaign object, calls **Google Gemini 2.0 Flash** to generate a personalized retention email body, then iterates over each customer to send individual emails to their `Email__c` address via AWS SES.
- **Input Parameters:**
  - `customers` (array, required) — minimum 1 item; each requires `Id` and `Email__c`; optional fields: `Name`, `FirstName`, `LastName`, `AccountNumber`, `Valid_Till__c`, `Total_Revenue__c`, `Churn_Score__c`
  - `campaign` (object, required) — `{Id (required), Name (required), IsActive, Description}`
- **LLM Prompt:** Instructs Gemini to generate an email body using named placeholders: `{{CUSTOMER_NAME}}`, `{{VALID_TILL}}`, `{{CHURN_SCORE}}`, `{{CAMPAIGN_NAME}}`
- **Personalization:** DataWeave inside `foreach` replaces each placeholder with per-customer field values
- **Returns:** Summary string — e.g. `"Emails sent to 3 customers for campaign: Summer Retention"`
- **Error Handling:** `on-error-propagate` block catches and logs any Gemini API or email-sending errors
- **Note:** ⚠️ `email:send` is currently **missing** from inside the `foreach` loop — emails are personalized and logged but **not actually dispatched**. See Known Issues.

---

## 🗄️ Salesforce Data Model

All flows query the **`Account`** object with the following custom fields:

| Field API Name | Type | Description |
|---|---|---|
| `Email__c` | Email | Customer email address |
| `Joined_on__c` | Date | Customer join date |
| `tenure_year__c` | Number | Years as a customer |
| `Last_Billing_Date__c` | Date | Most recent billing date |
| `Valid_Till__c` | Date | Plan/subscription expiry date |
| `Total_Revenue__c` | Currency | Total revenue generated by customer |
| `Autopay_Enabled__c` | Boolean | Whether autopay is enabled |
| `Contact_Preferances__c` | Picklist | Preferred contact channel |
| `Churn_Score__c` | Number | Numeric churn risk score |
| `Churn_Risk_Level__c` | Picklist | `High Risk` / `Medium Risk` / `Low Risk` |

Standard fields also used: `Id`, `Name`, `AccountNumber`

---

## ⚙️ Configuration Reference (`properties.yaml`)

| Property Key | Description |
|---|---|
| `https.host` | HTTP listener host |
| `https.port` | HTTP listener port |
| `salesforce.consumer_key` | Salesforce Connected App consumer key |
| `salesforce.consumer_secret` | Salesforce Connected App consumer secret |
| `mcp.name` | MCP server name exposed to agents |
| `mcp.version` | MCP server version |
| `aws.host` | AWS SES SMTP host |
| `aws.port` | AWS SES SMTP port |
| `aws.user` | SMTP username (AWS IAM Access Key) |
| `aws.password` | SMTP password (AWS SES SMTP Password) |
| `aws.from_email` | From address for outgoing emails |
| `aws.to_email` | Admin recipient for email notifications |
| `gemini.api.key` | Google Gemini API key for LLM email generation |
| `gemini.api.url` | Gemini API host — `generativelanguage.googleapis.com` |
| `https.path` | HTTP listener base path (default: `helloworld`) |

> ⚠️ **WARNING:** All credentials are currently stored as **plaintext**. See Security section below.

---

## 🚀 How to Run Locally

### Prerequisites
- Java 17
- Maven 3.8+
- Anypoint Studio or Anypoint Code Builder
- Active Salesforce org with the custom Account fields configured
- AWS SES account in `ap-southeast-2` with SMTP credentials

### Steps
1. Clone / open the project in Anypoint Code Builder
2. Update `src/main/resources/properties.yaml` with your own credentials
3. Run the application (Mule Runtime 4.11.0)
4. Salesforce connects automatically via Basic Auth — no OAuth browser flow required
5. MCP server is available at `http://localhost:8081/mcp` (streamable HTTP)

---

## 🔴 Known Issues & Improvement Recommendations

### Security (Critical)
- [ ] **Encrypt credentials** — Use `mule-secure-configuration-property-module` to encrypt `salesforce.consumer_key`, `salesforce.consumer_secret`, `aws.user`, `aws.password`, and `gemini.api.key` in `properties.yaml`
- [ ] **Salesforce uses Basic Auth** — Username and password are hardcoded in `agent_poc.xml`; migrate to OAuth2 JWT Bearer or a Named Credential for production
- [ ] **Add TLS** — HTTP listener has no TLS context; add `<tls:context>` for production

### Code Quality (High)
- [x] **File naming standardized** — All flow files now use `kebab-case` per MuleSoft conventions
- [x] **Flow names standardized** — All flow `name` attributes now use `kebab-case`
- [x] **MCP tool names standardized** — All MCP tool `name` attributes now use `kebab-case`
- [ ] **No error handling** — None of the 7 flows have `<error-handler>` blocks. Add global + per-flow error handlers
- [x] **Logging implemented** — Structured `INFO` loggers added at entry (with input params) and after Salesforce query (record count) in all 7 flows; email flow has 3 loggers (entry, pre-send, post-send)
- [ ] **`sent-email-to-customer.xml` issues:**
  - Sends to admin only, not to the actual customer
  - Email body content is empty
- [ ] **`send-bulk-retention-email.xml` incomplete:**
  - `email:send` component is **missing** from inside the `foreach` loop — emails are personalized and logged but never dispatched via AWS SES
  - Flow response incorrectly reports `"Emails sent to N customers"` even though no emails are actually sent

### Architecture (Medium)
- [ ] **No multi-environment config** — Single `properties.yaml` covers all environments. Add `properties-dev.yaml`, `properties-uat.yaml`, `properties-prod.yaml`
- [ ] **`get-campaign-details.xml`** — The `active` parameter is accepted but not used; query always filters `IsActive = true`
- [ ] **Empty `api/` folder** — Add an OAS 3.0 spec for the MCP HTTP interface
- [ ] **Empty `services/` folder** — Populate or remove

### Testing (Low)
- [ ] **Zero MUnit tests** — `src/test/munit/` is empty. Write tests for all 7 flows

---

## 💡 Suggested New Flows to Add

### Customer Lookup
| Flow File | MCP Tool | Purpose |
|---|---|---|
| `get-customer-by-id.xml` | `get-customer-by-id` | Fetch single customer by Salesforce Account `Id` |
| `get-customer-by-account-number.xml` | `get-customer-by-account-number` | Look up by `AccountNumber` |
| `search-customers.xml` | `search-customers` | Free-text name search via SOQL LIKE |

### Advanced Analytics
| Flow File | MCP Tool | Purpose |
|---|---|---|
| `get-churn-summary-stats.xml` | `get-churn-summary-stats` | Aggregate stats: count by risk level, avg churn score |
| `get-customers-without-autopay.xml` | `get-customers-without-autopay` | Customers with `Autopay_Enabled__c = false` |
| `get-customers-by-tenure.xml` | `get-customers-by-tenure` | Filter by `tenure_year__c` range |
| `get-revenue-at-risk.xml` | `get-revenue-at-risk` | Sum `Total_Revenue__c` for High Risk customers |

### Enhanced Email / Communication
| Flow File | MCP Tool | Purpose |
|---|---|---|
| `send-email-to-customer-directly.xml` | `send-email-to-customer-directly` | Send personalized email to customer's own `Email__c` |

### Campaign Management
| Flow File | MCP Tool | Purpose |
|---|---|---|
| `get-all-campaigns.xml` | `get-all-campaigns` | All campaigns (active + inactive) |
| `add-customer-to-campaign.xml` | `add-customer-to-campaign` | Create `CampaignMember` in Salesforce |

### Salesforce Write-Back (Currently Read-Only)
| Flow File | MCP Tool | Purpose |
|---|---|---|
| `update-customer-churn-score.xml` | `update-customer-churn-score` | Update `Churn_Score__c` and `Churn_Risk_Level__c` |
| `log-retention-activity.xml` | `log-retention-activity` | Create `Task` record for outreach tracking |
| `update-contact-preference.xml` | `update-contact-preference` | Update `Contact_Preferances__c` based on customer feedback |

### Operational / Utility
| Flow File | MCP Tool | Purpose |
|---|---|---|
| `health-check.xml` | `health-check` | Returns server status — confirms MCP server is alive |
| `get-customers-by-contact-preference.xml` | `get-customers-by-contact-preference` | Filter customers by preferred contact channel |

---

## 📐 Naming Conventions Applied

| Element | Convention | Example |
|---|---|---|
| Flow file names | `kebab-case` | `get-customer-by-churn-level.xml` |
| Flow `name` attributes | `kebab-case` | `get-customer-by-churn-level-flow` |
| MCP tool `name` attributes | `kebab-case` | `get-customer-by-churn-level` |
| DataWeave variables | `camelCase` | `riskLevel`, `churnScore` |
| Property keys | `dot.notation` | `salesforce.consumer_key` |
