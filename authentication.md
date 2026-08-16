# Odoo JSON-2 API: Authentication & Programmatic API Key Rotation Guide

This guide provides an exhaustive technical reference for backend developers integrating an **e-commerce platform** with Odoo's **Authentication & API Key Management (`res.users.apikeys`)** module using the **External JSON-2 API**. It details how to automate zero-downtime API key rotation, validate credentials, and handle expiration.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

---

## Table of Contents

1. [Authentication Architecture & Security Model](#1-authentication-architecture--security-model)
2. [Odoo System Configuration & Prerequisites](#2-odoo-system-configuration--prerequisites)
3. [API Key Methods & Field Specifications](#3-api-key-methods--field-specifications)
   - [3.1 Generate API Key (`res.users.apikeys.generate`)](#31-generate-api-key-resusersapikeysgenerate)
   - [3.2 Revoke API Key (`res.users.apikeys.revoke`)](#32-revoke-api-key-resusersapikeysrevoke)
   - [3.3 Identity Verification (`res.users.context_get`)](#33-identity-verification-resuserscontext_get)
4. [Zero-Downtime Key Rotation Lifecycle (Step-by-Step)](#4-zero-downtime-key-rotation-lifecycle-step-by-step)
   - [Step 1: Test Current Key Validity](#step-1-test-current-key-validity)
   - [Step 2: Generate a New API Key](#step-2-generate-a-new-api-key)
   - [Step 3: Securely Store the New Key](#step-3-securely-store-the-new-key)
   - [Step 4: Verify the New Key Connectivity](#step-4-verify-the-new-key-connectivity)
   - [Step 5: Revoke the Old API Key](#step-5-revoke-the-old-api-key)
5. [Complete HTTP Request & Response Reference](#5-complete-http-request--response-reference)
6. [Error Handling & Failure Responses](#6-error-handling--failure-responses)
7. [Production Best Practices for E-Commerce Developers](#7-production-best-practices-for-e-commerce-developers)

---

## 1. Authentication Architecture & Security Model

In Odoo 19+, all external API interactions use **Bearer Token Authentication** powered by cryptographically secure, 160-bit random API keys.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       E-Commerce Backend Application                        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ HTTP POST with Header:
                                       │ Authorization: bearer <160-bit-key>
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Odoo JSON-2 Gateway                               │
│  1. Resolves Database (Host / X-Odoo-Database)                              │
│  2. Validates Bearer Token in res.users.apikeys                             │
│  3. Verifies Key Expiration Date                                            │
│  4. Executes ORM Method under Authenticated Bot User Permissions           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Expiration & Rotation Rules
- **Maximum Lifetime:** For security reasons, Odoo API keys have an enforced maximum duration (typically **up to 90 days / 3 months**).
- **No In-Place Refresh:** API keys cannot be extended or refreshed under the same string token. Instead, rotation is achieved by **generating a new key using the valid existing key**, switching active traffic to the new key, and **revoking the old key**.
- **Dedicated Service User (Bot Account):** Always assign API keys to a dedicated bot user (e.g., `ecommerce_sync_bot`) rather than personal administrator accounts.

---

## 2. Odoo System Configuration & Prerequisites

To enable programmatic API key generation and rotation for non-admin bot users, ensure the following system parameters are configured in your Odoo instance.

### Enabling Programmatic Key Management
1. In Odoo, enable **Developer Mode**.
2. Navigate to **Settings > Technical > Parameters > System Parameters**.
3. Verify or add the following keys:

| Parameter Key | Type | Default | Required Value | Description |
| :--- | :--- | :--- | :--- | :--- |
| `base.enable_programmatic_api_keys` | `Boolean` | `False` | `True` | Allows users to generate and revoke API keys via the JSON-2 API. |
| `base.programmatic_api_keys_limit` | `Integer` | `10` | `10` (or higher) | Maximum number of active programmatic keys allowed per user at any given time. |

---

## 3. API Key Methods & Field Specifications

### 3.1 Generate API Key (`res.users.apikeys.generate`)

Generates a new 160-bit API key authenticated by an existing valid key.

- **Endpoint:** `POST https://<odoo-domain>/json/2/res.users.apikeys/generate`
- **Authentication:** `Authorization: bearer <CURRENT_VALID_KEY>`

#### Request Parameters Table

| Field Name | Type | Requirement | Default | Description & Constraints |
| :--- | :--- | :--- | :--- | :--- |
| `key` | `string` | **REQUIRED** | — | The currently active, valid API key being used to authenticate the request. |
| `scope` | `string` / `null` | Optional | `null` | The scope assigned to the new key. Pass `null` for general access, or `"rpc"` for standard external API access. |
| `name` | `string` | Optional | `null` | Human-readable label for auditing (e.g., `"E-Commerce Key 2026-Q3"`). Highly recommended. |
| `expiration_date` | `string` | Optional | Max allowed (90d) | Expiration date formatted as ISO 8601 (`"YYYY-MM-DD"`). Must not exceed the maximum allowed duration (90 days from generation date). |

---

### 3.2 Revoke API Key (`res.users.apikeys.revoke`)

Immediately invalidates an API key, rendering it unusable for all subsequent API requests.

- **Endpoint:** `POST https://<odoo-domain>/json/2/res.users.apikeys/revoke`
- **Authentication:** `Authorization: bearer <NEW_KEY>` or `bearer <CURRENT_KEY>`

#### Request Parameters Table

| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `key` | `string` | **REQUIRED** | — | The exact API key string to be revoked immediately. |

> [!TIP]
> **Rotation Best Practice:** Always authenticate the `revoke` call using the **newly generated API key** while passing the **old API key** in the `key` parameter. This guarantees that your new key is fully functional before you destroy the old one.

---

### 3.3 Identity Verification (`res.users.context_get`)

Used as a lightweight healthcheck to verify that an API key is valid and retrieve the active user context without mutating data.

- **Endpoint:** `POST https://<odoo-domain>/json/2/res.users/context_get`
- **Authentication:** `Authorization: bearer <API_KEY>`

---

## 4. Zero-Downtime Key Rotation Lifecycle (Step-by-Step)

Implement this automated workflow in your e-commerce backend (e.g. a scheduled cron job executing every 30 to 60 days).

```
[E-Commerce Cron Job]                  [Odoo JSON-2 API]
         │                                     │
         │---- 1. Healthcheck Current Key ---->| (/json/2/res.users/context_get)
         |<--- 200 OK (Key Valid) -------------|
         │                                     │
         │---- 2. Generate New API Key ------->| (/json/2/res.users.apikeys/generate)
         |<--- 200 OK ("new_key_string...") ---|
         │                                     │
    [Store New Key in Secrets Vault]           │
         │                                     │
         │---- 3. Healthcheck New Key -------->| (/json/2/res.users/context_get)
         |<--- 200 OK (New Key Works) ---------|
         │                                     │
    [Switch Active Backend Client to New Key]  │
         │                                     │
         │---- 4. Revoke Old API Key --------->| (/json/2/res.users.apikeys/revoke)
         |<--- 200 OK (Old Key Revoked) -------|
         │                                     │
    [Rotation Complete - Emit Metrics/Alert]
```

---

### Step 1: Test Current Key Validity

Perform a lightweight check to ensure the existing key has not been revoked or expired.

#### Request
```http
POST /json/2/res.users/context_get HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-AuthManager/1.0

{}
```

#### Success Response (`200 OK`)
```json
{
  "lang": "en_US",
  "tz": "America/New_York",
  "uid": 14
}
```

---

### Step 2: Generate a New API Key

Call `res.users.apikeys/generate` with a new expiration date (e.g. 60–90 days in the future) and a descriptive label.

#### Request
```http
POST /json/2/res.users.apikeys/generate HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-AuthManager/1.0

{
  "key": "6578616d706c65206a736f6e20617069206b6579",
  "scope": null,
  "name": "E-Commerce Auto-Rotated Key (2026-08-16)",
  "expiration_date": "2026-11-14"
}
```

#### Success Response (`200 OK`)
```json
"9f8c2b7e14a60d38e20f1b5c4901e87d3a2f64c1"
```
*(The response is the newly generated 160-bit API key string)*

> [!WARNING]
> **One-Time Display:** The API key string is returned **only once** upon creation and is never stored in plain text by Odoo. Capture and store it immediately in your secrets manager.

---

### Step 3: Securely Store the New Key

Persist the newly returned key (`9f8c2b7e14a60d38e20f1b5c4901e87d3a2f64c1`) in your secure secret management solution (e.g., AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, Google Secret Manager, or encrypted environment configuration).

---

### Step 4: Verify the New Key Connectivity

Before revoking the old key, execute an authenticated request using the **new key** to verify that permissions and connectivity are functional.

#### Request
```http
POST /json/2/res.users/context_get HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 9f8c2b7e14a60d38e20f1b5c4901e87d3a2f64c1
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-AuthManager/1.0

{}
```

#### Success Response (`200 OK`)
```json
{
  "lang": "en_US",
  "tz": "America/New_York",
  "uid": 14
}
```

---

### Step 5: Revoke the Old API Key

Now that the new key is active and verified, revoke the old key. Authenticate using the **new key** and pass the **old key** in the body.

#### Request
```http
POST /json/2/res.users.apikeys/revoke HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 9f8c2b7e14a60d38e20f1b5c4901e87d3a2f64c1
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-AuthManager/1.0

{
  "key": "6578616d706c65206a736f6e20617069206b6579"
}
```

#### Success Response (`200 OK`)
```json
true
```

*(The old key `6578...` is now permanently revoked and cannot be used again)*

---

## 5. Complete HTTP Request & Response Reference

### 5.1 Python Implementation Example for Automated Rotation
Below is a production-ready Python example demonstrating the rotation logic:

```python
import datetime
import requests

ODOO_DOMAIN = "mycompany.odoo.com"
ODOO_DB = "mycompany_db"
BASE_URL = f"https://{ODOO_DOMAIN}/json/2"

def rotate_odoo_api_key(current_api_key: str) -> str:
    headers = {
        "Authorization": f"bearer {current_api_key}",
        "Content-Type": "application/json; charset=utf-8",
        "X-Odoo-Database": ODOO_DB,
        "User-Agent": "MyStore-AuthManager/1.0",
    }
    
    # 1. Calculate expiration (60 days from today)
    new_expiry = (datetime.date.today() + datetime.timedelta(days=60)).isoformat()
    key_name = f"Ecommerce Key (Rotated {datetime.date.today().isoformat()})"
    
    # 2. Request new API key
    gen_payload = {
        "key": current_api_key,
        "scope": None,
        "name": key_name,
        "expiration_date": new_expiry,
    }
    
    res = requests.post(f"{BASE_URL}/res.users.apikeys/generate", headers=headers, json=gen_payload)
    res.raise_for_status()
    new_api_key = res.json()
    
    # 3. Verify new key
    new_headers = {
        "Authorization": f"bearer {new_api_key}",
        "Content-Type": "application/json; charset=utf-8",
        "X-Odoo-Database": ODOO_DB,
    }
    test_res = requests.post(f"{BASE_URL}/res.users/context_get", headers=new_headers, json={})
    test_res.raise_for_status()
    
    # 4. Revoke old key using new key credentials
    revoke_payload = {"key": current_api_key}
    revoke_res = requests.post(f"{BASE_URL}/res.users.apikeys/revoke", headers=new_headers, json=revoke_payload)
    revoke_res.raise_for_status()
    
    return new_api_key
```

---

## 6. Error Handling & Failure Responses

All failures return standard HTTP error status codes with structured JSON error bodies containing `name`, `message`, `arguments`, `context`, and `debug`.

### 6.1 Expired or Invalid API Key (`401 Unauthorized`)
Occurs when attempting to authenticate with a revoked or expired key.

#### Request
```http
POST /json/2/res.users.apikeys/generate HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer expired_or_invalid_key
Content-Type: application/json; charset=utf-8

{
  "key": "expired_or_invalid_key",
  "expiration_date": "2026-11-14"
}
```

#### Error Response (`401 Unauthorized`)
```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/base/models/ir_http.py\", line 240, in _auth_method_bearer\n    raise werkzeug.exceptions.Unauthorized('Invalid apikey')\nwerkzeug.exceptions.Unauthorized: 401 Unauthorized: Invalid apikey\n"
}
```

---

### 6.2 Exceeded Key Limit per User (`422 Unprocessable Content`)
Occurs when attempting to generate a key when the user already has reached `base.programmatic_api_keys_limit` active keys without revoking old ones.

#### Error Response (`422 Unprocessable Content`)
```json
{
  "name": "odoo.exceptions.UserError",
  "message": "You have reached the maximum limit of 10 programmatic API keys for this user. Please revoke unused keys before generating a new one.",
  "arguments": [
    "You have reached the maximum limit of 10 programmatic API keys for this user."
  ],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/base/models/res_users.py\", line 890, in _generate_api_key\n    raise UserError(_('You have reached the maximum limit...'))\nodoo.exceptions.UserError: You have reached the maximum limit of 10 programmatic API keys for this user.\n"
}
```

---

### 6.3 Expiration Date Exceeds Maximum Policy (`400 Bad Request` or `422 Unprocessable Content`)
Occurs when setting `expiration_date` further into the future than the security policy allows (e.g. requesting a 1-year duration).

#### Request
```http
POST /json/2/res.users.apikeys/generate HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "key": "6578616d706c65206a736f6e20617069206b6579",
  "expiration_date": "2028-01-01"
}
```

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.ValidationError",
  "message": "The expiration date exceeds the maximum allowed duration of 90 days for API keys.",
  "arguments": [
    "The expiration date exceeds the maximum allowed duration of 90 days for API keys."
  ],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/base/models/res_users.py\", line 915, in _validate_expiration\n    raise ValidationError(...)\nodoo.exceptions.ValidationError: The expiration date exceeds the maximum allowed duration of 90 days.\n"
}
```

---

### 6.4 Programmatic Keys Disabled by Admin (`403 Forbidden`)
Occurs if `base.enable_programmatic_api_keys` is set to `False` and the calling user is not a system administrator.

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "Programmatic API key management is disabled on this database.",
  "arguments": ["Programmatic API key management is disabled on this database."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/base/models/res_users.py\", line 870, in _check_programmatic_allowed\n    raise AccessError(...)\nodoo.exceptions.AccessError: Programmatic API key management is disabled on this database.\n"
}
```

---

### 6.5 Attempting to Revoke an Unknown or Already Revoked Key (`403 Forbidden` or `404 Not Found`)

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "The specified API key does not exist or has already been revoked.",
  "arguments": ["The specified API key does not exist or has already been revoked."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/base/models/res_users.py\", line 940, in revoke\n    raise AccessError(...)\nodoo.exceptions.AccessError: The specified API key does not exist or has already been revoked.\n"
}
```

---

## 7. Production Best Practices for E-Commerce Developers

### 1. Proactive Rotation Schedule (Every 30 to 60 Days)
Do not wait until day 89 to rotate a 90-day key. Set a recurring automated cron job in your e-commerce infrastructure that rotates keys every **30 to 45 days**. This guarantees that temporary network outages or cron delays will not lead to an expired key.

### 2. Dual-Key Overlap Window
During high-traffic flash sales or deployments across multiple microservice instances:
- Generate the new key.
- Deploy the new key to all workers.
- Wait for a 5-minute cooldown window to ensure all in-flight requests on all worker nodes have transitioned.
- Revoke the old key.

### 3. Dedicated Bot User Accounts
Never tie your e-commerce production API key to an individual employee's Odoo user. When that employee leaves or has their account modified, the API key will break. Create a dedicated user (e.g. `ecommerce_integration_user`) with an empty password and assign the minimum required group permissions (Sales, Inventory, Invoicing).

### 4. Alerting & Dead-Man's Snitch
If key rotation fails (e.g., receives a `401`, `422`, or `403`), immediately trigger high-priority alerts via PagerDuty, Slack, or email to the devops team so manual intervention can occur before the old key expires.
