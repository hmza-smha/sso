Below is a **clear text-only flow chart** showing how authentication works with **PingFederate** and how the user ends up with **your own internal access token** for:

* ProductA
* ProductB

Assumptions:

* You use Authorization Code Flow
* Backend handles OIDC
* You issue your own 1-hour JWT after SSO
* Both products are registered as separate OIDC clients

---

# 🔐 FLOW 1 — User Logs Into ProductA

```
USER
  ↓
[ProductA Login Page]
  ↓
User clicks "Login with PingFederate"
  ↓
Frontend calls:
GET /auth/sso/login
  ↓
ProductA Backend:
  - Generate state
  - Generate nonce
  - Store temporarily
  - Redirect to PingFederate Authorization Endpoint
  ↓
Browser → PingFederate /authorize
  ↓
PingFederate:
  - Check for existing PF session cookie
      IF no session:
          → Show login page
          → User enters credentials
          → PF creates SSO session
      IF session exists:
          → Skip login
  ↓
PingFederate:
  - Generate authorization code
  - Redirect back to:
    https://ProductA.xyz.com/auth/sso/callback?code=XYZ&state=ABC
  ↓
ProductA Backend (Callback Endpoint):
  - Validate state
  - POST to Token Endpoint:
        grant_type=authorization_code
        client_id
        client_secret
        code
        redirect_uri
  ↓
PingFederate Token Endpoint:
  - Validate code
  - Return:
        id_token
        access_token (PF token)
        refresh_token (optional)
  ↓
ProductA Backend:
  - Validate id_token signature using JWKS
  - Validate issuer
  - Validate audience
  - Validate nonce
  - Extract user identity (sub, email)
  ↓
ProductA Backend:
  - Find or create local user
  - Generate YOUR internal JWT access token (1 hour)
  - Generate YOUR refresh token (6 hours)
  - Store refresh token
  - Set HTTP-only cookie (or return token)
  ↓
User now authenticated in ProductA
  ↓
User owns:
  - Your internal access token
  - Your refresh token
```

---

# 🔐 FLOW 2 — User Opens ProductB (SSO Scenario)

User is already logged into PingFederate (SSO cookie exists).

```
USER
  ↓
User opens ProductB.xyz.com
  ↓
ProductB Backend:
  - No local session detected
  - Redirect to PingFederate Authorization Endpoint
  ↓
Browser → PingFederate /authorize
  ↓
Browser automatically sends:
  PF SSO session cookie
  ↓
PingFederate:
  - Detects active session
  - Skips login screen
  - Issues new authorization code
  - Redirect back to:
    https://ProductB.xyz.com/auth/sso/callback?code=XYZ
  ↓
ProductB Backend:
  - Exchange code at Token Endpoint
  - Receive id_token
  - Validate signature via JWKS
  - Extract user identity
  ↓
ProductB Backend:
  - Find existing local user (matched by email/sub)
  - Generate YOUR internal access token (1 hour)
  - Generate YOUR refresh token
  - Store refresh token
  ↓
User now authenticated in ProductB
  ↓
User owns:
  - ProductB internal access token
  - ProductB refresh token
```

---

# 🔎 Important Observations

### 1️⃣ ProductA and ProductB DO NOT share tokens

Each product:

* Performs its own OIDC flow
* Generates its own internal JWT

---

### 2️⃣ SSO happens ONLY at PingFederate level

Because:

* PingFederate sets its own session cookie
* When ProductB redirects to PF
* PF sees session and skips login

Your systems never share cookies.

---

### 3️⃣ Final Token Ownership

After everything:

| System       | Token Owner       | Purpose                 |
| ------------ | ----------------- | ----------------------- |
| PingFederate | PF access token   | External identity proof |
| ProductA     | Your internal JWT | API authorization       |
| ProductB     | Your internal JWT | API authorization       |

You should only use your own tokens inside your systems.

---

# 🎯 Final Simplified View

```
ProductA → Redirect → PingFederate → Code → Validate → Issue Internal JWT
ProductB → Redirect → PingFederate (SSO) → Code → Validate → Issue Internal JWT
```

---


Tell me what you want to visualize next.
