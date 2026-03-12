# Authentication Flows — Frontend System Design Guide

> A comprehensive guide to authentication flows (Session, JWT, OAuth 2.0, SSO) from a **frontend engineer's perspective**.

> **See also:** [Frontend Security Guide](Frontend_Security.md) — XSS, CSRF, CORS, CSP, Secure Cookies, Input Sanitization & Frontend Auth Architecture Patterns.

---

## Table of Contents

1. [Authentication vs Authorization](#1-authentication-vs-authorization)
2. [Session-Based Authentication](#2-session-based-authentication)
3. [JWT (JSON Web Token) Authentication](#3-jwt-json-web-token-authentication)
4. [OAuth 2.0](#4-oauth-20)
5. [Single Sign-On (SSO)](#5-single-sign-on-sso)
6. [Comparison Table — Auth Flows](#6-comparison-table--auth-flows)
7. [Interview Cheat Sheet](#7-interview-cheat-sheet)

---

## 1. Authentication vs Authorization

| Concept            | Authentication (AuthN)            | Authorization (AuthZ)                 |
| ------------------ | --------------------------------- | ------------------------------------- |
| **What**           | *Who are you?*                    | *What can you do?*                    |
| **When**           | At login / token exchange         | On every protected resource access    |
| **Example**        | Entering username + password      | Checking if user has `admin` role     |
| **Frontend role**  | Collect credentials, store tokens | Guard routes, hide/show UI per role   |

---

## 2. Session-Based Authentication

### How It Works

```
┌──────────┐         ┌──────────────┐         ┌──────────┐
│  Browser  │──POST──▶│   Server     │──────▶  │   DB /   │
│           │ /login  │  (creates    │         │  Session  │
│           │◀─Set────│   session)   │◀────────│  Store    │
│           │ Cookie  │              │         │ (Redis)   │
└──────────┘         └──────────────┘         └──────────┘

Subsequent requests carry the session cookie automatically.
```

### Concept

1. User submits credentials (username + password).
2. Server validates, creates a **session** in a store (memory, Redis, DB).
3. Server sends back a **session ID** inside a `Set-Cookie` header.
4. Browser **automatically** attaches this cookie on every subsequent request.
5. Server reads session ID → looks up session store → identifies user.

### Code — Express Server (Session Auth)

```js
// server.js — session-based auth with express-session
const express = require("express");
const session = require("express-session");
const app = express();

app.use(express.json());
app.use(
  session({
    secret: "my-secret-key",
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,   // JS cannot access this cookie
      secure: true,     // only sent over HTTPS
      sameSite: "lax",  // CSRF protection
      maxAge: 1000 * 60 * 30, // 30 min
    },
  })
);

// Login
app.post("/login", (req, res) => {
  const { username, password } = req.body;
  const user = authenticateUser(username, password); // your DB check
  if (!user) return res.status(401).json({ error: "Invalid credentials" });

  req.session.userId = user.id; // store user in session
  res.json({ message: "Logged in" });
});

// Protected route
app.get("/profile", (req, res) => {
  if (!req.session.userId) return res.status(401).json({ error: "Not authenticated" });
  res.json({ userId: req.session.userId });
});

// Logout
app.post("/logout", (req, res) => {
  req.session.destroy(() => {
    res.clearCookie("connect.sid");
    res.json({ message: "Logged out" });
  });
});
```

### Code — Frontend (Session Auth)

```js
// login.js — browser side
async function login(username, password) {
  const res = await fetch("/login", {
    method: "POST",
    credentials: "include", // ← IMPORTANT: sends cookies cross-origin
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password }),
  });
  return res.json();
}

async function getProfile() {
  const res = await fetch("/profile", {
    credentials: "include", // cookie sent automatically
  });
  return res.json();
}
```

### Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Simple & battle-tested | Server must store session (stateful) |
| Revocation is instant (delete session) | Hard to scale horizontally without shared store (Redis) |
| Cookie sent automatically by browser | Vulnerable to CSRF if `SameSite` not set |
| HttpOnly cookie = safe from XSS token theft | Not ideal for mobile apps / cross-domain APIs |

---

## 3. JWT (JSON Web Token) Authentication

### How It Works (Overview)

```
┌──────────┐         ┌──────────────┐
│  Browser  │──POST──▶│   Server     │
│           │ /login  │  (creates    │
│           │◀─JSON───│   JWT)       │  No session store needed!
│  stores   │  token  │              │
│  token    │         └──────────────┘
└──────────┘
     │
     │  Authorization: Bearer <token>
     ▼
┌──────────────┐
│   Server     │  Verifies signature → extracts payload
│  (stateless) │
└──────────────┘
```

### JWT Auth Flow — Step by Step

```
┌──────────┐                              ┌──────────────┐
│  Browser  │                              │    Server     │
└────┬─────┘                              └──────┬───────┘
     │                                           │
     │──1. POST /login ─────────────────────────▶│
     │   { username, password }                  │
     │                                           │
     │                          2. Validate credentials against DB
     │                          3. Generate Access Token (short: 15m)
     │                             + Refresh Token (long: 7d)
     │                                           │
     │◀─4. Response ─────────────────────────────│
     │   Body: { accessToken: "eyJ..." }         │
     │   Set-Cookie: refreshToken=eyJ...         │
     │   (HttpOnly, Secure, SameSite)            │
     │                                           │
     │  5. Store access token                    │
     │     in memory (JS variable)               │
     │                                           │
     │──6. GET /api/profile ────────────────────▶│
     │   Header: Authorization: Bearer eyJ...    │
     │                                           │
     │                          7. Verify signature (HMAC/RSA)
     │                          8. Check exp claim (not expired?)
     │                          9. Extract payload { userId, role }
     │                                           │
     │◀─10. Response { user: { ... } } ─────────│
     │                                           │

── When Access Token Expires (after ~15 min) ──

     │──11. GET /api/data ──────────────────────▶│
     │   Header: Authorization: Bearer eyJ...    │
     │                                           │
     │◀─12. 401 Unauthorized (token expired) ────│
     │                                           │
     │──13. POST /refresh ──────────────────────▶│
     │   Cookie: refreshToken=eyJ... (auto-sent) │
     │                                           │
     │                         14. Verify refresh token
     │                         15. Issue NEW access token
     │                             (optionally rotate refresh token)
     │                                           │
     │◀─16. { accessToken: "new_eyJ..." } ──────│
     │                                           │
     │  17. Retry original request               │
     │      with new access token                │
     │──18. GET /api/data ──────────────────────▶│
     │   Header: Authorization: Bearer new_eyJ.. │
     │                                           │
     │◀─19. 200 OK { data: [...] } ─────────────│
```

**Key points:**
- **Step 1–4**: Login returns two tokens — a short-lived **access token** (in response body) and a long-lived **refresh token** (as HttpOnly cookie).
- **Step 5**: Access token is stored **in memory** (JS variable) — NOT in `localStorage`. This prevents XSS from stealing it.
- **Step 6–9**: Every API request sends the access token in the `Authorization: Bearer` header. Server verifies the signature and expiry **without** hitting a database — that's why it's **stateless**.
- **Steps 11–19**: When the access token expires, the client **silently** refreshes using the refresh token cookie. The user never sees a login screen. This is typically done via an **Axios interceptor** or a `fetchWithAuth` wrapper.

> **Why two tokens?** The access token is short-lived (15 min) to limit damage if stolen. The refresh token is long-lived but stored in an HttpOnly cookie (JS can't read it) and only sent to the `/refresh` endpoint.

### Concept

A JWT has **three parts** separated by dots:

```
header.payload.signature
```

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEsInJvbGUiOiJhZG1pbiJ9.HMAC_SIGNATURE
│                      │                                          │
│  Header              │  Payload (claims)                        │  Signature
│  { alg, typ }        │  { userId, role, exp, iat }              │  HMAC(header+payload, secret)
```

- **Header** — algorithm + token type
- **Payload** — claims (user data, expiry, issuer)  
- **Signature** — ensures the token was **not tampered with**

> ⚠️ JWT is **encoded**, NOT encrypted. Anyone can decode the payload. Never put secrets in it.

### Code — Server (JWT Auth)

```js
const jwt = require("jsonwebtoken");
const SECRET = process.env.JWT_SECRET; // keep this safe!

// Login — issue tokens
app.post("/login", (req, res) => {
  const user = authenticateUser(req.body.username, req.body.password);
  if (!user) return res.status(401).json({ error: "Invalid credentials" });

  const accessToken = jwt.sign(
    { userId: user.id, role: user.role },
    SECRET,
    { expiresIn: "15m" }  // short-lived
  );

  const refreshToken = jwt.sign(
    { userId: user.id },
    SECRET,
    { expiresIn: "7d" }   // long-lived
  );

  // Send refresh token as HttpOnly cookie
  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });

  res.json({ accessToken }); // access token in response body
});

// Middleware — verify JWT
function authMiddleware(req, res, next) {
  const authHeader = req.headers.authorization; // "Bearer <token>"
  if (!authHeader) return res.status(401).json({ error: "No token" });

  const token = authHeader.split(" ")[1];
  try {
    const decoded = jwt.verify(token, SECRET);
    req.user = decoded; // { userId, role, iat, exp }
    next();
  } catch (err) {
    return res.status(401).json({ error: "Invalid or expired token" });
  }
}

app.get("/profile", authMiddleware, (req, res) => {
  res.json({ user: req.user });
});
```

### Code — Frontend (JWT with Refresh Token Rotation)

```js
// auth.js — token management on the client
let accessToken = null;

async function login(username, password) {
  const res = await fetch("/login", {
    method: "POST",
    credentials: "include", // for refresh token cookie
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password }),
  });
  const data = await res.json();
  accessToken = data.accessToken; // store in memory (NOT localStorage)
}

// Wrapper with auto-refresh
async function fetchWithAuth(url, options = {}) {
  let res = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: `Bearer ${accessToken}`,
    },
  });

  // If 401, try refreshing
  if (res.status === 401) {
    const refreshRes = await fetch("/refresh", {
      method: "POST",
      credentials: "include", // sends refresh token cookie
    });

    if (refreshRes.ok) {
      const data = await refreshRes.json();
      accessToken = data.accessToken;

      // Retry original request with new token
      res = await fetch(url, {
        ...options,
        headers: {
          ...options.headers,
          Authorization: `Bearer ${accessToken}`,
        },
      });
    } else {
      // Refresh failed — force logout
      window.location.href = "/login";
    }
  }

  return res;
}
```

### Where to Store JWT on the Frontend?

| Storage | XSS Safe? | CSRF Safe? | Recommendation |
|---------|-----------|------------|---------------|
| `localStorage` | ❌ JS can read it | ✅ Not sent automatically | **Avoid** — XSS can steal tokens |
| `sessionStorage` | ❌ JS can read it | ✅ Not sent automatically | Slightly better (tab-scoped) |
| `HttpOnly Cookie` | ✅ JS cannot read it | ❌ Sent automatically | **Best** — combine with CSRF token |
| **In-memory variable** | ✅ Hard to steal | ✅ Not sent automatically | **Best for access token** — lost on refresh |

> **Best Practice**: Access token **in memory**, refresh token in **HttpOnly secure cookie**.

### Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Stateless — no server-side session store | Cannot revoke a specific token easily |
| Works great for APIs, mobile, microservices | Larger payload than session ID |
| Can carry claims (roles, permissions) | Must handle token refresh logic |
| Scales horizontally easily | If secret is leaked, all tokens are compromised |

---

## 4. OAuth 2.0

### Concept

OAuth 2.0 is a **delegation protocol** — it lets a user grant a third-party app limited access to their resources **without sharing their password**.

**Key Actors:**

| Role | Example |
|------|---------|
| **Resource Owner** | The user (you) |
| **Client** | Your frontend app |
| **Authorization Server** | Google, GitHub, Auth0 |
| **Resource Server** | Google API, GitHub API |

### Authorization Code Flow (Most Secure — for web apps)

```
┌──────────┐                        ┌─────────────────┐
│  Browser  │──1. Click "Login"────▶│  Authorization  │
│  (Client) │   (redirect directly  │  Server         │
│           │    to Auth Server)    │  (Google/GitHub) │
│           │                       │                 │
│           │──2. User logs in ────▶│                 │
│           │   & grants consent    │                 │
│           │◀─3. Redirect back ────│                 │
│           │   to YOUR app with    └─────────────────┘
│           │   AUTH CODE (in URL)
│           │
│           │──4. Send auth code──▶┌─────────────────┐
│           │   (to your backend)  │  Your Server    │
│           │                      │  (Backend)      │
│           │                      │                 │
│           │                      │──5. code→token──▶ Auth Server
│           │                      │   + client_secret│
│           │                      │◀─ access_token──│
│           │                      │   + id_token    │
│           │◀─6. Set session/JWT──│                 │
└──────────┘                      └─────────────────┘
```

**Key points:**
- **Step 1**: The browser redirects **directly** to the Authorization Server (Google, GitHub, etc.) — your backend is NOT involved yet.
- **Step 2**: The user logs in at the Auth Server's page (e.g. Google's login screen) and grants consent (e.g. "Allow this app to access your profile & email"). Your app **never** sees the user's Google/GitHub password.
- **Step 3**: After login + consent, the Auth Server redirects back to your app's **callback URL** with a short-lived **authorization code** in the query string (`?code=abc123`).
- **Steps 4–5**: Your **backend** exchanges the code for tokens. This is where the `client_secret` is used — it **never** leaves the server.
- **Why the code exchange?** The access token is **never exposed to the browser**. Only the backend sees it.

### Code — Step 1: Redirect User to OAuth Provider

```js
// frontend — initiate OAuth login
function loginWithGoogle() {
  const params = new URLSearchParams({
    client_id: "YOUR_CLIENT_ID",
    redirect_uri: "https://yourapp.com/callback",
    response_type: "code",          // Authorization Code flow
    scope: "openid profile email",
    state: generateRandomState(),   // CSRF protection
  });
  window.location.href = `https://accounts.google.com/o/oauth2/v2/auth?${params}`;
}

// Generate random state for CSRF mitigation
function generateRandomState() {
  const state = crypto.randomUUID();
  sessionStorage.setItem("oauth_state", state);
  return state;
}
```

### Code — Step 2: Handle Callback (Backend)

```js
// server.js — handle OAuth callback
app.get("/callback", async (req, res) => {
  const { code, state } = req.query;

  // Verify state to prevent CSRF
  if (state !== expectedState) return res.status(403).send("Invalid state");

  // Exchange authorization code for tokens
  const tokenRes = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      code,
      client_id: process.env.GOOGLE_CLIENT_ID,
      client_secret: process.env.GOOGLE_CLIENT_SECRET, // NEVER expose to frontend
      redirect_uri: "https://yourapp.com/callback",
      grant_type: "authorization_code",
    }),
  });

  const { access_token, id_token, refresh_token } = await tokenRes.json();

  // Decode id_token to get user info (or call /userinfo endpoint)
  const userInfo = jwt.decode(id_token); // { sub, email, name, picture }

  // Create session or issue your own JWT
  req.session.user = {
    id: userInfo.sub,
    email: userInfo.email,
    name: userInfo.name,
  };

  res.redirect("/dashboard");
});
```

### Code — Step 3: Protected Route on Frontend (React)

```jsx
// ProtectedRoute.jsx
import { Navigate } from "react-router-dom";
import { useAuth } from "./AuthContext";

export function ProtectedRoute({ children }) {
  const { user, isLoading } = useAuth();

  if (isLoading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" replace />;

  return children;
}

// Usage in router
<Route path="/dashboard" element={
  <ProtectedRoute>
    <Dashboard />
  </ProtectedRoute>
} />
```

### OAuth 2.0 Flow Variants

| Flow | Use Case | Token exposed to browser? |
|------|----------|--------------------------|
| **Authorization Code** | Server-side web apps | ❌ No (most secure) |
| **Authorization Code + PKCE** | SPAs, mobile apps (no client_secret) | Access token in memory |
| **Implicit** (deprecated) | Legacy SPAs | ✅ Yes (insecure) |
| **Client Credentials** | Machine-to-machine | N/A (no user) |
| **Resource Owner Password** (deprecated) | Trusted first-party apps only | ✅ Yes |

### PKCE (Proof Key for Code Exchange) — For SPAs

#### Why PKCE Exists

The standard Authorization Code flow relies on a `client_secret` to exchange the code for tokens. But **SPAs and mobile apps are public clients** — their code runs entirely on the user's device, so there's **no safe place to store a secret**.

Without PKCE, if an attacker intercepts the authorization code (e.g. via a malicious browser extension, redirect hijack, or OS-level URL scheme interception on mobile), they can exchange it for tokens since no secret is needed.

**PKCE solves this** by proving that the app that *started* the flow is the same app *finishing* it — using a one-time cryptographic challenge.

#### How PKCE Works (Step by Step)

```
┌──────────────┐                              ┌─────────────────┐
│  Browser/SPA │                              │  Authorization  │
│              │                              │  Server         │
│  1. Generate │                              │                 │
│  code_verifier                              │                 │
│  (random)    │                              │                 │
│              │                              │                 │
│  2. Hash it: │                              │                 │
│  code_challenge                             │                 │
│  = SHA256(   │                              │                 │
│    verifier) │                              │                 │
│              │                              │                 │
│  3. Redirect │──/authorize?──────────────▶  │                 │
│              │  code_challenge=abc123        │  4. Auth Server │
│              │  code_challenge_method=S256   │  stores the     │
│              │  client_id=...               │  code_challenge  │
│              │  redirect_uri=...            │  alongside the   │
│              │                              │  auth code       │
│              │  (User logs in & consents)   │                 │
│              │                              │                 │
│              │◀─redirect with AUTH CODE──── │                 │
│              │  ?code=xyz789                │                 │
│              │                              │                 │
│  5. Exchange │──POST /token ─────────────▶  │  6. Auth Server │
│     code     │  code=xyz789                 │  computes       │
│              │  code_verifier=original_rand │  SHA256(verifier)│
│              │  client_id=...               │  and checks:    │
│              │                              │  SHA256(verifier)│
│              │                              │  == stored       │
│              │                              │  code_challenge? │
│              │                              │                 │
│              │◀─ access_token ──────────────│  ✅ Match! Issue │
│              │   + id_token                 │  tokens          │
└──────────────┘                              └─────────────────┘
```

#### The Cryptographic Proof

```
code_verifier  = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"   ← random string (kept secret in browser)
                         │
                    SHA-256 hash
                         │
                         ▼
code_challenge = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"   ← sent to auth server

Attacker intercepts the auth code but does NOT have the code_verifier.
When they try to exchange the code → Auth server says: "Prove it. Send the verifier." → ❌ Fails.
```

| Without PKCE | With PKCE |
|-------------|-----------|
| Attacker intercepts `?code=xyz` | Attacker intercepts `?code=xyz` |
| Attacker calls `/token` with the code | Attacker calls `/token` with the code |
| ✅ Gets tokens (no secret needed) | ❌ Rejected — can't provide matching `code_verifier` |

#### Code — Generate PKCE Pair

```js
// pkce.js — generate code_verifier + code_challenge

async function generatePKCE() {
  // 1. Generate random code_verifier (43-128 chars, URL-safe)
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  const codeVerifier = base64URLEncode(array);

  // 2. Create code_challenge = SHA256(code_verifier), base64url-encoded
  const digest = await crypto.subtle.digest(
    "SHA-256",
    new TextEncoder().encode(codeVerifier)
  );
  const codeChallenge = base64URLEncode(new Uint8Array(digest));

  return { codeVerifier, codeChallenge };
}

function base64URLEncode(buffer) {
  return btoa(String.fromCharCode(...buffer))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
}
```

#### Code — Step 1: Start Auth Flow with Challenge

```js
async function loginWithPKCE() {
  const { codeVerifier, codeChallenge } = await generatePKCE();

  // Store verifier — we'll need it after redirect
  sessionStorage.setItem("pkce_verifier", codeVerifier);

  const params = new URLSearchParams({
    client_id: "YOUR_CLIENT_ID",
    redirect_uri: "https://yourapp.com/callback",
    response_type: "code",
    scope: "openid profile email",
    code_challenge: codeChallenge,         // ← hashed verifier
    code_challenge_method: "S256",          // ← tells server we used SHA-256
    state: crypto.randomUUID(),             // ← CSRF protection
  });

  // Redirect user to auth server
  window.location.href = `https://auth-server.com/authorize?${params}`;
}
```

#### Code — Step 2: Exchange Code with Verifier (After Redirect)

```js
async function handleCallback() {
  const params = new URLSearchParams(window.location.search);
  const code = params.get("code");
  const codeVerifier = sessionStorage.getItem("pkce_verifier");

  // Exchange auth code + code_verifier for tokens
  const res = await fetch("https://auth-server.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      code,
      redirect_uri: "https://yourapp.com/callback",
      client_id: "YOUR_CLIENT_ID",
      code_verifier: codeVerifier,  // ← proof we started the flow
    }),
  });

  if (!res.ok) throw new Error("Token exchange failed");

  const { access_token, id_token } = await res.json();

  // Clean up
  sessionStorage.removeItem("pkce_verifier");

  // Store access_token in memory (NOT localStorage)
  return { access_token, id_token };
}
```

#### When to Use PKCE

| Client Type | Has `client_secret`? | Use PKCE? |
|-------------|---------------------|-----------|
| **SPA** (React, Vue, Angular) | ❌ No | ✅ Yes (required) |
| **Mobile app** (iOS, Android) | ❌ No | ✅ Yes (required) |
| **Desktop app** (Electron) | ❌ No | ✅ Yes (required) |
| **Server-side web app** | ✅ Yes | ✅ Recommended (defense in depth) |
| **Machine-to-machine** | ✅ Yes | ❌ Not applicable (no user) |

> **Best Practice (2024+):** OAuth 2.1 draft spec **requires PKCE for ALL authorization code flows**, even for confidential clients with a `client_secret`. It's defense in depth.

---

## 5. Single Sign-On (SSO)

### What Is SSO?

SSO (Single Sign-On) allows a user to **authenticate once** with a central Identity Provider (IdP) and then access **multiple independent applications** without logging in again.

**Real-world example:**
- You log in at **accounts.google.com** once.
- You can now visit Gmail, YouTube, Google Drive, Google Docs — all without re-entering your password.
- That's SSO.

### Key Terminology

| Term | Meaning | Example |
|------|---------|---------|
| **IdP (Identity Provider)** | The central server that authenticates users and issues identity assertions | Okta, Auth0, Azure AD, Google Accounts |
| **SP (Service Provider)** | The app that trusts the IdP and relies on its assertions | Gmail, Your internal CRM, HR portal |
| **SSO Session** | A session cookie at the IdP's domain that proves "this user already authenticated" | Cookie on `accounts.google.com` |
| **App Session** | A local session/token in each SP after the IdP confirms identity | Cookie on `mail.google.com` |
| **Assertion** | The identity proof the IdP gives to the SP | SAML XML assertion or OIDC `id_token` (JWT) |
| **Federated Identity** | A user identity that is shared/trusted across security domains | Your company Google account working across all SaaS tools |

### How SSO Works — The Big Picture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│    App A      │    │    App B      │    │    App C      │
│  mail.com     │    │  drive.com    │    │  crm.com      │
│               │    │               │    │               │
│  Has its own  │    │  Has its own  │    │  Has its own  │
│  app session  │    │  app session  │    │  app session  │
└──────┬────────┘    └──────┬────────┘    └──────┬────────┘
       │                    │                    │
       │    "Is this user   │                    │
       │     logged in?"    │                    │
       └────────────────────┼────────────────────┘
                            │
                    ┌───────▼────────┐
                    │   Identity     │
                    │   Provider     │
                    │   (IdP)        │
                    │                │
                    │  ● Authenticates│
                    │    the user    │
                    │  ● Maintains   │
                    │    SSO session │
                    │  ● Issues      │
                    │    assertions  │
                    └────────────────┘
```

### SSO Flow — Step by Step (First App Login)

```
User visits App A for the first time (not logged in anywhere)

┌──────┐         ┌──────────┐         ┌─────────────────┐
│ User │         │  App A   │         │  IdP            │
│      │         │  (SP)    │         │  (sso.company   │
│      │         │          │         │   .com)         │
└──┬───┘         └────┬─────┘         └───────┬─────────┘
   │                  │                       │
   │──1. Visit app──▶│                       │
   │  (no session)    │                       │
   │                  │──2. Redirect to IdP──▶│
   │                  │  "User not authed,    │
   │                  │   go log in"          │
   │◀─────────────────│                       │
   │  (302 redirect)  │                       │
   │                  │                       │
   │──3. Browser goes to IdP ──────────────▶ │
   │   (no SSO session cookie yet)           │
   │                  │                       │
   │◀─4. IdP shows login page ──────────────│
   │                  │                       │
   │──5. User enters credentials ──────────▶│
   │   (username + password / MFA)           │
   │                  │                       │
   │                  │         6. IdP validates credentials
   │                  │            Creates SSO SESSION COOKIE
   │                  │            on sso.company.com domain
   │                  │                       │
   │◀─7. Redirect back to App A ─────────── │
   │   WITH identity assertion               │
   │   (auth code, SAML assertion, or token) │
   │                  │                       │
   │──8. Land at ───▶│                       │
   │   App A callback │                       │
   │                  │──9. Validate assertion│
   │                  │   (or exchange code   │
   │                  │    for tokens)        │
   │                  │                       │
   │                  │──10. Create LOCAL ─── │
   │                  │   app session for     │
   │                  │   App A               │
   │                  │                       │
   │◀─11. Logged in!──│                       │
   │   (App A session │                       │
   │    cookie set)   │                       │
```

### SSO Flow — Subsequent App (Already Logged In at IdP)

This is where the **magic** happens:

```
User visits App B (already authenticated at IdP from App A login)

┌──────┐         ┌──────────┐         ┌─────────────────┐
│ User │         │  App B   │         │  IdP            │
│      │         │  (SP)    │         │  (sso.company   │
│      │         │          │         │   .com)         │
└──┬───┘         └────┬─────┘         └───────┬─────────┘
   │                  │                       │
   │──1. Visit app──▶│                       │
   │  (no App B      │                       │
   │   session)      │                       │
   │                  │──2. Redirect to IdP──▶│
   │◀─────────────────│                       │
   │                  │                       │
   │──3. Browser goes to IdP ──────────────▶ │
   │   (SSO session cookie EXISTS! ✅)       │
   │                  │                       │
   │                  │         4. IdP checks SSO session
   │                  │            → User already authenticated!
   │                  │            → NO login page shown
   │                  │                       │
   │◀─5. IMMEDIATELY redirect back ──────── │
   │   to App B with assertion               │
   │   (user never saw a login screen!)      │
   │                  │                       │
   │──6. Land at ───▶│                       │
   │   App B callback │                       │
   │                  │──7. Validate, create──│
   │                  │   App B session       │
   │                  │                       │
   │◀─8. Logged in!──│                       │
   │   (seamless!)   │                       │
```

> **Key insight**: The SSO magic is the **IdP session cookie**. When the browser is redirected to the IdP, the browser automatically sends this cookie. The IdP sees "oh, this user already authenticated" and immediately redirects back — no login screen.

### SSO Protocols Compared

| | **SAML 2.0** | **OpenID Connect (OIDC)** | **CAS** |
|---|---|---|---|
| **Format** | XML assertions | JWT tokens (built on OAuth 2.0) | Ticket-based |
| **Transport** | Browser redirect (POST/Redirect binding) | Browser redirect + backend token exchange | Browser redirect |
| **Token** | SAML Assertion (XML, signed) | `id_token` (JWT) + `access_token` | Service Ticket |
| **Common In** | Enterprise (Okta, ADFS, Salesforce) | Modern web/mobile (Auth0, Google, Azure) | Universities |
| **Complexity** | High (XML parsing, certificate management) | Medium (JSON, standard HTTP) | Low-Medium |
| **Mobile Friendly** | ❌ Poor (XML, browser-dependent) | ✅ Great (JSON, token-based) | ❌ Poor |
| **Year Introduced** | 2005 | 2014 | 2004 |

### SAML 2.0 SSO Flow (Enterprise)

```
┌──────┐         ┌──────────┐         ┌─────────────────┐
│ User │         │  SP      │         │  IdP            │
│      │         │(your app)│         │  (Okta/ADFS)    │
└──┬───┘         └────┬─────┘         └───────┬─────────┘
   │──Visit app──────▶│                       │
   │                  │──SAML AuthnRequest───▶│  (XML, base64-encoded
   │                  │  (who are you? prove  │   in URL or POST form)
   │                  │   this user's identity)│
   │                  │                       │
   │   (User logs in at IdP if needed)       │
   │                  │                       │
   │                  │◀─SAML Response────────│  (XML, digitally signed)
   │                  │  Contains:            │
   │                  │  • NameID (user email) │
   │                  │  • Attributes (roles) │
   │                  │  • Signature          │
   │                  │  • Conditions (expiry)│
   │                  │                       │
   │                  │  SP validates XML     │
   │                  │  signature against    │
   │                  │  IdP's public cert    │
   │                  │                       │
   │◀─Logged in!──────│                       │
```

### OIDC SSO Flow (Modern — Recommended)

```
┌──────┐         ┌──────────┐         ┌─────────────────┐
│ User │         │  SP      │         │  IdP            │
│      │         │(your app)│         │  (Auth0/Google)  │
└──┬───┘         └────┬─────┘         └───────┬─────────┘
   │──Visit app──────▶│                       │
   │                  │──/authorize───────────▶│  (standard OAuth 2.0
   │                  │  response_type=code   │   authorization request)
   │                  │  scope=openid profile │
   │                  │                       │
   │   (User logs in at IdP if needed)       │
   │                  │                       │
   │                  │◀─?code=abc123─────────│  (redirect with auth code)
   │                  │                       │
   │                  │──POST /token──────────▶│  (backend exchanges code)
   │                  │  code + client_secret │
   │                  │                       │
   │                  │◀─────────────────────│
   │                  │  {                    │
   │                  │    access_token: "...",│
   │                  │    id_token: "eyJ...", │ ← JWT with user identity
   │                  │    refresh_token:"..." │
   │                  │  }                    │
   │                  │                       │
   │                  │  Decode id_token:     │
   │                  │  { sub, email, name,  │
   │                  │    iss, aud, exp }    │
   │                  │                       │
   │◀─Logged in!──────│                       │
```

### SSO Session vs App Session — The Two Layers

This is a crucial concept people often miss:

```
┌─────────────────────────────────────────────────────┐
│                    IdP Domain                        │
│                 sso.company.com                      │
│                                                     │
│   ┌─────────────────────────────────────┐           │
│   │  SSO Session Cookie                 │           │
│   │  (e.g. "sso_sid=abc123")           │           │
│   │  Domain: sso.company.com           │           │
│   │  Lives as long as IdP decides      │           │
│   │  (e.g. 8 hours for a work day)     │           │
│   └─────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│  App A Domain       │  │  App B Domain       │  │  App C Domain       │
│  mail.company.com   │  │  crm.company.com    │  │  hr.company.com     │
│                     │  │                     │  │                     │
│  App Session Cookie │  │  App Session Cookie │  │  App Session Cookie │
│  (independent!)     │  │  (independent!)     │  │  (independent!)     │
│  maxAge: 30min      │  │  maxAge: 1hr        │  │  maxAge: 15min      │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
```

| Layer | Where | Controlled By | Purpose |
|-------|-------|--------------|---------|
| **SSO Session** | IdP's domain cookie | IdP (Okta, Auth0) | Proves "this user authenticated at the IdP" |
| **App Session** | Each app's own cookie/token | Each SP individually | Proves "this user is logged into THIS app" |

> **Important**: Deleting App A's session does NOT affect App B or the SSO session. Deleting the SSO session means future redirects to the IdP will require re-login, but existing app sessions remain until they expire.

### How the Frontend Triggers SSO

From the frontend's perspective, initiating SSO is just a **redirect**:

```
window.location.href = "https://sso.company.com/authorize?client_id=...&redirect_uri=...&scope=openid"
```

Key parameters in the redirect URL:

| Parameter | Purpose |
|-----------|---------|
| `client_id` | Identifies your app to the IdP |
| `redirect_uri` | Where to send the user after auth |
| `response_type=code` | Requests an authorization code |
| `scope=openid profile email` | What user info you want |
| `state` | Random string for CSRF protection |
| `prompt=none` | Skip login screen if SSO session exists (silent check) |
| `prompt=login` | Force login screen even if SSO session exists |

After authentication, the IdP redirects back to your `redirect_uri` with an authorization code. Your **backend** exchanges this code for tokens (same as OAuth 2.0 Authorization Code flow).

### The `prompt` Parameter — Key to SSO UX

| Value | Behavior | Use Case |
|-------|----------|----------|
| *(omitted)* | Show login if no SSO session, skip if session exists | Default — best for most cases |
| `none` | **Never** show login UI. If no SSO session → return `error=login_required` | Silent checks, iframe-based renewal |
| `login` | **Always** show login UI, even if SSO session exists | Force re-authentication (sensitive actions) |
| `consent` | Always show consent screen | Re-authorize permissions |

### Silent Token Renewal

When the access token expires, you can **silently** get a new one without showing a login screen — as long as the SSO session at the IdP is still active.

**How it works:**

```
┌──────────────┐                          ┌──────────────┐
│  Your App    │                          │     IdP      │
│              │                          │              │
│  Token       │   ┌─────────────────┐    │              │
│  expired!    │──▶│ Hidden <iframe>  │──▶│              │
│              │   │ src=/authorize?  │   │  SSO cookie  │
│              │   │ prompt=none      │   │  exists? ✅  │
│              │   │                  │◀──│  Issue new   │
│              │   │ Gets code via    │   │  auth code   │
│              │   │ redirect in      │   │              │
│              │   │ iframe           │   │              │
│              │   └────────┬────────┘    │              │
│              │            │             │              │
│              │◀─postMessage(token)──    │              │
│              │                          │              │
│  New token!  │                          │              │
│  Continue    │                          │              │
│  working     │                          │              │
└──────────────┘                          └──────────────┘
```

**The process:**
1. Your app creates a **hidden iframe** pointing to the IdP's `/authorize` endpoint with `prompt=none`.
2. The browser sends the **IdP's SSO session cookie** along with the iframe request.
3. If the SSO session is valid, the IdP redirects the iframe to your callback URL with a new auth code.
4. The callback page inside the iframe uses `postMessage` to send the code/token back to the parent window.
5. Your app schedules the next renewal ~60 seconds before the new token expires.

**If the SSO session has expired:** The IdP returns `error=login_required`, and your app redirects the user to the full login page.

> **Note:** Some modern IdPs also support **refresh tokens** as an alternative to iframe-based silent renewal, avoiding iframe-related issues (CSP restrictions, third-party cookie blocking).

### SSO Logout — The Hard Part

Logout in SSO is more complex than login because you need to consider **all apps** the user accessed.

**The problem:** User clicks "Logout" in App A. But they also have active sessions in App B and App C. What happens to those?

#### Logout Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Local Logout Only** | Destroy only the current app's session | Simple | User still logged in to other apps & IdP |
| **IdP Logout (RP-Initiated)** | Redirect user to IdP's `/logout` endpoint | Clears SSO session, prevents future silent logins | Other apps still have active local sessions until they expire |
| **Front-Channel Logout** | IdP renders a page with hidden iframes to each SP's logout URL | All apps get notified via browser | Unreliable (blocked iframes, CSP rules, ad blockers) |
| **Back-Channel Logout** | IdP sends server-to-server HTTP POST to each SP's logout webhook | Most reliable, no browser dependency | Each SP must expose a logout endpoint |

#### Front-Channel Logout (via browser)

```
User clicks "Logout"
         │
         ▼
┌────────────────────┐
│  IdP Logout Page   │
│                    │
│  Renders hidden    │
│  iframes:          │
│  ┌───────────────┐ │
│  │ <iframe>      │ │    → app-a.com/logout → destroys App A session
│  │ <iframe>      │ │    → app-b.com/logout → destroys App B session
│  │ <iframe>      │ │    → app-c.com/logout → destroys App C session
│  └───────────────┘ │
│                    │
│  Then redirect to  │
│  post-logout URL   │
└────────────────────┘
```

#### Back-Channel Logout (server-to-server — most reliable)

```
┌─────────────────┐
│      IdP        │
│                 │
│  User logged    │──POST → app-a.com/backchannel-logout  { logout_token: "eyJ..." }
│  out            │──POST → app-b.com/backchannel-logout  { logout_token: "eyJ..." }
│                 │──POST → app-c.com/backchannel-logout  { logout_token: "eyJ..." }
└─────────────────┘

Each SP validates the logout_token JWT → destroys the matching session.
```

#### Best Practice for SSO Logout

1. Redirect user to IdP's `/logout` endpoint (RP-initiated logout).
2. Configure **back-channel logout** on each SP so the IdP can notify them server-to-server.
3. Keep app session lifetimes **short** (15–60 min) as a safety net — even if logout notification fails, sessions expire quickly.

### SSO — Common Real-World Architectures

| Architecture | Example | How Apps Share Identity |
|-------------|---------|----------------------|
| **Subdomain SSO** | `*.company.com` | Shared cookie on `.company.com` domain |
| **Central IdP (OIDC/SAML)** | Okta + various SaaS apps | Each app redirects to Okta for auth |
| **Token-based SSO** | Microservices behind API gateway | Gateway validates JWT, forwards claims |
| **Social Login SSO** | "Login with Google" across indie apps | Each app is a separate OIDC client |

### SSO Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Users log in once, access everything | Single point of failure (IdP goes down = all apps down) |
| Centralized user management (disable one account = disabled everywhere) | Complex logout (must notify all SPs) |
| Consistent security policies (MFA, password rules) in one place | Increased security risk if IdP is compromised |
| Better UX — no password fatigue | Initial setup complexity (SAML certs, OIDC config) |
| Easier compliance & audit trail | Token/session lifetime management across apps |

### SSO Interview Quick Reference

**Q: How does SSO work without sharing cookies across domains?**
> The IdP maintains its own session cookie on its domain. Each app **redirects** to the IdP. The browser sends the IdP's cookie automatically. If valid, the IdP redirects back with an assertion — no cross-domain cookie sharing needed.

**Q: What happens if the SSO session expires but an app session is still active?**
> The app continues working until its own session expires. Next time the app tries to silently renew via the IdP, it will fail (`login_required`), and the user will need to re-authenticate.

**Q: SAML vs OIDC — when to use which?**
> **SAML**: Legacy enterprise integrations, apps that only support SAML (Salesforce, older SaaS). **OIDC**: Everything new — SPAs, mobile apps, microservices. It's simpler, JSON-based, and mobile-friendly.

**Q: How do you handle SSO logout properly?**
> Best approach: **back-channel logout** (IdP sends server-to-server calls to each SP). Fallback: RP-initiated logout (redirect user to IdP `/logout` which triggers front-channel iframes). Always set short app session lifetimes as a safety net.

---

## 6. Comparison Table — Auth Flows

| Feature | Session-Based | JWT | OAuth 2.0 | SSO (OIDC) |
|---------|:------------:|:---:|:---------:|:----------:|
| **Stateful/Stateless** | Stateful | Stateless | Depends | Depends |
| **Token Storage** | Cookie (session ID) | Memory / Cookie | Backend stores tokens | Cookie / Memory |
| **Scalability** | Needs shared session store | Excellent | Excellent | Excellent |
| **Revocation** | Instant | Hard (need blocklist) | Revoke at auth server | Revoke at IdP |
| **Best For** | Traditional web apps | APIs, microservices | Third-party login | Enterprise multi-app |
| **Complexity** | Low | Medium | High | High |
| **Cross-Domain** | Hard (cookies) | Easy (header-based) | Built for it | Built for it |

---

## 7. Interview Cheat Sheet

### Quick Answers for System Design Interviews

**Q: Where should I store tokens on the frontend?**
> Access token in **memory** (JS variable). Refresh token in an **HttpOnly, Secure, SameSite cookie**.

**Q: JWT vs Session — when to pick which?**
> **Session** for traditional server-rendered apps (simple, instant revocation).  
> **JWT** for APIs serving multiple clients (mobile, SPA, microservices) where statelessness matters.

**Q: What is PKCE and why?**
> PKCE prevents authorization code interception for **public clients** (SPAs, mobile) that can't securely store a `client_secret`.

**Q: How does silent token renewal work in SSO?**
> Hidden iframe hits the IdP's `/authorize` endpoint with `prompt=none`. If the SSO session is still active, a new token is returned via `postMessage`.

**Q: OAuth 2.0 vs OpenID Connect?**
> OAuth 2.0 handles **authorization** (what can you access). OIDC adds an **identity layer** on top (who are you) via the `id_token`.

---

> **Further Reading:**
> - [OAuth 2.0 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
> - [JWT RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)
> - [OWASP Top 10](https://owasp.org/www-project-top-ten/)
> - [Frontend Security Guide](Frontend_Security.md) — XSS, CSRF, CORS, CSP, and more
