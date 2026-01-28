# Backend Requirements: Google & Apple Sign-In

This document describes what the backend must implement so that Google and Apple sign-in work for both **signup** and **login** on the frontend.

---

## Overview

- **Google**: The frontend obtains a Google **ID token** (JWT) and sends it to your API. The backend verifies the token and returns your app’s access/refresh tokens (and user when new).
- **Apple**: The frontend obtains an Apple **identity token** (JWT) and optional user name/email (only on first authorization). The backend verifies the token and returns your app’s access/refresh tokens (and user when new).

Same endpoint is used for both **signup** (new user) and **login** (existing user): if the provider user does not exist, create the user and return tokens; if they exist, return tokens.

---

## 1. Google Sign-In

### Endpoint

- **Method**: `POST`
- **URL**: `{API_BASE}/auth/google/`
- **Headers**: `Content-Type: application/json`

### Request body

```json
{
  "id_token": "<Google ID token (JWT)>"
}
```

The frontend sends the **credential** (ID token) returned by Google Identity Services. It is a JWT.

### Backend responsibilities

1. **Verify the ID token**
   - Use Google’s public keys (JWKS) or a library (e.g. `google-auth-library` in Node, or verify JWT signature and claims).
   - Validate: `iss`, `aud` (your Google Client ID), `exp`, `sub`.

2. **Identify or create user**
   - Decode the JWT to get at least: `sub` (unique Google user ID), `email`, `email_verified`, `name` (or `given_name` / `family_name`).
   - Find user by provider = `google` and provider subject = `sub`.
   - If no user: create user (e.g. email, name from token; store `google` + `sub` as linked identity).
   - If user exists: use that user.

3. **Return your app tokens**
   - Issue your normal access and refresh tokens for that user.
   - Optionally return the user object in the same shape as your other auth endpoints.

### Success response (200)

```json
{
  "access": "<your JWT access token>",
  "refresh": "<your refresh token>",
  "user": {
    "id": 1,
    "user_id": 1,
    "username": "...",
    "email": "...",
    "first_name": "...",
    "last_name": "...",
    "role": "...",
    "user_uid": "...",
    "phone_number": "...",
    "media_data": {},
    "other_data": {},
    "user_data": {}
  }
}
```

- `user` is optional; if omitted, the frontend will call your existing “get profile” endpoint with the returned `access` token.

### Error response (4xx / 5xx)

```json
{
  "error": "Human-readable message",
  "detail": "Optional detail"
}
```

The frontend shows `error` or `detail` to the user.

---

## 2. Apple Sign-In

### Endpoint

- **Method**: `POST`
- **URL**: `{API_BASE}/auth/apple/`
- **Headers**: `Content-Type: application/json`

### Request body

```json
{
  "identity_token": "<Apple identity token (JWT)>",
  "user": {
    "name": {
      "firstName": "First",
      "lastName": "Last"
    },
    "email": "user@privaterelay.appleid.com"
  }
}
```

- **`identity_token`** (required): Apple’s JWT. Always sent.
- **`user`** (optional): Only sent on the **first** authorization. After that, Apple does not send name/email again, so the frontend may send only `identity_token`. Backend should rely on `identity_token` claims for `sub` and optionally email.

### Backend responsibilities

1. **Verify the identity token**
   - Use Apple’s public keys (JWKS) to verify the JWT.
   - Validate: `iss` (https://appleid.apple.com), `aud` (your Apple Services ID / Client ID), `exp`, `nonce` if you use it.

2. **Extract claims**
   - `sub`: unique, stable Apple user ID. Use this to find or create the user (e.g. provider = `apple`, provider subject = `sub`).
   - `email`: may be present in the token (or in the optional `user` object only on first sign-in).
   - For name: use the optional `user.name` from the request body when present; Apple does not put name in the JWT.

3. **Identify or create user**
   - Find user by provider = `apple` and provider subject = `sub`.
   - If no user: create user (email/name from token and/or `user`; store `apple` + `sub`).
   - If user exists: use that user.

4. **Return your app tokens**
   - Same as Google: return `access`, `refresh`, and optionally `user`.

### Success response (200)

Same shape as Google:

```json
{
  "access": "<your JWT access token>",
  "refresh": "<your refresh token>",
  "user": { ... }
}
```

### Error response

Same as Google: `{ "error": "...", "detail": "..." }`.

---

## 3. Token refresh and profile

- **Token refresh**: Use your existing `POST {API_BASE}/auth/token/refresh/` with `{ "refresh": "<refresh token>" }` and return `{ "access": "<new access token>" }`. No change needed for social users.
- **Profile**: Use your existing profile endpoint (e.g. `GET /user/profile/` or similar) with the `Authorization: Bearer <access>` token. No change needed; social and password users share the same session shape.

---

## 4. Summary table

| Provider | Endpoint             | Request body                                                | Backend action                                                                     |
| -------- | -------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Google   | `POST /auth/google/` | `{ "id_token": "<JWT>" }`                                   | Verify Google JWT → find/create user → return access/refresh (and optionally user) |
| Apple    | `POST /auth/apple/`  | `{ "identity_token": "<JWT>", "user?": { name?, email? } }` | Verify Apple JWT → find/create user → return access/refresh (and optionally user)  |

---

## 5. Frontend flow recap

- **Google**: User clicks “Continue with Google” → Google popup → frontend gets ID token → `POST /auth/google/` with `id_token` → store access/refresh and user → redirect to account.
- **Apple**: User clicks “Continue with Apple” → redirect to `/{locale}/auth/apple` → redirect to Apple → user signs in → Apple POSTs to `https://yourdomain.com/auth/apple/callback` → backend callback (Next.js) redirects browser to `/{locale}/auth/callback?apple_id_token=...&apple_user=...` → frontend calls `POST /auth/apple/` with `identity_token` and optional `user` → store access/refresh and user → redirect to account.

Once the backend implements `POST /auth/google/` and `POST /auth/apple/` as above, Google and Apple sign-in and signup will work end-to-end.

---

## 6. What to request from the backend team

Ask the backend to implement the following so Google and Apple login work:

| Item | Description |
|------|-------------|
| **Google** | `POST {API_BASE}/auth/google/` with body `{ "id_token": "<JWT>" }`. Verify the Google ID token, find or create user by `sub`, return `{ "access", "refresh", "user"?: {...} }`. |
| **Apple** | `POST {API_BASE}/auth/apple/` with body `{ "identity_token": "<JWT>", "user"?: { "name"?: { "firstName", "lastName" }, "email"?: string } }`. Verify the Apple identity token, find or create user by `sub`, return `{ "access", "refresh", "user"?: {...} }`. |
| **Token refresh** | Existing `POST {API_BASE}/auth/token/refresh/` with `{ "refresh": "<token>" }` → `{ "access": "<new access>" }`. No change; social and password users use the same refresh. |
| **Profile** | Existing profile endpoint (e.g. `GET /user/profile/` or similar) with `Authorization: Bearer <access>`. No change; social and password users share the same user shape. |

**Environment (frontend):**

- `NEXT_PUBLIC_GOOGLE_CLIENT_ID` – Google OAuth client ID (Web application).
- `NEXT_PUBLIC_APPLE_CLIENT_ID` – Apple Services ID. In Apple Developer, configure the **Return URL** for Sign in with Apple to: `https://<your-domain>/api/auth/apple/callback`.
- `NEXT_PUBLIC_APP_ORIGIN` (optional) – Full origin (e.g. `https://www.wasal.com`) for server-side redirects; otherwise the app uses the request origin.
