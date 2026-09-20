# Auth API

**Base URL:** `api/v1`

**FastAPI Router:** `auth.py`

## Associated Tables

### `users`

| Column           | Data Type    | Key / Constraint                | Description                               |
| ---------------- | ------------ | ------------------------------- | ----------------------------------------- |
| `id`             | UUID         | PK                              | UUID v7 — timestamp sortable              |
| `email_id`       | VARCHAR(255) | NOT NULL, UNIQUE                | User email address                        |
| `password`       | VARCHAR(255) | NOT NULL                        | User password/hash                        |
| `country_code`   | VARCHAR(255) | —                               | One-to-one mapping with country code list |
| `mobile_no`      | BIGINT       | NOT NULL, UNIQUE                | User mobile number                        |
| `created_at`     | DATETIME     | —                               | Record creation timestamp                 |
| `updated_at`     | DATETIME     | —                               | Record last update timestamp              |
| `role`           | ENUM         | `Admin`, `Member`               | User role                                 |
| `status`         | ENUM         | `Active`, `Inactive`, `Deleted` | Account status                            |
| `last_login_dt`  | DATETIME     | Default: CURRENT_TIMESTAMP      | Last login time                           |
| `last_active_at` | DATETIME     | Default: CURRENT_TIMESTAMP      | Last active time                          |
| `deleted_at`     | DATETIME     | Default: NULL                   | Soft-delete timestamp                     |
| `longitude_dim`  | DOUBLE       | —                               | Auto-fetched longitude                    |
| `latitude_dim`   | DOUBLE       | —                               | Auto-fetched latitude                     |

## APIs

```text
POST  /auth/login
POST  /auth/logout
POST  /auth/signup
POST  /auth/refresh
GET   /auth/me
POST  /auth/forgot-password
POST  /auth/reset-password

--------------- PHASE 2 DEVELOPMENT ---------------
POST  /auth/send-otp
POST  /auth/verify-otp
```

---

## 1.1 :: POST `/auth/login`

**Request Body:**

```json
{
  "email_id": "aarav@buddyadda.com",
  "mobile_no": 9876543210,
  "password": "Password123!"
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Login is successful",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Login Failed",
  "error_message": "User doesn't exist"
}
```

**Note:**

- Login using Email or Phone No.
- JWT Access Token Structure: `user_id`, `role`, `expiry_time`, `JTI`, `subject`
- JWT Refresh Token Structure: `user_id`, `expiry_time`, `JTI`, `subject`
  - `sub` = subject / user identifier (chat | ai-companion | any micro-services)
  - `exp` = expiration time
  - `JTI` = unique JWT ID

## 1.2 :: POST `/auth/logout`

**Request Body:**

```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Logout Successful"
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Logout Failed",
  "error_message": "Refresh Token is not valid"
}
```

## 1.3 :: POST `/auth/signup`

**Request Body:**

```json
{
  "email_id": "pooja@example.com",
  "password": "SecurePassword123!",
  "confirm_password": "SecurePassword123!",
  "country_code": "+91",
  "mobile_no": 9100010001,
  "name_first": "Pooja",
  "name_last": "Hegde",
  "gender": "Female",
  "dob": "1996-05-20",
  "city_id": "Bengaluru",
  "pincode": 560034,
  "longitude_dim": 12.9716,
  "latitude_dim": 77.5946
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Account Creation Successful",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Account Creation Failed",
  "error_message": "Account already exists"
}
```

**Note:**

- `confirm_password` should not be stored in the `users` table.

## 1.4 :: POST `/auth/refresh`

**Request Body:**

```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Token Refresh Successful",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Token Refresh Failed",
  "error_message": "Refresh Token is not valid"
}
```

**Note:**

- App automatically logs out the user when refresh token expires.
- ID and password generate access tokens and refresh tokens.

## 1.5 :: GET `/auth/me`

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "User Details Retrieved Successfully",
  "data": {
    "user_id": "USR123456",
    "email_id": "pooja@example.com",
    "country_code": "+91",
    "mobile_no": 9100010001,
    "name_first": "Pooja",
    "name_last": "Hegde",
    "gender": "Female",
    "dob": "1996-05-20",
    "city_id": "Bengaluru",
    "pincode": 560034,
    "longitude_dim": 12.9716,
    "latitude_dim": 77.5946,
    "role": "USER"
  }
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Unauthorized",
  "error_message": "Access Token is invalid or expired"
}
```

## 1.6 :: POST `/auth/forgot-password`

**Note:**

* Validate whether the user exists in the DB.
* Backend calls `/auth/send-otp` to send OTP through mobile/email.
* `/auth/verify-otp` validates the user-entered OTP.
* Frontend prompts for new password and confirm password.
* Frontend calls `/auth/reset-password`.

**Request Body:**

```json
{
  "email_id": "pooja@example.com",
  "mobile_no": "9000080000",
  "country_code": "+91"
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Password Reset Link Sent Successfully"
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Password Reset Failed",
  "error_message": "User doesn't exist"
}
```

**Note:**

- Validate whether the user exists in the DB.
- Backend calls `/auth/send-otp` to send OTP through mobile/email.
- `/auth/verify-otp` validates the user-entered OTP.
- Frontend prompts for new password and confirm password.
- Frontend calls `/auth/reset-password`.
- `mobile_no` and `country_code` are optional — to be decided later.

## 1.7 :: POST `/auth/reset-password`

**Request Body:**

```json
{
  "password": "NewPassword123!",
  "confirm_password": "NewPassword123!",
  "email_id": "asd@gmail.com",
  "mobile_no": "9000080000",
  "country_code": "+91"
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Password Reset Successful"
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Password Reset Failed",
  "error_message": "Reset Token is not valid"
}
```

**Note:**

- After successful password reset, redirect the user to the **Login Page**.
- `mobile_no` and `country_code` are optional — to be decided later.
