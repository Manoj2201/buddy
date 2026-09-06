# Authentication API

**Base URL:** `api/v1`
**FastAPI Router:** `auth.py`

## Associted Tables
- `users`

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

```text id="t4j1qs"
POST  /auth/login
POST  /auth/logout
POST  /auth/signup
POST  /auth/refresh           :: Generate new JWT token
GET   /auth/me                :: Fetch respective user details
POST  /auth/forgot-password
POST  /auth/reset-password

--------------- PHASE 2 DEVELOPMENT ---------------
POST  /auth/send-otp
POST  /auth/verify-otp
```

# Login Page

## 1.1 :: POST `/auth/login`

**Request Body:**

```json id="2mgx5q"
{
  "emailId": "aarav@buddyadda.com",
  "mobileNo": 9876543210,
  "password": "Password123!"
}
```

**Response Body:**

**Status: 200**

```json id="6prv5h"
{
  "success": true,
  "message": "Login is successful",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Status: 401**

```json id="0qpj0k"
{
  "success": false,
  "message": "Login Failed",
  "error_message": "User doesn't exist"
}
```

**Note:**

* Login using Email or Phone No.
* JWT Structure: `user-id`, `role`, `expiry-time`, `JTI`, `subject`

## 1.2 :: POST `/auth/logout`

**Request Body:**

```json id="7ofn84"
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response Body:**

**Status: 200**

```json id="f1r2gj"
{
  "success": true,
  "message": "Logout Successful"
}
```

**Status: 401**

```json id="6gvn2b"
{
  "success": false,
  "message": "Logout Failed",
  "error_message": "Refresh Token is not valid"
}
```

## 1.3 :: POST `/auth/signup`

**Request Body:**

```json id="tl4h6i"
{
  "emailId": "pooja@example.com",
  "password": "SecurePassword123!",
  "confirm-password": "SecurePassword123!",
  "countryCode": "+91",
  "mobileNo": 9100010001,
  "nameFirst": "Pooja",
  "nameLast": "Hegde",
  "gender": "Female",
  "dob": "1996-05-20",
  "cityId": "Bengaluru",
  "pincode": 560034,
  "longitudeDim": 12.9716,
  "latitudeDim": 77.5946
}
```

**Note:** `confirm-password` should not be stored in the `users` table.

**Response Body:**

**Status: 200**

```json id="9l91a6"
{
  "success": true,
  "message": "Account Creation Successful",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Status: 401**

```json id="i91ev6"
{
  "success": false,
  "message": "Account Creation Failed",
  "error_message": "Account already exists"
}
```

## 1.4 :: POST `/auth/refresh`

**Request Body:**

```json id="8p7e88"
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response Body:**

**Status: 200**

```json id="qrl4s4"
{
  "success": true,
  "message": "Token Refresh Successful",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Status: 401**

```json id="l3x5qr"
{
  "success": false,
  "message": "Token Refresh Failed",
  "error_message": "Refresh Token is not valid"
}
```

## 1.5 :: GET `/auth/me`

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json id="nsfepk"
{
  "success": true,
  "message": "User Details Retrieved Successfully",
  "data": {
    "userId": "USR123456",
    "emailId": "pooja@example.com",
    "countryCode": "+91",
    "mobileNo": 9100010001,
    "nameFirst": "Pooja",
    "nameLast": "Hegde",
    "gender": "Female",
    "dob": "1996-05-20",
    "cityId": "Bengaluru",
    "pincode": 560034,
    "longitudeDim": 12.9716,
    "latitudeDim": 77.5946,
    "role": "USER"
  }
}
```

**Status: 401**

```json id="5jq4o7"
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

```json id="x4vkya"
{
  "emailId": "pooja@example.com",
  "mobileNo": "9000080000",
  "countryCode": "+91"
}
```

`mobileNo` and `countryCode` are optional — to be decided later.

**Response Body:**

**Status: 200**

```json id="3kpsat"
{
  "success": true,
  "message": "Password Reset Link Sent Successfully"
}
```

**Status: 401**

```json id="avb8cr"
{
  "success": false,
  "message": "Password Reset Failed",
  "error_message": "User doesn't exist"
}
```

## 1.7 :: POST `/auth/reset-password`

**Note:** After successful password reset, redirect the user to the **Login Page**.

**Request Body:**

```json id="n0p9vj"
{
  "password": "NewPassword123!",
  "confirm-password": "NewPassword123!",
  "email_id": "asd@gmail.com",
  "mobileNo": "9000080000",
  "countryCode": "+91"
}
```

`mobileNo` and `countryCode` are optional — to be decided later.

**Response Body:**

**Status: 200**

```json id="j6dg6x"
{
  "success": true,
  "message": "Password Reset Successful"
}
```

**Status: 401**

```json id="34j8va"
{
  "success": false,
  "message": "Password Reset Failed",
  "error_message": "Reset Token is not valid"
}
```
