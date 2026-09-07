

🔑 **The big picture — how every section below connects:**

```
1. AUTHENTICATION  → "Who are you?"        (OAuth2 / OIDC / SAML → issues a JWT token)
2. CONDITIONAL ACCESS → "Should we trust this sign-in?"  (MFA? Compliant device? Risky?)
3. AUTHORIZATION   → "What are you allowed to do?"       (RBAC roles at a scope)
4. PIM             → "If it's a PRIVILEGED role, how/when is it granted?" (JIT, approval)
```

> 🆕 Throughout this guide, sections tagged **🆕 Added** are extra topics not in your original outline but commonly grouped with IAM — worth knowing.

---

---

## PART 1: Microsoft Entra ID (formerly Azure AD)

### 🔹 Tenant

🔑 **Keyword:** _"Your organization's dedicated identity boundary"_

- **Definition:** A dedicated, isolated instance of Entra ID representing one organization. Every user, group, device, and app lives inside a tenant.
- **Identifiers:** Has a unique Tenant ID (GUID) and a default domain like `contoso.onmicrosoft.com`
- **Key fact:** One Azure subscription trusts exactly **one** tenant for identity (though one tenant can have many subscriptions)

---

### 🔹 Users

🔑 **Keyword:** _"Identities that sign in"_

- **Member users:** Created natively inside your tenant (your employees)
- **Guest users (B2B):** Invited from another organization or a personal Microsoft/Google account — for external collaboration

> 🆕 **Added: External Identities (B2B / B2C)**
> 
> - **B2B (Business-to-Business):** Invite external partners as guests to collaborate using _their own_ credentials
> - **B2C (Business-to-Consumer):** A separate Entra ID service for customer-facing apps, letting end users sign in with social accounts (Google, Facebook) or local accounts

---

### 🔹 Groups

🔑 **Keyword:** _"Bundle users together for easier management"_

- **Security groups:** Used to control access to resources
- **Microsoft 365 groups:** Used for collaboration (shared mailbox, calendar, files)
- **Assignment types:**
    - **Assigned** — you manually add/remove members
    - **Dynamic** — membership rule-based, auto-updates based on user/device attributes (e.g., "all users in Sales department")

---

### 🔹 Devices

🔑 **Keyword:** _"Endpoints that can be identified and trusted"_

- **Entra Joined (Azure AD Joined):** Fully cloud-managed device, no on-prem AD needed
- **Hybrid Entra Joined:** Joined to on-prem AD **and** registered in Entra ID — common in enterprises mid-migration
- **Entra Registered (Azure AD Registered):** BYOD — personal device just registered for basic access, not fully managed

---

### 🔹 Applications

🔑 **Keyword:** _"Software that needs an identity to authenticate/authorize"_

- **App Registration:** The **global definition/blueprint** of an application (its permissions, redirect URIs, secrets)
- **Enterprise Application:** The **local instance** of that app inside your specific tenant — this is actually the Service Principal (see below)

---

### 🔹 Service Principals

🔑 **Keyword:** _"The identity an application actually uses to sign in"_

- **Definition:** The local security identity of an application within a specific tenant. The App Registration defines _what_ the app is; the Service Principal is _the actual account_ that gets granted permissions and authenticates.
- **Analogy:** App Registration = blueprint of a car model. Service Principal = the actual car sitting in your garage with license plates (an identity that can be authorized to drive).

---

### 🔹 Managed Identities

🔑 **Keyword:** _"Auto-managed service principal — no credentials to store or rotate"_

- **System-assigned:** Tied 1:1 to a single resource's lifecycle — deleted automatically when the resource is deleted
- **User-assigned:** A standalone identity with its own lifecycle — can be attached to multiple resources at once
- **Why it matters:** Eliminates storing secrets/passwords in code — Azure handles credential rotation behind the scenes
- **Example:** A Function App uses a Managed Identity to read secrets from Key Vault, with zero stored credentials

> 🆕 **Added: Hybrid Identity (Microsoft Entra Connect)**
> 
> - **Password Hash Sync (PHS):** Simplest method — a hash of the password hash is synced to the cloud
> - **Pass-through Authentication (PTA):** Password validated against on-prem AD in real time; password hash never leaves on-prem
> - **Federation:** Redirects sign-in to an on-prem federation server (e.g., AD FS) — see Federation below

---

---

## PART 2: Authentication (proving _who you are_)

### 🔹 OAuth 2.0

🔑 **Keyword:** _"Authorization framework — grants access without sharing your password"_

- **Definition:** Industry-standard protocol for **delegated access** — lets an app access a resource on a user's behalf without ever seeing their password.
- **Key roles:** Resource Owner (user) · Client (the app) · Authorization Server (Entra ID) · Resource Server (the API)

**🔸 Authorization Code Flow** 🔑 _"The gold-standard flow for apps with a backend"_

- User is redirected to sign in → app receives a short-lived **auth code** → app exchanges that code (+ client secret) for an **access token** via a secure back-channel
- **Used for:** Web apps with a server-side backend
- 🆕 **Added — PKCE (Proof Key for Code Exchange):** An extension for public clients (mobile apps, SPAs) that can't safely store a client secret. Prevents an intercepted auth code from being redeemed by an attacker.

**🔸 Device Code Flow** 🔑 _"Sign in using a different device"_

- For input-constrained devices (smart TVs, IoT) with no browser/keyboard
- Device displays a short code → user visits a URL on their phone/laptop, enters the code, authenticates there
- **Example:** Logging into a streaming app on your TV by visiting a link on your phone

**🔸 Client Credentials Flow** 🔑 _"App talking to app — no user involved"_

- Service-to-service / daemon authentication — the app proves its own identity (client ID + secret/certificate)
- **Used for:** Background jobs, daemons, service principals calling APIs unattended

> 🆕 **Added — Refresh Tokens:** Long-lived tokens used to silently get a new access token without forcing the user to log in again 🆕 **Added — Implicit Flow (legacy):** Older flow for browser apps that returned tokens directly in the redirect URL — now **deprecated** in favor of Auth Code + PKCE due to security weaknesses

---

### 🔹 OpenID Connect (OIDC)

🔑 **Keyword:** _"An identity layer built ON TOP of OAuth 2.0"_

- **Definition:** OAuth 2.0 alone only handles _authorization_ (access to resources). OIDC adds _authentication_ — it introduces the **ID Token** (a JWT) that proves who the user actually is.
- **Key addition:** ID Token, alongside the OAuth Access Token

---

### 🔹 SAML

🔑 **Keyword:** _"XML-based, older enterprise SSO standard"_

- **Definition:** Security Assertion Markup Language — exchanges authentication data between an **Identity Provider (IdP)** and a **Service Provider (SP)**, using XML "assertions"
- **Used for:** Legacy enterprise apps; many SaaS apps still support SAML SSO alongside modern OIDC
- **vs OIDC:** SAML = XML-based, older, heavier. OIDC/OAuth = JSON/JWT-based, modern, mobile-friendly

---

### 🔹 Federation

🔑 **Keyword:** _"A trust relationship between two identity systems"_

- **Definition:** Lets a user authenticate with credentials from one system (e.g., on-prem AD FS) to access resources in another (e.g., Entra ID) — no duplicated credentials needed
- **Example:** AD FS federating with Entra ID (a legacy hybrid identity approach, mostly replaced today by Password Hash Sync/PTA)

---

### 🔹 JWT Tokens (JSON Web Tokens)

🔑 **Keyword:** _"A compact, self-contained token proving identity/claims"_

**Structure:** `header.payload.signature` — three Base64URL-encoded parts separated by dots

#### Header

🔑 _"Metadata about the token itself"_

- Contains: token type (`JWT`) and the signing algorithm used (e.g., `RS256`)

#### Payload

🔑 _"The actual claims — data about the user/token"_

- Contains claims like: `sub` (subject/user ID), `iss` (issuer), `aud` (audience), `exp` (expiration time), roles, scopes, tenant ID
- ⚠️ **Important:** The payload is only **encoded**, not encrypted — anyone can read it. Never put secrets in it.

#### Signature

🔑 _"Proof the token hasn't been tampered with"_

- Created by signing the header + payload with a private key
- The verifier recomputes it using the matching public key to confirm the token is authentic and untouched

---

---

## PART 3: Authorization (deciding _what you can do_)

### 🔹 RBAC (Role-Based Access Control)

🔑 **Keyword:** _"What can you DO, scoped to a resource"_

- **Definition:** Azure's system for granting permissions to **Azure resources** based on roles assigned at a given scope

> 🆕 **Added — Critical distinction: Azure RBAC vs Microsoft Entra roles** (commonly confused!)
> 
> ||Azure RBAC|Entra ID roles|
> |---|---|---|
> |Controls access to|**Azure resources** (VMs, storage, etc.)|**Directory tasks** (managing users, resetting passwords)|
> |Scope|Management Group / Subscription / Resource Group / Resource|Tenant-wide|
> |Example roles|Owner, Contributor, Reader|Global Administrator, User Administrator|
> |These are **two completely separate permission systems** — having one doesn't grant the other.|||

**Built-in roles:**

**Owner** 🔑 _"Full control + can manage access"_

- Everything Contributor can do, **plus** the ability to grant access to others (assign roles)

**Contributor** 🔑 _"Full control, but CANNOT manage access"_

- Can create/manage all resources, but cannot grant permissions to others or manage policy assignments

**Reader** 🔑 _"Look but don't touch"_

- Can view existing resources only — zero changes allowed

**Custom Roles** 🔑 _"Build your own role with exactly the permissions needed"_

- When built-in roles don't fit, define exact `Actions` (allowed) and `NotActions` (denied) at a chosen scope
- **Example:** A role that can restart VMs but never delete them

---

### 🔹 Role Assignments

🔑 **Keyword:** _"The binding: WHO + WHAT ROLE + WHERE"_

A role assignment = 3 parts combined:

1. **Security principal** — user, group, service principal, or managed identity
2. **Role definition** — Owner, Contributor, Reader, or a custom role
3. **Scope** — Management Group, Subscription, Resource Group, or single Resource

---

### 🔹 Inheritance (RBAC)

🔑 **Keyword:** _"Roles flow downward through scope"_

- A role assigned at a Management Group or Subscription level automatically flows down to every Resource Group and Resource beneath it (same principle as the resource hierarchy from earlier)

---

---

## PART 4: Conditional Access

🔑 **Keyword:** _"If-this-then-that policies applied at sign-in"_

- **Definition:** A policy engine that evaluates signals (user, location, device, application, risk level) at sign-in time, and enforces controls **before** granting access
- **Structure:** _Assignments_ (who/what/where the policy applies to) + _Access controls_ (what to enforce)

---

### 🔹 MFA (Multi-Factor Authentication)

🔑 **Keyword:** _"Something you know + something you have/are"_

- **Definition:** Requires 2+ verification methods before granting access — e.g., password + Authenticator app approval/SMS code/biometric/security key

> 🆕 **Added — Passwordless Authentication**
> 
> - **Windows Hello for Business**, **Microsoft Authenticator (passwordless)**, **FIDO2 security keys**
> - Removes the password entirely — considered the strongest _and_ most convenient auth method today

---

### 🔹 Device Compliance

🔑 **Keyword:** _"Is this device healthy/managed enough to be trusted?"_

- **Definition:** Conditional Access can require a device be marked **compliant** (via Intune — checks like encryption enabled, up-to-date OS, not jailbroken/rooted) before allowing access

---

### 🔹 Risk Policies

🔑 **Keyword:** _"Block or challenge based on how risky this looks"_

- **Powered by:** Microsoft Entra ID Protection
- **Sign-in risk:** Unfamiliar location, anonymous IP, "impossible travel" (login from two far-apart places too quickly)
- **User risk:** Leaked credentials found on the dark web, consistently unusual behavior
- Policies can require MFA — or block access entirely — when risk is Medium/High

---

### 🔹 Session Controls

🔑 **Keyword:** _"What happens WHILE the session is active — not just at sign-in"_

- **Definition:** Controls enforced **during** an active session
- **Examples:** Blocking downloads via Conditional Access App Control, forcing re-authentication after a set time ("sign-in frequency"), persistent browser session settings

---

---

## PART 5: Privileged Identity Management (PIM)

🔑 **Keyword:** _"Just-in-time, time-limited privileged access"_

- **Definition:** Reduces **standing access** to privileged roles — instead of being a permanent Global Administrator, you're made _eligible_ and only activate the role when you actually need it, for a limited time.

---

### 🔹 JIT Access (Just-In-Time)

🔑 **Keyword:** _"Get the privilege only when you need it, only for as long as you need it"_

- **Definition:** No permanent/standing access — the user activates the role on demand, and access automatically expires after a set duration

---

### 🔹 Eligible Roles

🔑 **Keyword:** _"You COULD have this role, but don't yet"_

- **Eligible assignment:** User must actively **activate** the role before using it (time-bound)
- **Active assignment:** Permanent access, no activation step required — this is the risky "standing access" PIM is designed to reduce

---

### 🔹 Approval Workflow

🔑 **Keyword:** _"Someone else must sign off before you get the role"_

- **Definition:** PIM can require designated approvers to approve an activation request before the privileged role becomes active
- **Adds:** Required justification text, a time-bound duration, and optional MFA re-confirmation at activation

> 🆕 **Added — Access Reviews**
> 
> - Periodic reviews confirming users still need their assigned roles/access — can auto-remove access if unconfirmed
> 
> 🆕 **Added — PIM Alerts**
> 
> - Automatic alerts for suspicious activity, e.g., a role activated too frequently, or roles assigned **outside** of PIM

---

---

## 🧠 Quick Recall Cheat Sheet

| Term                    | One-Line Keyword                                 |
| ----------------------- | ------------------------------------------------ |
| Tenant                  | Your org's dedicated identity boundary           |
| Guest User (B2B)        | External user invited to collaborate             |
| Dynamic Group           | Auto-membership based on rules                   |
| Hybrid Joined Device    | In both on-prem AD and Entra ID                  |
| App Registration        | Blueprint/definition of an app                   |
| Service Principal       | The app's actual sign-in identity                |
| Managed Identity        | Auto-managed identity, no stored credentials     |
| OAuth 2.0               | Delegated access without sharing passwords       |
| Auth Code Flow          | Standard flow for backend web apps               |
| PKCE                    | Auth code protection for mobile/SPA apps         |
| Device Code Flow        | Sign in via a second device                      |
| Client Credentials Flow | App-to-app, no user involved                     |
| OpenID Connect          | Identity layer on top of OAuth                   |
| SAML                    | XML-based legacy SSO standard                    |
| Federation              | Trust link between two identity systems          |
| JWT Header              | Token type + signing algorithm                   |
| JWT Payload             | The claims — readable, not encrypted             |
| JWT Signature           | Proof of no tampering                            |
| Azure RBAC              | Access to Azure resources                        |
| Entra ID roles          | Access to directory tasks (separate system!)     |
| Owner                   | Full control + can manage access                 |
| Contributor             | Full control, no access management               |
| Reader                  | View-only                                        |
| Custom Role             | Exact permissions you define                     |
| Role Assignment         | Who + What Role + Where (scope)                  |
| Conditional Access      | If-this-then-that at sign-in                     |
| MFA                     | 2+ proof factors required                        |
| Passwordless            | No password at all (FIDO2, Hello, Authenticator) |
| Device Compliance       | Is the device trusted/healthy?                   |
| Risk Policy             | Block/challenge based on risk signals            |
| Session Controls        | Rules during an active session                   |
| PIM                     | Just-in-time privileged access                   |
| Eligible Role           | Must activate before use                         |
| Approval Workflow       | Someone must approve activation                  |
| Access Review           | Periodic recheck that access is still needed     |