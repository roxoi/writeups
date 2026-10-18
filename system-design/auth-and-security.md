---
title: "Authentication and Security in System Design — Sessions, JWT, OAuth, Authorization, and Common Attacks"
description: "A practical guide to authentication and security for system design: password storage, sessions vs tokens, cookies, JWT pitfalls, OAuth 2.0 and OpenID Connect, SSO, RBAC and ABAC, API security, secrets, encryption, and the common web attacks."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/session-token.png"
tags: [security, authentication, authorization, oauth, jwt, sessions, system-design]
keywords: ["Authentication vs authorization", "JWT vs session", "OAuth 2.0 authorization code PKCE", "OpenID Connect", "RBAC vs ABAC", "Password hashing bcrypt argon2", "System design security"]
---

# Authentication and Security in System Design — Sessions, JWT, OAuth, Authorization, and Common Attacks

Security is part of every design, and interviewers expect you to mention it: "How do users log in?", "How do services trust each other?", "Where do you store secrets?" This guide covers the essentials: how authentication works, sessions and tokens, JWT and OAuth, authorization, protecting data, and the common attacks.

![Sessions and Tokens](/images/session-token.png)

## Authentication vs Authorization

|              | Authentication (AuthN)                        | Authorization (AuthZ)                                   |
| ------------ | --------------------------------------------- | ------------------------------------------------------- |
| Question     | **Who are you?**                              | **What are you allowed to do?**                         |
| Happens      | First                                         | After authentication                                    |
| Examples     | Password, OTP, passkey, social login          | Roles, permissions, "can this user edit this document?" |
| Failure code | `401 Unauthorized` (really "unauthenticated") | `403 Forbidden`                                         |

Both must be enforced on the **server**. Anything checked only in the client (hiding a button) can be bypassed.

## Authentication Basics

### Storing Passwords

- **Never store plain-text passwords, and never use a fast hash** (MD5, SHA-1, plain SHA-256).
- Use a **slow, salted, purpose-built password hash**: **Argon2id** (preferred), **bcrypt**, or **scrypt**. They are deliberately expensive, so attackers can't try billions of guesses per second.
- A **salt** (random, unique per user) ensures identical passwords hash differently and defeats precomputed rainbow tables.
- Optionally add a **pepper** (a secret key stored outside the database).
- Enforce rate limits and lockouts on login (see below), and check new passwords against known-breached lists.

### Multi-Factor Authentication (MFA)

Combine something you **know** (password), something you **have** (phone, hardware key), and something you **are** (biometrics).

- **TOTP** (authenticator app) and **hardware keys / passkeys (WebAuthn)** are much stronger than SMS codes, which can be intercepted or SIM-swapped.
- **Passkeys** replace passwords with public-key cryptography: the server stores only a public key, and the private key never leaves the user's device, which makes phishing and credential theft far harder.

### Protecting the Login Endpoint

- **Rate limit** by account and IP, and slow down repeated failures (see [Rate Limiting](system-design/rate-limiting)).
- Use **uniform error messages** ("invalid email or password") so attackers can't tell which accounts exist.
- Defend against **credential stuffing** (using leaked passwords from other sites) with MFA, breached-password checks, and bot detection.
- Always use **HTTPS**.

## Sessions vs Tokens

After login, the server must recognize the user on every request. There are two main approaches.

### Server-Side Sessions

1. The user logs in; the server creates a **session** (a record with user ID and expiry) in a store such as Redis.
2. It sends the client a random **session ID** in a cookie.
3. On each request, the server looks up the session ID in the store.

**Pros:** easy to **revoke** instantly (delete the session), the client holds no sensitive data, simple to reason about.
**Cons:** a lookup on every request, and a shared session store that must be highly available (it becomes a dependency for every service).

### Token-Based (Stateless) Authentication

The server issues a **signed token** (commonly a JWT) containing the user's identity and claims. Services verify the signature **without a database lookup**.

**Pros:** no shared session store, scales easily across many services, convenient for APIs and microservices.
**Cons:** **hard to revoke** before expiry (the token is valid until it expires), and tokens can get large.

### Comparison

|                  | Server-side session  | Stateless token (JWT)                    |
| ---------------- | -------------------- | ---------------------------------------- |
| State            | Stored on server     | Carried by the client                    |
| Revocation       | Immediate            | Hard (needs short expiry or a deny list) |
| Per-request cost | Store lookup         | Signature check only                     |
| Scaling          | Needs shared store   | Easy                                     |
| Best for         | Traditional web apps | APIs, microservices, mobile              |

Many systems use **both**: a short-lived access token for API calls plus a longer-lived, revocable **refresh token** stored server-side.

### Cookies: Get the Flags Right

When using cookies for sessions or tokens, set:

| Flag                                    | Purpose                                                        |
| --------------------------------------- | -------------------------------------------------------------- |
| `HttpOnly`                              | JavaScript can't read the cookie, which limits damage from XSS |
| `Secure`                                | Sent only over HTTPS                                           |
| `SameSite=Lax` or `Strict`              | Not sent on most cross-site requests, which helps against CSRF |
| Short expiry, plus rotation after login | Limits the window for stolen cookies                           |

## JSON Web Tokens (JWT)

A JWT has three Base64URL-encoded parts separated by dots: **header . payload . signature**.

```text
header:    { "alg": "RS256", "typ": "JWT" }
payload:   { "sub": "user-123", "iss": "auth.example.com", "aud": "api", "exp": 1760000000, "roles": ["editor"] }
signature: signs the header and payload with a secret or private key
```

- The payload is **encoded, not encrypted**. Anyone can read it, so **never put secrets or sensitive data in a JWT**.
- The **signature** proves it hasn't been tampered with.
- **HS256** uses one shared secret (every verifier can also forge tokens); **RS256/ES256** use a private key to sign and a public key to verify, which is safer when many services verify tokens.

### Standard Claims

`iss` (issuer), `sub` (subject/user), `aud` (intended audience), `exp` (expiry), `iat` (issued at), `nbf` (not before), `jti` (unique ID).

### Common JWT Mistakes

| Mistake                                                                                        | Fix                                                                                                  |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Accepting any algorithm, including `none`, or trusting the algorithm named in the token header | Fix the allowed algorithm(s) on the server                                                           |
| Not checking `exp`, `iss`, and `aud`                                                           | Validate all of them on every request                                                                |
| Long-lived access tokens                                                                       | Use short lifetimes (minutes), with refresh tokens                                                   |
| Putting sensitive data in the payload                                                          | Keep only what's needed; the payload is readable                                                     |
| Storing tokens in `localStorage` (readable by any XSS script)                                  | Prefer `HttpOnly` cookies where possible                                                             |
| No way to revoke                                                                               | Short expiry, refresh-token rotation, and a deny list for emergencies                                |
| Weak or leaked signing keys                                                                    | Strong keys, stored in a secret manager, with **key rotation** (use a `kid` header to identify keys) |

### Refresh Tokens

- Access token: short-lived (for example 5–15 minutes), sent with every API call.
- Refresh token: long-lived, used **only** to get new access tokens, stored securely, and **rotated** on each use (the old one becomes invalid). If an old refresh token is reused, treat it as theft and revoke the whole token family.

## OAuth 2.0 and OpenID Connect

**OAuth 2.0** is a framework for **delegated authorization**: letting an app access a user's resources at another service **without sharing the user's password** ("Allow this app to read your Google Calendar").

**OpenID Connect (OIDC)** is a layer on top of OAuth 2.0 that adds **authentication**: it tells the app **who the user is** via an **ID token** (a JWT). "Sign in with Google" is OIDC.

### Roles

| Role                     | Meaning                                                                       |
| ------------------------ | ----------------------------------------------------------------------------- |
| **Resource Owner**       | The user                                                                      |
| **Client**               | The application requesting access                                             |
| **Authorization Server** | Authenticates the user and issues tokens (for example Google, Okta, Keycloak) |
| **Resource Server**      | The API that holds the data and accepts access tokens                         |

### Authorization Code Flow with PKCE (the standard for web and mobile apps)

```text
1. App → redirects the user to the Authorization Server
        (client_id, requested scopes, redirect_uri, state, code_challenge)
2. User logs in and approves the requested permissions
3. Authorization Server → redirects back to the app with a short-lived authorization code
4. App → sends the code (+ code_verifier) to the token endpoint, over a back channel
5. Authorization Server → returns an access token (and refresh token / ID token)
6. App → calls the API with:  Authorization: Bearer <access token>
```

- **PKCE** (Proof Key for Code Exchange) protects the code from interception by binding it to a one-time secret the app created. It is recommended for **all** clients, and essential for mobile and single-page apps that can't keep a secret.
- The **`state`** parameter protects against CSRF in the flow.
- The **implicit flow** (tokens directly in the redirect) and the **password grant** are discouraged and removed from current best practice.

### Other Flows

| Flow                   | Use                                                               |
| ---------------------- | ----------------------------------------------------------------- |
| **Client credentials** | Machine-to-machine (service calls an API as itself, with no user) |
| **Device code**        | Devices without a good browser (smart TVs, CLIs)                  |
| **Refresh token**      | Get new access tokens without asking the user again               |

### Scopes

Scopes limit what a token can do (`calendar.read`, `profile`). Request the **minimum** needed (least privilege).

### Single Sign-On (SSO)

One login gives access to many applications. Typically built on **OIDC** (modern) or **SAML** (common in enterprises), with a central identity provider. Benefits: fewer passwords, central control and MFA, easy offboarding.

## Authorization Models

| Model                          | How it works                                                                                             | Good for                                       |
| ------------------------------ | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| **ACL** (access control list)  | A list per resource of who can do what                                                                   | Simple per-resource sharing (file permissions) |
| **RBAC** (role-based)          | Users get roles (admin, editor, viewer); roles carry permissions                                         | Most business applications                     |
| **ABAC** (attribute-based)     | Decisions use attributes of the user, resource, and context ("manager, same department, business hours") | Fine-grained, dynamic rules                    |
| **ReBAC** (relationship-based) | Access follows relationships ("owner of the folder → can edit files in it")                              | Sharing models like Google Docs                |

**Principles:**

- **Least privilege:** grant the minimum access needed.
- **Deny by default.**
- **Check on every request**, on the server, close to the data, and always verify the user **owns or may access the specific object** (to avoid insecure direct object references, where changing an ID in the URL exposes someone else's data).

## Where to Enforce Security in a System

```text
Client ──► CDN / WAF / DDoS protection ──► Load Balancer ──► API Gateway ──► Services ──► Database
          (block bad traffic)            (TLS)           (authN, rate limit)  (authZ)   (encrypted, least privilege)
```

- **Edge (CDN/WAF):** filters common attacks and absorbs volumetric DDoS.
- **API gateway:** validates tokens, applies rate limits, and passes verified identity to services (see [API Gateway](system-design/api-gateway) and [Rate Limiting](system-design/rate-limiting)).
- **Services:** still **authorize** each action for their own data. Don't assume traffic behind the gateway is trusted (defense in depth).
- **Service-to-service:** use **mutual TLS (mTLS)** and service identities, often through a service mesh.
- **Data layer:** encryption, least-privilege database accounts, and network isolation.

**Zero trust** is the idea of "never trust, always verify": every request is authenticated and authorized, regardless of where it comes from in the network.

## Protecting Data

### Encryption in Transit

- Use **TLS** everywhere: client to edge, **and** between internal services. Enable HSTS to force HTTPS.
- Keep certificates automated and monitored so they don't expire unexpectedly.
- Use **mTLS** between services when each side must prove its identity.

### Encryption at Rest

- Encrypt disks, databases, and backups, usually with keys held in a **KMS** (key management service).
- Use **envelope encryption**: data is encrypted with a data key, and the data key is encrypted with a master key in the KMS, so you can rotate keys without re-encrypting everything.
- For very sensitive fields (card numbers, national IDs), use **field-level encryption or tokenization**, and avoid storing them at all when possible (use a payment processor).

### Secrets Management

- **Never** commit secrets to source control or bake them into images.
- Store them in a **secrets manager** (HashiCorp Vault, AWS Secrets Manager, cloud KMS), inject them at runtime, and **rotate** them regularly.
- Give each service its own credentials with only the permissions it needs.
- Scan repositories and logs for leaked secrets.

### Privacy Basics

Collect only the data you need, define **retention** limits, support deletion requests, and avoid putting personal data in logs.

## Common Attacks and Defenses

| Attack                                                  | What happens                                                                           | Main defenses                                                                                    |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **SQL injection**                                       | Attacker's input becomes part of a database query                                      | Parameterized queries / prepared statements, ORM, least-privilege DB accounts                    |
| **XSS** (cross-site scripting)                          | Malicious script runs in a victim's browser                                            | Output encoding, Content Security Policy, `HttpOnly` cookies, sanitize HTML                      |
| **CSRF** (cross-site request forgery)                   | A malicious site makes the victim's browser send authenticated requests                | `SameSite` cookies, CSRF tokens, checking origin                                                 |
| **SSRF** (server-side request forgery)                  | The server is tricked into requesting internal URLs (metadata services, internal APIs) | Allow-list outbound destinations, block internal address ranges, don't fetch arbitrary user URLs |
| **Broken access control / IDOR**                        | User accesses others' data by changing an ID                                           | Authorize every object access on the server                                                      |
| **Brute force / credential stuffing**                   | Automated login guessing                                                               | Rate limiting, MFA, breached-password checks, lockouts                                           |
| **Session hijacking / fixation**                        | Stolen or planted session IDs                                                          | HTTPS, secure cookie flags, regenerate the session after login                                   |
| **Clickjacking**                                        | A page is hidden in an invisible frame to trick clicks                                 | `X-Frame-Options` or CSP `frame-ancestors`                                                       |
| **DDoS**                                                | Overwhelm the service with traffic                                                     | CDN/Anycast, WAF, rate limiting, autoscaling, upstream scrubbing                                 |
| **Man-in-the-middle**                                   | Traffic intercepted                                                                    | TLS, HSTS, certificate validation                                                                |
| **Dependency / supply-chain attacks**                   | A compromised library or build                                                         | Pin and scan dependencies, signed artifacts, minimal images                                      |
| **Insecure deserialization / injection of other kinds** | Untrusted data executed as code or commands                                            | Validate input, avoid unsafe deserialization, escape for the right context                       |

The **OWASP Top 10** is the standard checklist of the most common web risks; mention it in interviews.

**Input handling rule:** validate and sanitize all input on the server, treat everything from the client as untrusted, and escape output for the context where it is used (HTML, SQL, shell).

## Auditing and Detection

- Keep **audit logs** for sensitive actions (logins, permission changes, data exports) in tamper-resistant storage.
- Monitor for anomalies (many failed logins, access from new locations) and alert, using the tools in [Observability and Deployment](system-design/observability-and-deployment).
- Keep dependencies patched, and have an incident response plan.

## Designing a Login System: A Worked Outline

1. **Sign-up:** validate input, hash the password with Argon2id, store the hash and salt, send a verification email.
2. **Login:** rate limit; verify the hash; check MFA if enabled; on success, issue a short-lived **access token** (JWT) and a **refresh token** (stored server-side, hashed, rotated on use).
3. **API requests:** the gateway validates the access token (signature, `exp`, `iss`, `aud`) and forwards identity to services, which authorize each action.
4. **Logout / revocation:** delete the refresh token (and the whole token family if reuse is detected); the short access-token lifetime limits remaining exposure.
5. **Password reset:** one-time, short-lived, single-use token sent to a verified channel; invalidate existing sessions after reset.
6. **Social login:** use OIDC authorization code flow with PKCE, and link the external identity to the account.
7. **Protect everything:** HTTPS, secure cookie flags, rate limits, audit logs, and alerts on suspicious activity.

## Interview Questions

**Q: What's the difference between authentication and authorization?**
Authentication verifies who you are; authorization decides what you may do. Authentication comes first, and both must be enforced server-side.

**Q: Sessions or JWT?**
Server-side sessions are easy to revoke but need a shared store. JWTs avoid per-request lookups and scale well across services but are hard to revoke before expiry. A common design is short-lived JWT access tokens plus revocable refresh tokens.

**Q: How do you securely store passwords?**
Hash with a slow, salted algorithm such as Argon2id or bcrypt, never with a fast hash or encryption, and add rate limiting and MFA.

**Q: How do you revoke a JWT?**
Keep access tokens short-lived and revoke the refresh token; for emergencies, maintain a small deny list of token IDs or rotate signing keys.

**Q: What is OAuth 2.0 and how is it different from OIDC?**
OAuth 2.0 delegates authorization (an app gets limited access to a user's resources without their password). OIDC adds an ID token on top to authenticate the user ("Sign in with Google").

**Q: Why PKCE?**
It binds the authorization code to a secret the client generated, so an intercepted code is useless. It is essential for mobile and single-page apps and recommended for all clients.

**Q: How do microservices trust each other?**
Verify user identity from a validated token passed through the gateway, authorize at each service, and authenticate services themselves with mTLS or signed service identities.

**Q: How would you prevent SQL injection and XSS?**
Use parameterized queries for SQL. For XSS, encode output for its context, use a Content Security Policy, and set `HttpOnly` cookies.

**Q: Where do you keep secrets?**
In a secrets manager or KMS, injected at runtime, rotated regularly, with per-service least-privilege access, and never in code or images.

**Q: What is zero trust?**
No request is trusted just because it comes from inside the network; every request is authenticated, authorized, and encrypted.

## Summary

- Authenticate to learn who the user is; authorize every action on the server, on every request.
- Hash passwords with Argon2id or bcrypt, add MFA or passkeys, and rate limit logins.
- Sessions revoke easily; JWTs scale easily. Combine short-lived access tokens with rotated refresh tokens.
- Use OAuth 2.0 (authorization code with PKCE) for delegated access and OIDC for sign-in; SSO centralizes identity.
- Use RBAC for most apps, ABAC or ReBAC for finer rules, always with least privilege and deny by default.
- Layer defenses: edge (WAF, DDoS), gateway (authN, rate limits), services (authZ), mTLS between services, encryption in transit and at rest, and a secrets manager.
- Know the OWASP Top 10 attacks and their standard defenses.

## Continue Learning

- [API Gateway](system-design/api-gateway) — where tokens are validated and limits enforced.
- [Rate Limiting](system-design/rate-limiting) — protecting login and APIs from abuse.
- [Observability and Deployment](system-design/observability-and-deployment) — audit logging, alerting, and incident response.
- [Microservices Architecture](system-design/microservices) — service-to-service trust.
- [System Design Cheat Sheet](system-design/cheat-sheet) — quick revision for interviews.
