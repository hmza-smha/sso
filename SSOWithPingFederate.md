# SSO With PingFederate

You have:

* `ProductA.xyz.com`
* `ProductB.xyz.com`
* `ProductC.xyz.com`
* Running on **client infrastructure**
* Client wants SSO using **PingFederate**

So now:

👉 PingFederate will be the **Identity Provider (IdP)**
👉 Your products will become **Service Providers (SP)** or **OIDC Clients**

Below is a practical step-by-step guide.

---

# 🔎 First: Decide the Protocol

PingFederate supports:

* SAML 2.0
* OAuth 2.0
* OpenID Connect (OIDC)
* WS-Federation

✅ **Recommendation:** Use **OpenID Connect (OIDC)**
Why?

* Modern
* REST friendly
* Works well for web + APIs
* Easier than SAML for most apps

Only use SAML if your products are legacy.

---

# 🏗 High-Level Architecture

```
User
   ↓
ProductA.xyz.com
   ↓ (redirect)
PingFederate (IdP)
   ↓
Back to ProductA with tokens
```

Same flow for ProductB and ProductC.

PingFederate becomes the central login system.

---

# 🚀 STEP 1 – Install & Setup PingFederate

PingFederate is usually deployed:

* On Windows or Linux
* As a Java-based server
* Behind HTTPS
* With a public URL like:

```
https://auth.xyz.com
```

### Installation Steps (High-Level)

1. Install Java (if not bundled)
2. Download PingFederate installer
3. Run installation wizard
4. Configure:

   * Admin port
   * Runtime port
   * SSL certificates
5. Access Admin Console:

   ```
   https://auth.xyz.com:9999/pingfederate/app
   ```

Now you configure it via UI.

---

# 🔐 STEP 2 – Configure Identity Store

PingFederate must validate users.

Options:

* Active Directory (most common)
* LDAP
* SQL database
* External IdP (like Azure AD)

In Admin UI:

```
System → Data Stores → Add New Data Store
```

Configure connection to:

* AD / LDAP / DB

Then:

```
Authentication → IdP Adapters
```

Create an adapter like:

* HTML Form Adapter (username/password login)

Test login locally before integrating products.

---

# 🔑 STEP 3 – Configure OpenID Connect

Now enable OIDC:

```
OAuth Settings → Authorization Server
```

Configure:

* Issuer URL (e.g., [https://auth.xyz.com](https://auth.xyz.com))
* Token signing certificate
* Access token lifetime
* ID token lifetime

---

# 🧩 STEP 4 – Register Each Product as an OIDC Client

For each product:

Go to:

```
OAuth Settings → Clients → Create New Client
```

Create 3 clients:

### Client 1

* Name: ProductA
* Redirect URI:

  ```
  https://ProductA.xyz.com/callback
  ```

### Client 2

* ProductB
* Redirect URI:

  ```
  https://ProductB.xyz.com/callback
  ```

### Client 3

* ProductC
* Redirect URI:

  ```
  https://ProductC.xyz.com/callback
  ```

For each client configure:

* Grant Type: Authorization Code
* PKCE: Recommended
* Scopes:

  * openid
  * profile
  * email
* Token Endpoint Auth:

  * client_secret_basic (or private_key_jwt for higher security)

Save the:

* client_id
* client_secret

You will need these in your applications.

---

# 🔄 STEP 5 – Implement OIDC in Your Products

Now changes required in:

* ProductA
* ProductB
* ProductC

---

# 🛠 What You Must Implement in Each Product

Each product must:

### 1️⃣ Detect If User Is Not Authenticated

If no local session:

Redirect to:

```
https://auth.xyz.com/as/authorization.oauth2?
  client_id=XXXX
  &response_type=code
  &scope=openid profile email
  &redirect_uri=https://ProductA.xyz.com/callback
  &state=xyz
  &nonce=abc
```

---

### 2️⃣ Handle Callback

PingFederate redirects back:

```
https://ProductA.xyz.com/callback?code=AUTH_CODE
```

Your product must:

* Exchange code for tokens via backchannel:

```
POST https://auth.xyz.com/as/token.oauth2
```

Send:

* client_id
* client_secret
* code
* redirect_uri

Receive:

* access_token
* id_token
* refresh_token (optional)

---

### 3️⃣ Validate ID Token

Your product must:

* Verify signature
* Verify issuer
* Verify audience
* Verify expiration
* Validate nonce

You can use OIDC libraries for this (don’t write manually).

---

### 4️⃣ Create Local Session

After validation:

* Extract user ID from ID token
* Create local app session
* Store session in secure cookie

Now user is logged in.

---

# 🔁 How SSO Works Between Products

User logs into ProductA:

1. Redirect to PingFederate
2. Login
3. PingFederate sets session cookie on:

```
auth.xyz.com
```

Now user opens ProductB:

1. ProductB redirects to PingFederate
2. Browser automatically sends PingFederate session cookie
3. PingFederate sees active session
4. Immediately redirects back with auth code
5. No login screen shown

🎉 That’s SSO.

---

# 📱 Cross-Browser / Cross-Device Behavior

* Same browser → SSO works
* Different browser → user logs in again
* Different device → login again

This is normal and expected.

---

# 🔓 STEP 6 – Implement Logout (Very Important)

You should configure:

OIDC RP-Initiated Logout:

User clicks logout in ProductA:

1. ProductA clears local session
2. Redirect to:

```
https://auth.xyz.com/idp/startSLO.ping
```

Or OIDC end_session endpoint.

This logs out centrally.

---

# 🔐 Security Best Practices

* Always use HTTPS
* Use Authorization Code Flow (not implicit)
* Enable PKCE
* Use short-lived access tokens
* Validate tokens properly
* Use secure HttpOnly cookies
* Restrict redirect URIs strictly

---

# 📋 Summary of What You Must Do

## In PingFederate

✔ Install server
✔ Connect identity store
✔ Configure OIDC
✔ Register 3 clients
✔ Configure certificates

## In Your Products

✔ Implement OIDC client logic
✔ Add login redirect
✔ Add callback endpoint
✔ Validate tokens
✔ Create local session
✔ Implement logout

---

# 🎯 Final Advice

If your products are:

* Java → use Spring Security OIDC
* .NET → use Microsoft OpenID Connect middleware
* Node.js → use passport-openidconnect or oidc-client
* Angular/React → use oidc-client-ts

Never implement token validation manually.

---

## Client already has **PingFederate** running.

So:

* ✅ PingFederate = Identity Provider (IdP)
* ✅ Your Products (A, B, C) = Service Providers (SP) / OIDC Clients
* 🎯 Goal = Enable SSO across ProductA.xyz.com, ProductB.xyz.com, ProductC.xyz.com

Now let’s clearly separate responsibilities.

---

# 🏗 Big Picture

PingFederate handles:

* Authentication
* Session at IdP level
* Token issuing

Your products handle:

* Redirecting to PingFederate
* Validating tokens
* Creating local application sessions

Your products **do NOT manage global SSO sessions**.

---

# ✅ What YOU Must Do (Product Side Changes)

You must convert each product into an **OIDC client**.

---

# 🔧 1️⃣ Add OIDC Authentication Support

Each product must:

### ✔ Stop using its own login form

OR

### ✔ Support external login via PingFederate

Instead of:

```
Username + Password form inside your app
```

You now:

Redirect unauthenticated users to PingFederate.

---

# 🔄 2️⃣ Implement Authorization Code Flow

When user is not logged in:

Redirect to:

```
https://PINGFEDERATE_DOMAIN/as/authorization.oauth2
```

With parameters:

* client_id
* response_type=code
* scope=openid profile email
* redirect_uri=[https://ProductA.xyz.com/callback](https://ProductA.xyz.com/callback)
* state
* nonce

---

# 🔁 3️⃣ Implement Callback Endpoint

You must create:

```
/callback
```

When PingFederate redirects back:

```
https://ProductA.xyz.com/callback?code=AUTH_CODE
```

Your backend must:

1. Call PingFederate token endpoint
2. Exchange code for:

   * id_token
   * access_token
3. Validate ID token
4. Extract user identity
5. Create local session
6. Set your own secure cookie

---

# 🔐 4️⃣ Add Token Validation Logic

Your products must validate:

* Token signature
* Issuer
* Audience
* Expiration
* Nonce

Use OIDC libraries — never manually decode JWT only.

---

# 🧩 5️⃣ Map External User to Internal User

You must decide:

When you receive ID token with:

```
email = john@company.com
sub = 987654
```

What do you do?

Options:

### Option A – Just-in-time user provisioning

Create user automatically if not exists.

### Option B – Pre-provision users

Only allow users already in your database.

You must implement that logic.

---

# 🔓 6️⃣ Update Logout Logic

When user logs out from ProductA:

You should:

1. Clear local session
2. Redirect to PingFederate logout endpoint

So SSO session ends centrally.

---

# 🧠 Important: Your Products Still Need Local Sessions

Even with SSO:

* You still create your own app session
* You still use your own cookies
* You do NOT rely on PingFederate for every request

PingFederate is only for authentication.

---

# ✅ What THE CLIENT Must Do (PingFederate Side)

They must:

---

## 1️⃣ Register Your Products as OIDC Clients

For each product they must configure:

* Client ID
* Client Secret
* Redirect URI:

  * [https://ProductA.xyz.com/callback](https://ProductA.xyz.com/callback)
  * [https://ProductB.xyz.com/callback](https://ProductB.xyz.com/callback)
  * [https://ProductC.xyz.com/callback](https://ProductC.xyz.com/callback)
* Allowed grant type:

  * Authorization Code
* Allowed scopes:

  * openid
  * profile
  * email

---

## 2️⃣ Provide You With:

They must give you:

* Issuer URL
* Authorization endpoint
* Token endpoint
* JWKS endpoint (for signature validation)
* Client ID
* Client Secret
* Supported scopes
* Logout endpoint

You cannot proceed without this information.

---

## 3️⃣ Configure User Attributes

They must configure what claims are sent in ID token:

Example claims:

* sub
* email
* given_name
* family_name
* groups
* roles

You must tell them what you need.

---

## 4️⃣ Configure SSO Policy

They must ensure:

* SSO session is enabled
* Proper cookie domain setup
* Session lifetime policy defined

---

# 🔄 How SSO Will Work After Integration

User logs into ProductA:

1. Redirect → PingFederate
2. Login
3. PingFederate sets session cookie

User opens ProductB:

1. Redirect → PingFederate
2. Browser sends existing PF session cookie
3. No login screen
4. Immediate redirect back
5. ProductB logs user in

---

# 📋 Summary – Clear Responsibility Matrix

| Task                       | You         | Client |
| -------------------------- | ----------- | ------ |
| Install PingFederate       | ❌           | ✅      |
| Configure identity store   | ❌           | ✅      |
| Register OIDC clients      | ❌           | ✅      |
| Provide client credentials | ❌           | ✅      |
| Implement OIDC flow        | ✅           | ❌      |
| Token validation           | ✅           | ❌      |
| Local session creation     | ✅           | ❌      |
| User mapping logic         | ✅           | ❌      |
| Configure claims           | ⚠️ (define) | ✅      |
| Logout integration         | ✅           | ✅      |

---

# 🎯 What Changes In Your Code?

You will need to:

* Add OIDC middleware/library
* Add login redirect logic
* Add callback endpoint
* Add token validation
* Possibly modify user model
* Add external identity mapping
* Adjust logout flow

---

# 🚨 What You Should Clarify With Client Now

Ask them:

1. Are we using OIDC or SAML?
2. What attributes will be sent?
3. What is the issuer URL?
4. What is the token signing algorithm?
5. Do you require MFA?
6. Do you require specific session timeout?
7. Should logout be global?

---

# 🧠 Very Important Advice

Do NOT:

* Share sessions between ProductA/B/C manually
* Try to read PingFederate cookies
* Bypass redirect flow

Everything must go through standard OIDC flow.

