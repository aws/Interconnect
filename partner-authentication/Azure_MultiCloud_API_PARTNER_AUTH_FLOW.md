# **ExpressRoute MultiCloud API — Partner Authentication Guide**

---

## **Table of Contents**

- [1. Overview](#1-overview)
- [2. Onboarding](#2-onboarding)
  - [2.1 What the partner provides to Microsoft](#21-what-the-partner-provides-to-microsoft)
  - [2.2 What Microsoft provides to the partner](#22-what-microsoft-provides-to-the-partner)
  - [2.3 Partner Entra App Setup](#23-partner-entra-app-setup)
- [3. Placeholder Reference](#3-placeholder-reference)
- [4. Authentication Flow](#4-authentication-flow)
  - [4.1 Step 1 — Obtain a token from your identity provider (WIF only)](#41-step-1--obtain-a-token-from-your-identity-provider-wif-only)
  - [4.2 Step 2 — Exchange for a Microsoft Entra access token](#42-step-2--exchange-for-a-microsoft-entra-access-token)
  - [4.3 Step 3 — Call the API](#43-step-3--call-the-api)
  - [4.4 curl Example — end-to-end (WIF)](#44-curl-example--end-to-end-wif)
  - [4.5 curl Example — end-to-end (Client Secret)](#45-curl-example--end-to-end-client-secret)
- [5. Common Errors](#5-common-errors)
- [6. Reference Links](#6-reference-links)

---

## **1. Overview**

**The ExpressRoute MultiCloud API uses Microsoft Entra ID (formerly Azure AD) for authentication.** Partners must obtain a Microsoft Entra access token and present it in the `Authorization: Bearer` header on every API request.

**Two authentication methods are supported:**

- **Workload Identity Federation (WIF)** — recommended for partners running services on external identity platforms (e.g., AWS). Your service obtains a token from your own identity provider, then exchanges it with Microsoft Entra for an access token targeting the API. No secrets need to be stored or rotated — the trust is established through a federated identity credential configured on your Entra app registration.

- **Client Secret** — for partners who prefer a simpler setup. You create a client secret on your Entra app registration and use it directly to acquire an access token from Microsoft Entra.

**In both cases, the resulting access token contains:**
- **`aud`** — set to the API's application ID (confirms the token is intended for this API)
- **`azp`** — set to your application ID (identifies who is calling)

---

## **2. Onboarding**

### **2.1 What the partner provides to Microsoft**

| **Item** | **Description** |
|---|---|
| **`{partner-app-id}`** | **Your Entra application (client) ID** |
| **`{partner-tenant-id}`** | **Your Entra tenant ID (GUID)** |
| **`{partner-provider-name}`** | **Your provider name (used in the API URL path)** |

### **2.2 What Microsoft provides to the partner**

| **Item** | **Description** |
|---|---|
| **`{MSFT-app-id}`** | **The API's Entra application ID — used in the `scope` parameter when acquiring tokens** |
| **`{expressroute-multicloud-endpoint}`** | **The API's DNS hostname** |

### **2.3 Partner Entra App Setup**

**Step 1: Register an app in your Entra tenant**
- **Entra admin center → App registrations → New registration**

**Step 2: Configure credentials (choose one):**

#### **Option A: Workload Identity Federation (WIF)**

**Add a Federated Identity Credential on your app:**
- **Entra admin center → App registrations → your app → Certificates & secrets → Federated credentials → Add credential**

| **Field** | **Example value** | **Notes** |
|---|---|---|
| **Issuer** | `https://sts.amazonaws.com` | **Must match the `iss` claim in your external JWT** |
| **Subject identifier** | `arn:aws:iam::123456789012:role/my-service-role` | **Must match the `sub` claim in your external JWT** |
| **Audience** | `api://AzureADTokenExchange` | **Default for WIF exchange. Do NOT set this to the API's app ID.** |

#### **Option B: Client Secret**

**Create a client secret on your app:**
- **Entra admin center → App registrations → your app → Certificates & secrets → Client secrets → New client secret**
- **Copy the secret value immediately (it is only shown once)**
- **Note the expiry — you must rotate the secret before it expires**

**Step 3: Share your `{partner-app-id}`, `{partner-tenant-id}`, and `{partner-provider-name}` with the Microsoft team**

**Step 4: Microsoft will provide `{MSFT-app-id}` and `{expressroute-multicloud-endpoint}`**

---

## **3. Placeholder Reference**

| **Placeholder** | **Who provides** | **Description** |
|---|---|---|
| **`{partner-tenant-id}`** | **Partner** | **Your Entra tenant ID (GUID). This goes in the token endpoint URL.** |
| **`{partner-app-id}`** | **Partner** | **Your registered application (client) ID in your Entra tenant. Becomes the `azp` claim in the token.** |
| **`{partner-provider-name}`** | **Partner** | **Your provider name (used in the API URL path). Coordinate with Microsoft during onboarding.** |
| **`{MSFT-app-id}`** | **Microsoft** | **The API's Entra application ID. Used in the `scope` parameter to set the token's `aud` claim. Microsoft provides this during onboarding.** |
| **`{expressroute-multicloud-endpoint}`** | **Microsoft** | **The API's DNS hostname. Microsoft provides this during onboarding.** |

---

## **4. Authentication Flow**

```
        ┌──────────────────────────────────────────────────────────────┐
        │  Option A: WIF                                               │
        │    Your IdP issues a JWT (issuer and subject must match      │
        │    the values configured in your Entra app)                  │
        │                                                              │
        │  Option B: Client Secret                                     │
        │    You have a client secret configured in your Entra app     │
        └────────────────────────────┬─────────────────────────────────┘
                                     │
                                     ▼
        POST https://login.microsoftonline.com/{partner-tenant-id}/oauth2/v2.0/token
                                     │
                                     │  client_id        = {partner-app-id}
                                     │  scope            = api://{MSFT-app-id}/.default
                                     │  + WIF assertion OR client_secret
                                     │
                                     ▼
        Microsoft Entra issues an access_token:
                                     │
                                     │  aud = api://{MSFT-app-id}     (identifies the target API)
                                     │  azp = {partner-app-id}         (identifies the caller)
                                     │
                                     ▼
        Call the API with:  Authorization: Bearer <access_token>
                                     │
                                     ▼
        API validates the token and processes the request
```

### **4.1 Step 1 — Obtain a token from your identity provider (WIF only)**

**If using WIF, your service first obtains a JWT from your external identity provider. If using a client secret, skip to Step 2.**

```bash
# Example: read the JWT provisioned by your platform (e.g., AWS IAM role token)
EXTERNAL_TOKEN=$(cat /var/run/secrets/tokens/aws-identity-token)
```

### **4.2 Step 2 — Exchange for a Microsoft Entra access token**

#### **Using WIF:**

```http
POST https://login.microsoftonline.com/{partner-tenant-id}/oauth2/v2.0/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

client_id={partner-app-id}
&scope=api://{MSFT-app-id}/.default
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion={jwt-from-partner-external-idp}
&grant_type=client_credentials
```

#### **Using Client Secret:**

```http
POST https://login.microsoftonline.com/{partner-tenant-id}/oauth2/v2.0/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

client_id={partner-app-id}
&scope=api://{MSFT-app-id}/.default
&client_secret={partner-client-secret}
&grant_type=client_credentials
```

### **4.3 Step 3 — Call the API**

```bash
curl -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://{expressroute-multicloud-endpoint}/providers/{partner-provider-name}/environments"
```

### **4.4 curl Example — end-to-end (WIF)**

```bash
# Step 1: Get the JWT from your external IdP (e.g., AWS STS)
EXTERNAL_TOKEN=$(cat /var/run/secrets/tokens/aws-identity-token)

# Step 2: Exchange it for a Microsoft Entra access token
RESPONSE=$(curl -s -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "client_id={partner-app-id}" \
  --data-urlencode "scope=api://{MSFT-app-id}/.default" \
  --data-urlencode "client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer" \
  --data-urlencode "client_assertion=${EXTERNAL_TOKEN}" \
  --data-urlencode "grant_type=client_credentials" \
  "https://login.microsoftonline.com/{partner-tenant-id}/oauth2/v2.0/token")

ACCESS_TOKEN=$(echo $RESPONSE | jq -r .access_token)

# Step 3: Call the API
curl -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://{expressroute-multicloud-endpoint}/providers/{partner-provider-name}/environments"
```

### **4.5 curl Example — end-to-end (Client Secret)**

```bash
# Step 1: Acquire a Microsoft Entra access token using client secret
RESPONSE=$(curl -s -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "client_id={partner-app-id}" \
  --data-urlencode "scope=api://{MSFT-app-id}/.default" \
  --data-urlencode "client_secret={partner-client-secret}" \
  --data-urlencode "grant_type=client_credentials" \
  "https://login.microsoftonline.com/{partner-tenant-id}/oauth2/v2.0/token")

ACCESS_TOKEN=$(echo $RESPONSE | jq -r .access_token)

# Step 2: Call the API
curl -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://{expressroute-multicloud-endpoint}/providers/{partner-provider-name}/environments"
```

> **Note:** `--data-urlencode` tells curl to handle URL-encoding automatically — no need to manually encode special characters.

---

## **5. Common Errors**

| **HTTP Status** | **Cause** | **Fix** |
|---|---|---|
| **401** | **Token has wrong `aud` (wrong `scope` in token request)** | **Set `scope` to `api://{MSFT-app-id}/.default`** |
| **401** | **Token expired** | **Refresh the token (tokens are typically valid for 1 hour)** |
| **401** | **No `Authorization: Bearer` header sent** | **Include the header in every request** |
| **403** | **Your app ID is not yet registered with the API** | **Contact the Microsoft team to complete onboarding** |
| **403** | **Requesting a provider path that doesn't match your registration** | **Use the provider name coordinated during onboarding** |

---

## **6. Reference Links**

- **[Microsoft identity platform — Client credentials flow (v2)](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow#get-a-token)** — token request format and examples
- **[Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)** — how to set up federated credentials
- **[Configure a federated identity credential](https://learn.microsoft.com/en-us/entra/identity-platform/workload-identity-federation-create-trust)** — step-by-step for the Entra app registration
- **[Azure REST API overview](https://learn.microsoft.com/en-us/rest/api/azure/#create-the-request)** — general REST API authentication patterns
