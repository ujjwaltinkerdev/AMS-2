# DaisyBill API — Complete Integration Guide
https://dev.daisybill.com/reference/getting-started

 
## Table of Contents
 
1. [API Overview](#1-api-overview)
2. [Authentication](#2-authentication)
3. [Base URL & Request Format](#3-base-url--request-format)
4. [Pagination](#4-pagination)
5. [Complete API Endpoint Reference](#5-complete-api-endpoint-reference)
6. [Data Model & Hierarchy](#6-data-model--hierarchy)
7. [Webhooks — Complete Guide](#7-webhooks--complete-guide)
8. [Your Integration Flow (Step-by-Step)](#8-your-integration-flow-step-by-step)
9. [Filterable Endpoints](#9-filterable-endpoints)
10. [Error Handling](#10-error-handling)
11. [HIPAA Compliance Notes](#11-hipaa-compliance-notes)
12. [FastAPI Code Examples](#12-fastapi-code-examples)
 
---
 
## 1. API Overview
 
- **Type**: REST API
- **URL Structure**: Predictable, resource-oriented URLs
- **Request Body**: Form-encoded
- **Response Format**: JSON-encoded
- **HTTP Verbs**: GET, POST, PATCH, DELETE
- **Response Codes**: Standard HTTP codes (200, 201, 400, 401, 404, etc.)
- **Official Docs**: https://dev.daisybill.com/reference/getting-started
 
---
 
## 2. Authentication
 
### How to Get the API Token
The client generates the token from **their** DaisyBill account:
DaisyBill UI → Integration icon → API Tokens → Create API Token
> **Requirements**:
> - User must have **Administrator** role
 
### How to Use the Token
 
Send the API token as a **Bearer token** in the `Authorization` header with every request:
 
```http
GET /api/v1/billing_providers HTTP/1.1
Host: go.daisybill.com
Authorization: Bearer YOUR_API_TOKEN
Content-Type: application/json
Accept: application/json
```
 
Python (httpx/requests):
```python
import httpx
 
BASE_URL = "https://go.daisybill.com/api/v1"
HEADERS = {
    "Authorization": "Bearer YOUR_API_TOKEN",
    "Content-Type": "application/json",
    "Accept": "application/json",
}
 
response = httpx.get(f"{BASE_URL}/billing_providers", headers=HEADERS)
```
 
---
 
## 3. Base URL & Request Format
 
### Base URL
 
```
https://go.daisybill.com/api/v1/
```
 
### Request Format
 
| Aspect | Detail |
|---|---|
| **Content-Type** | `application/json` or `application/x-www-form-urlencoded` |
| **Accept** | `application/json` |
| **Auth** | `Authorization: Bearer <token>` |
| **GET** requests | Query params in URL |
| **POST/PATCH** requests | Form-encoded or JSON body |
| **DELETE** requests | Resource ID in URL path |
 
### Example — GET Request (cURL)
 
```bash
curl -X GET "https://go.daisybill.com/api/v1/bills/12345" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Accept: application/json"
```
 
### Example — POST Request (cURL)
 
```bash
curl -X POST "https://go.daisybill.com/api/v1/patients/100/injuries" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "claims_administrator_id=50&date_of_injury=2024-01-15"
```
 
### Example — PATCH Request (cURL)
 
```bash
curl -X PATCH "https://go.daisybill.com/api/v1/injuries/200" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "claims_number=WC2024-001"
```
 
## 5. Complete API Endpoint Reference
 
All endpoints are under `https://go.daisybill.com/api/v1/`
 
### 5.1 Billing Providers
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers` | List all billing providers for the account |
| `GET` | `/billing_providers/{id}` | Get a specific billing provider |
 
> **Starting point** — Every other resource lives under a billing_provider_id.
 
---
 
### 5.2 Patients
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers/{billing_provider_id}/patients` | Search for patients |
| `GET` | `/patients/{id}` | Get patient detail |
| `POST` | `/billing_providers/{billing_provider_id}/patients` | Create a new patient |
| `PATCH` | `/patients/{id}` | Update a patient |
| `DELETE` | `/patients/{id}` | Delete a patient |
 
**Search Parameters** (on the list/search endpoint):
 
| Param | Description |
|---|---|
| `first_name` | First name search query |
| `last_name` | Last name search query |
| `ssn` | SSN search query |
| `date_of_birth` | Date of birth search query |
| `practice_internal_id` | Patient practice internal ID query |
| `page` | Page offset |
| `per_page` | Results per page |
 
---
 
### 5.3 Injuries
 
An **Injury** links a Patient to a specific workers' comp claim and payer.
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/patients/{patient_id}/injuries` | List injuries for a patient |
| `GET` | `/billing_providers/{billing_provider_id}/injuries` | List all injuries for billing provider |
| `GET` | `/injuries/{id}` | Get injury detail |
| `POST` | `/patients/{patient_id}/injuries` | Create an injury |
| `PATCH` | `/injuries/{id}` | Update an injury |
| `DELETE` | `/injuries/{id}` | Delete an injury |
 
> **Note**: When creating an injury, DaisyBill automatically attempts to assign a payer
> if a `claims_administrator_id` is provided and no payer is specified.
 
> **Note**: To submit a bill, the injury must have a `claims_administrator_id` set.
 
---
 
### 5.4 Bills (= Your 837 Claims)
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/injuries/{injury_id}/bills` | List all bills for an injury |
| `GET` | `/billing_providers/{billing_provider_id}/bills` | List all bills for billing provider |
| `GET` | `/bills/{id}` | Get a specific bill |
| `POST` | `/injuries/{injury_id}/bills` | Create a new bill |
| `PATCH` | `/bills/{id}` | Update a bill |
| `DELETE` | `/bills/{id}` | Delete a bill |
| `POST` | `/bills/{id}/close` | Close a bill |
 
**List Bills Filter Parameters**:
 
| Param | Description |
|---|---|
| `status` | Filter by status: `incomplete`, `sent`, `accepted`, `rejected`, `processed`, `lien`, `closed` |
| `practice_bill_id` | Filter by practice bill ID |
| `page` | Page offset |
| `per_page` | Results per page |
 
> **Important for your app**: Creating a bill requires `place_of_service_id` and `rendering_provider_id`.
 
> **NDC Numbers**: For physician-dispensed drugs, daisyBill automatically calculates charges
> and expected amounts based on NDC numbers in California's workers' comp pharmacy fee schedule.
 
---
 
### 5.5 Bill Submissions
 
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/bills/{bill_id}/bill_submission` | Submit a bill to the payer (sends the 837) |
 
---
 
### 5.6 Service Line Items (Procedures on a Bill)
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/bills/{bill_id}/service_line_items` | List all line items on a bill |
| `GET` | `/service_line_items/{id}` | Get a specific line item |
| `POST` | `/bills/{bill_id}/service_line_items` | Create a line item |
| `PATCH` | `/service_line_items/{id}` | Update a line item |
 
> **Critical for AI analysis**: Each service line item contains CPT codes, charges,
> and modifiers — this is what was billed and what the AI will compare against payments.
 
---
 
### 5.7 Bill Payments (= Your 835 EOR Responses)
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/bills/{bill_id}/bill_payments` | List all payments for a bill |
| `GET` | `/bill_payments/{id}` | Get a specific payment detail |
 
> **Key fields**: `void_date` and `void_reason` keys are included in the response if the
> bill payment has been voided.
 
> **This is the 835 data** — contains what the insurance company paid, denied, or adjusted.
 
---
 
### 5.8 Paper EORs (Manually Entered EOR Data)
 
When the payer sends a paper EOR instead of an electronic 835:
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/bills/{bill_id}/paper_eors` | List paper EORs for a bill |
| `GET` | `/paper_eors/{id}` | Get a specific paper EOR |
| `POST` | `/bills/{bill_id}/paper_eors` | Create a paper EOR entry |
 
---
 
### 5.9 Remittances (Batch Payment Summaries)
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers/{billing_provider_id}/remittances` | List all remittances |
| `GET` | `/remittances/{id}` | Get remittance detail |
| `POST` | `/billing_providers/{billing_provider_id}/remittances` | Create a remittance |
| `PATCH` | `/remittances/{id}` | Update a remittance |
 
---
 
### 5.10 Bill Notes
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/bills/{bill_id}/bill_notes` | Get bill notes |
| `POST` | `/bills/{bill_id}/bill_notes` | Create a bill note |
| `PATCH` | `/bill_notes/{id}` | Update a bill note |
 
---
 
### 5.11 Attachments
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/bills/{bill_id}/attachments` | List attachments for a bill |
| `GET` | `/attachments/{id}` | Get a specific attachment |
| `POST` | `/bills/{bill_id}/attachments` | Create an attachment (file upload) |
| `POST` | `/bills/{bill_id}/attachments/base64` | Create an attachment (Base64 encoded) |
| `PATCH` | `/attachments/{id}` | Update an attachment |
| `DELETE` | `/attachments/{id}` | Delete an attachment |
 
---
 
### 5.12 Second Review Appeals
 
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/bills/{bill_id}/request_for_second_review` | **Submit an appeal** for a bill |
 
> **This is the endpoint your AI suggestions will trigger** — when the AI recommends
> appealing a denied/underpaid claim, your app calls this endpoint.
 
---
 
### 5.13 Tasks
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/bills/{bill_id}/tasks` | List tasks for a specific bill |
| `GET` | `/billing_providers/{billing_provider_id}/tasks` | List all tasks for billing provider |
| `GET` | `/tasks/{id}` | Get a specific task |
| `POST` | `/bills/{bill_id}/tasks` | Create a task |
 
> Tasks are action items that daisyBill generates (e.g., compliance corrections needed).
 
---
 
### 5.14 Claims Administrators & Payers
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers/{billing_provider_id}/claims_administrators` | List claims administrators |
| `GET` | `/claims_administrators/{id}` | Get claims administrator detail |
| `GET` | `/claims_administrators/{claims_administrator_id}/payers` | List payers for a claims admin |
| `GET` | `/payers/{id}` | Get payer detail |
 
> Claims Administrators have properties `main_payer_present` and `payer_required` which
> determine if setting the payer is required to send a bill.
 
---
 
### 5.15 Rendering Providers
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers/{billing_provider_id}/rendering_providers` | List rendering providers |
| `GET` | `/rendering_providers/{id}` | Get a rendering provider |
| `POST` | `/billing_providers/{billing_provider_id}/rendering_providers` | Create rendering provider |
| `PATCH` | `/rendering_providers/{id}` | Update rendering provider |
 
---
 
### 5.16 Prescribing Providers
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers/{billing_provider_id}/prescribing_providers` | List prescribing providers |
| `GET` | `/prescribing_providers/{id}` | Get a prescribing provider |
| `POST` | `/billing_providers/{billing_provider_id}/prescribing_providers` | Create prescribing provider |
| `PATCH` | `/prescribing_providers/{id}` | Update prescribing provider |
 
---
 
### 5.17 Places of Service
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/billing_providers/{billing_provider_id}/places_of_service` | List places of service |
| `GET` | `/places_of_service/{id}` | Get a place of service |
| `POST` | `/billing_providers/{billing_provider_id}/places_of_service` | Create place of service |
 
---
 
### 5.18 Contacts
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/injuries/{injury_id}/contacts` | List contacts for an injury |
| `GET` | `/contacts/{id}` | Get a specific contact |
| `POST` | `/injuries/{injury_id}/contacts` | Create a contact |
| `PATCH` | `/contacts/{id}` | Update a contact |
 
---
 
### 5.19 Practice Contacts
 
| Method | Endpoint | Description |
|---|---|---|
| `PATCH` | `/practice_contacts/{id}` | Update a practice contact |
 
---
 
### 5.20 Bill Mailing Addresses
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/payers/{payer_id}/bill_mailing_addresses` | List bill mailing addresses for a payer |
| `GET` | `/claims_administrators/{id}/bill_mailing_addresses` | List bill mailing addresses for a claims admin |
 
---
 
### 5.21 Pharmacy Bills
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/injuries/{injury_id}/pharmacy_bills` | List pharmacy bills |
| `GET` | `/billing_providers/{billing_provider_id}/pharmacy_bills` | List pharmacy bills for billing provider |
| `GET` | `/pharmacy_bills/{id}` | Get a pharmacy bill |
| `POST` | `/injuries/{injury_id}/pharmacy_bills` | Create a pharmacy bill |
| `PATCH` | `/pharmacy_bills/{id}` | Update a pharmacy bill |
| `DELETE` | `/pharmacy_bills/{id}` | Delete a pharmacy bill |
 
---
 
### 5.22 Calculations (Fee Schedule Lookups)
 
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/calculations/ca` | California workers' comp fee schedule calculation |
| `POST` | `/calculations/ny` | New York workers' comp fee schedule calculation |
| `POST` | `/calculations/owcp` | OWCP (federal) fee schedule calculation |
 
---
 
### 5.23 Claim Verification
 
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/verify_claim_number` | Verify a claim number is valid |
 
---
 
### 5.24 Error Reports
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/error_reports` | List error reports |
| `GET` | `/error_reports/{id}` | Get an error report |
| `POST` | `/error_reports` | Create an error report |
 
---
 
### 5.25 Events (Webhook Event History)
 
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/events` | List recent events (last 30 days) |
| `GET` | `/events/{id}` | Get a specific event |
 
> **Fallback for webhooks**: If you miss a webhook, you can poll these endpoints to catch up.
> DaisyBill retains the last **30 days** of event and webhook delivery information.
 
---
 
## 6. Data Model & Hierarchy
 
```
Account (authenticated via API token)
│
├── Billing Provider (your client's organization)
│   ├── Patients
│   │   └── Injuries (linked to Claims Administrator + Payer)
│   │       ├── Bills (837 claims)
│   │       │   ├── Service Line Items (CPT codes, charges)
│   │       │   ├── Bill Payments (835 EOR responses)
│   │       │   ├── Paper EORs (manual EOR entries)
│   │       │   ├── Attachments (supporting documents)
│   │       │   ├── Bill Notes
│   │       │   ├── Tasks (action items)
│   │       │   └── Second Review Appeals
│   │       ├── Contacts
│   │       └── Pharmacy Bills
│   │
│   ├── Rendering Providers (doctors who performed service)
│   ├── Prescribing Providers
│   ├── Places of Service
│   ├── Remittances (batch payment summaries)
│   └── Claims Administrators
│       └── Payers
│           └── Bill Mailing Addresses
│
└── Events (webhook event history)
```
 
### Key Relationships
 
| Resource | Parent | Linked Via |
|---|---|---|
| Patient | Billing Provider | `billing_provider_id` |
| Injury | Patient | `patient_id` |
| Bill | Injury | `injury_id` |
| Bill Payment | Bill | `bill_id` |
| Service Line Item | Bill | `bill_id` |
| Paper EOR | Bill | `bill_id` |
| Attachment | Bill | `bill_id` |
| Task | Bill | `bill_id` |
| Second Review | Bill | `bill_id` |
| Remittance | Billing Provider | `billing_provider_id` |
 
### How 837 (Bill) Links to 835 (Payment)
 
**DaisyBill does the linking for you.** The `bill_payment` resource is always associated
with a specific `bill_id`. You do NOT need to manually match 837 ↔ 835 files.
 
```
Bill (837)  ──────────  bill_id  ──────────  Bill Payment (835)
  └── service_line_items                      └── payment amounts
      (what was billed)                           (what was paid/denied)
```
 
---
 
## 7. Webhooks — Complete Guide
 
### 7.1 What Are Webhooks in DaisyBill?
 
Webhooks are HTTP POST notifications sent to your server when events happen in DaisyBill.
 
### 7.2 How to Set Up Webhooks
 
Webhook endpoints are created **via the DaisyBill UI only** (not via API):
 
```
Step 1: Log in to DaisyBill as Administrator
Step 2: Click Integration icon → Select "Webhooks"
Step 3: Click "Create Webhook Endpoint"
Step 4: Enter a Description (e.g., "AMS-2 Claim Processor")
Step 5: Enter your URL (e.g., https://yourdomain.com/api/webhooks/daisybill)
Step 6: Select Event Types to subscribe to
Step 7: (Optional) Add Basic Auth Username and Password
Step 8: Click "Create"
```
> **Note**: Only users with the Administrator role can access Webhooks.

### 7.3 Webhook Payload Structure
 
Every webhook POST contains:
 
```json
{
  "event_type": "bill_payment.created",
  "event_source": "daisybill",
  "billing_provider": {
    "id": 123,
    "name": "Client Medical Corp"
  },
  "user": {
    "id": 456,
    "email": "user@client.com"
  },
  "payload": {
    "resource": {
      // Same JSON structure as the corresponding GET API endpoint response
      // e.g., for bill_payment.created, this matches GET /bill_payments/{id}
    },
    "previous_attributes": {
      // ONLY present on .updated events
      // Shows the resource state BEFORE the update
    }
  }
}
```
 
### 7.4 All Supported Webhook Event Types
 
Based on documentation and the pattern `{resource}.{action}`:
 
| Event Type | Triggers When | Priority for Your App |
|---|---|---|
| `patient.created` | New patient added to DaisyBill | Medium |
| `patient.updated` | Patient record modified | Low |
| `bill.created` | New bill created (837 claim entered) | High |
| `bill.updated` | Bill status changes (sent → accepted → processed → etc.) | High |
| `bill_payment.created` | **Insurance responds with payment/EOR (835)** | **CRITICAL** |
| `bill_payment.updated` | Payment modified or voided | High |
| `task.created` | DaisyBill flags a compliance issue or action item | Medium |
| `task.updated` | Task is resolved or modified | Low |
| `remittance.created` | New remittance batch received | Medium |
| `remittance.updated` | Remittance modified | Low |
 
### 7.5 Which Webhooks You MUST Subscribe To (and Why)
 
#### CRITICAL — Subscribe to these:
 
**1. ``bill.updated`** — THE most important event
- **Why**: This fires when the insurance company responds with 835 EOR data
- **What you do**: Fetch the original bill (837) + service line items, compare against payment, run AI analysis
- **Payload contains**: Payment amounts, adjustment codes, denial reasons
 
**2. `bill.updated`** — Track the claim lifecycle
- **Why**: Bills go through a lifecycle: `incomplete → sent → accepted → rejected → processed → closed`
- **What you do**: Update your dashboard to show current status
- **Payload contains**: New bill state + `previous_attributes` showing what changed
 
**3. `bill.created`** — Know about new claims
- **Why**: New bills (837 claims) are entered in DaisyBill
- **What you do**: Sync to your system, start tracking this claim
 
#### RECOMMENDED — Subscribe to these:
 
**4. `task.created`** — Compliance alerts
- **Why**: DaisyBill auto-generates tasks when compliance corrections are needed
- **What you do**: Alert the client that action is required on a bill
 
**5. `bill_payment.updated`** — Payment changes
- **Why**: Payments can be voided or adjusted
- **What you do**: Re-run AI analysis if payment was voided
 
**6. `patient.created`** — New patient sync
- **Why**: Keep your local patient records in sync
 
### 7.6 Webhook Security — Basic Auth
 
When creating the webhook endpoint, you can optionally set:
- **Basic Auth Username**
- **Basic Auth Password**
 
DaisyBill will include HTTP Basic Authentication in every webhook request.
Use this to verify that the request is genuinely from DaisyBill.
 
```python
import base64
 
def verify_webhook_auth(authorization_header: str) -> bool:
    expected = base64.b64encode(b"your_username:your_password").decode()
    return authorization_header == f"Basic {expected}"
```
 
### 7.7 Webhook Fallback — Event Polling
 
If your server goes down and misses webhooks:
 
```
GET /api/v1/events?page=1&per_page=25
```
 
DaisyBill retains the last **30 days** of event and webhook delivery information.
Use this to catch up on missed events.
 
### 7.8 Webhook Endpoint Management
 
Once created, webhook endpoints can be:
- **Edited** (change URL, events, auth)
- **Paused** (temporarily stop receiving webhooks)
- **Deleted** (permanently remove)
 
All managed via the DaisyBill UI.
 
```
1. Client generates API token in DaisyBill (Admin → Integration → API Tokens)
2. Client creates webhook endpoint pointing to your FastAPI URL
   - URL: https://yourdomain.com/api/webhooks/daisybill
   - Events: bill.created, bill.updated, bill_payment.created,
             bill_payment.updated, task.created
   - Basic Auth: set username + password
3. Store the API token securely (encrypted, never in source code)
```
---
 
## 11. HIPAA Compliance Notes
 
Since you're handling ePHI (electronic Protected Health Information):
 
### Transport Security
- All DaisyBill API communication is over **HTTPS** (TLS encrypted)
- Your webhook endpoint **MUST** use HTTPS
- Use Basic Auth on webhook endpoints for additional verification
 
### Data Storage
- Store PHI in **encrypted databases** (AES-256 at rest)
- Never log full patient SSN, DOB, or medical details in plain text
- Implement **audit logging** for all PHI access
 
### Access Control
- Store the API token in **environment variables** or a **secrets manager**
- Never commit tokens to source code
- Implement role-based access in your Next.js app
 
### Business Associate Agreement (BAA)      
- Your client should have a **BAA with DaisyBill** (DaisyBill likely already requires this)
- **You** (as the developer building the integration) may also need a BAA with the client
 
### Webhook Security
- Use Basic Auth on all webhook endpoints
- Validate the payload structure before processing
- Implement HTTPS-only webhook URLs
- Reject requests that fail Basic Auth verification
 
### Base URL
 
```
https://go.daisybill.com/api/v1/
```