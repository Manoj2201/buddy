## Access Token and Refresh Token — Refined Notes

```text
AUTHENTICATION FLOW
══════════════════════════════════════════════════════════════

1. SIGNUP
─────────

┌──────────────────────────┐
│ Enter account details    │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ POST /api/v1/auth/signup │
└────────────┬─────────────┘
             ▼
      ◇ Account exists? ◇
          ╱         ╲
        Yes          No
         │            │
         ▼            ▼
┌──────────────┐  ┌───────────────────┐
│ 409 Conflict │  │ Hash password and │
└──────────────┘  │ create user       │
                  └─────────┬─────────┘
                            ▼
                  ┌───────────────────┐
                  │ Return token pair │
                  │                   │
                  │ • Access token    │
                  │ • Refresh token   │
                  └───────────────────┘


2. LOGIN
────────

┌─────────────────────────┐
│ Email/phone + password  │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ POST /api/v1/auth/login │
└────────────┬────────────┘
             ▼
   ◇ Credentials valid? ◇
        ╱           ╲
      No             Yes
       │              │
       ▼              ▼
┌────────────────┐  ┌─────────────────────┐
│ 401            │  │ Return token pair   │
│ Login failed   │  │                     │
└────────────────┘  │ • Access token      │
                    │ • Refresh token     │
                    └─────────────────────┘


3. PROTECTED API REQUEST
────────────────────────

┌────────────────────────────────────────┐
│ Call protected API                     │
│                                        │
│ Authorization: Bearer <access_token>   │
└───────────────────┬────────────────────┘
                    ▼
┌────────────────────────────────────────┐
│ Server validates access token          │
│                                        │
│ • Signature                            │
│ • Expiration                           │
│ • User ID                              │
│ • Token type                           │
│ • Role and audience                    │
└───────────────────┬────────────────────┘
                    ▼
            ◇ Token valid? ◇
               ╱         ╲
             Yes          No
              │            │
              ▼            ▼
┌─────────────────────┐  ┌─────────────────┐
│ Process API request │  │ Return 401      │
└─────────────────────┘  │ Refresh token   │
                         └─────────────────┘


4. TOKEN REFRESH
────────────────

┌──────────────────────────────┐
│ Access token has expired     │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ POST /api/v1/auth/refresh    │
│ Send refresh token           │
└──────────────┬───────────────┘
               ▼
      ◇ Refresh token valid? ◇
           ╱             ╲
         No               Yes
          │                │
          ▼                ▼
┌──────────────────┐  ┌──────────────────────┐
│ Return 401       │  │ Revoke old refresh  │
│ Clear session    │  │ token               │
│ Go to login      │  └──────────┬───────────┘
└──────────────────┘             ▼
                      ┌──────────────────────┐
                      │ Issue new access and │
                      │ refresh tokens       │
                      └──────────┬───────────┘
                                 ▼
                      ┌──────────────────────┐
                      │ Retry original API   │
                      │ request              │
                      └──────────────────────┘


5. LOGOUT
─────────

┌──────────────────────────────┐
│ POST /api/v1/auth/logout     │
│ Send refresh token           │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ Revoke refresh token/session │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ Delete tokens from client    │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ Redirect to login            │
└──────────────────────────────┘


6. PASSWORD RESET
─────────────────

┌────────────────────────────────────┐
│ POST /auth/forgot-password         │
│ Enter email or phone               │
└─────────────────┬──────────────────┘
                  ▼
┌────────────────────────────────────┐
│ Send OTP                         │
└─────────────────┬──────────────────┘
                  ▼
┌────────────────────────────────────┐
│ POST /auth/verify-otp              │
└─────────────────┬──────────────────┘
                  ▼
             ◇ OTP valid? ◇
                ╱       ╲
              No         Yes
               │          │
               ▼          ▼
┌──────────────────┐  ┌────────────────────┐
│ Reject request   │  │ Issue one-time     │
└──────────────────┘  │ reset token        │
                      └──────────┬─────────┘
                                 ▼
                      ┌────────────────────────┐
                      │ POST /reset-password  │
                      │ Reset token + password│
                      └───────────┬────────────┘
                                  ▼
                      ┌────────────────────────┐
                      │ Update password hash   │
                      │ Revoke all sessions    │
                      └───────────┬────────────┘
                                  ▼
                      ┌────────────────────────┐
                      │ Redirect to login      │
                      └────────────────────────┘
```

## Understanding

### Access token

The access token is used to access protected APIs.

```http
GET /api/v1/auth/me
Authorization: Bearer <access_token>
```

* Short lifetime: usually **10–30 minutes**
* Sent with protected API requests
* Contains the user identity and authorization information
* Cannot normally be used after expiry

### Refresh token

The refresh token is used to obtain a new token pair.

* Longer lifetime: usually **7–30 days**
* Sent only to `/auth/refresh` and `/auth/logout`
* Should not be sent with normal API requests
* Must be revocable
* Should be rotated after every refresh

## Token claims

```json
{
  "sub": "user-uuid",
  "type": "access",
  "role": "Member",
  "aud": ["chat", "ai-companion"],
  "jti": "unique-token-id",
  "iat": 1789891200,
  "exp": 1789892100
}
```

| Claim  | Meaning                              |
| ------ | ------------------------------------ |
| `sub`  | User ID                              |
| `type` | `access` or `refresh`                |
| `role` | `Admin` or `Member`                  |
| `aud`  | Services allowed to accept the token |
| `jti`  | Unique token identifier              |
| `iat`  | Token creation time                  |
| `exp`  | Token expiration time                |

`sub` should contain the **user ID**. Values such as `chat` and `ai-companion` should be stored in `aud`.

## Complete lifecycle

```text
Login
  │
  ▼
Receive access token + refresh token
  │
  ▼
Call API using access token
  │
  ├── Access token valid ──► Process request
  │
  └── Access token expired
           │
           ▼
      Use refresh token
           │
           ├── Valid ──► New tokens ──► Retry API
           │
           └── Invalid/expired ──► Login again
```

## Important security rules

1. Hash passwords using **Argon2id** or **bcrypt**.
2. Keep access tokens short-lived.
3. Store refresh tokens in `HttpOnly`, `Secure` cookies.
4. Rotate refresh tokens after every refresh.
5. Revoke all sessions after a password reset.
6. Never place passwords or sensitive information inside JWTs.
7. Return a generic error for invalid login credentials.
8. Rate-limit login, OTP and password-reset requests.
