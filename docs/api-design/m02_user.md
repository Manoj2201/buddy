# User API

**Base URL:** `api/v1`

**FastAPI Router:** `users.py`

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

### `user_profile`

| Column              | Data Type     | Key / Constraint                | Description                                |
| ------------------- | ------------- | ------------------------------- | ------------------------------------------ |
| `user_id`           | UUID          | PK, FK → `users.id`             | User profile identifier; cascade on delete |
| `name_first`        | VARCHAR(255)  | —                               | First name                                 |
| `name_middle`       | VARCHAR(255)  | —                               | Middle name                                |
| `name_last`         | VARCHAR(255)  | —                               | Last name                                  |
| `name_full`         | VARCHAR(255)  | —                               | Combined first, middle and last name       |
| `dob`               | DATE          | —                               | Date of birth; YYYYMMDD                    |
| `gender`            | ENUM          | `Male`, `Female`, `Undisclosed` | Gender                                     |
| `bio`               | VARCHAR(300)  | —                               | User biography                             |
| `profession`        | VARCHAR(255)  | —                               | User profession                            |
| `city_id`           | VARCHAR(255)  | —                               | City identifier                            |
| `pincode`           | BIGINT        | —                               | Postal/PIN code                            |
| `profile_photo_uri` | VARCHAR(2048) | —                               | Profile image URI; stored in Cloudflare R2 |
| `created_at`        | DATETIME      | —                               | Profile creation timestamp                 |
| `updated_at`        | DATETIME      | —                               | Profile last update timestamp              |

## APIs

Note :  
* JWT Structure: `user-id`, `role`, `expiry-time`, `JTI`, `subject`
- To retrieve the user-id from the JWT

```
POST    /user/profile                :: Creating the User Profile
GET     /user/profile                :: Get User Profile 
PATCH   /user/profile                :: Edit the User Profile
PUT     /user/profile/deactivate     :: deactivate = {TRUE | FLASE}
DELETE  /user                        :: Hard Delete
```

**Header:**

- Access Token (JWT Structure: `user-id`, `role`, `expiry-time`, `JTI`, `subject`)
- Request-Id

**Note:**

- JWT Structure: `user_id`, `role`, `expiry_time`, `JTI`, `subject`
- To retrieve the `user_id` from the JWT

---

## 2.1 :: POST `/user/profile`

**Request Body:**

```json
{
  "country_code": "+91",
  "mobile_no": 9100010001,
  "name_first": "Pooja",
  "name_last": "Hegde",
  "gender": "Female",
  "dob": "1996-05-20",
  "city_id": "Bengaluru",
  "pincode": 560034,
  "bio": "Updated bio: Weekend cyclist & photographer.",
  "profession": "Senior Product Designer",
  "profile_photo_uri": "https://r2.buddyadda.com/avatars/new.png",
  "interests": ["cycling", "photography", "trekking"],
  "languages": ["English", "Hindi"]
}
```

```python
mandatory_user_profile_fields = [
    "name_first",
    "name_last",
    "dob",
    "gender",
    "city_id",
    "pincode",
    "profession",
    "profile_photo"
]
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Profile Created Successfully",
  "data": {
    "user_id": "#####",
    "bio": "Updated bio: Weekend cyclist & photographer.",
    "profession": "Senior Product Designer",
    "profile_photo_uri": "https://r2.buddyadda.com/avatars/new.png",
    "interests": ["cycling", "photography", "trekking"],
    "languages": ["English", "Hindi"]
  }
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Profile Creation Failed",
  "error_message": "User doesn't exist"
}
```

**Note:**

- Mandatory user profile fields: `name_first`, `name_last`, `dob`, `gender`, `city_id`, `pincode`, `profession`, `profile_photo`

## 2.2 :: GET `/user/profile`

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "User Details Retrieved Successfully",
  "data": {
    "id": "USR123456",
    "email_id": "aarav@buddyadda.com",
    "mobile_no": 9876543210,
    "status": "Active",
    "profile": {
      "name_full": "Aarav Sharma",
      "bio": "Software engineer by weekday, mountain climber by weekend.",
      "profession": "Software Engineer",
      "city_id": "Bengaluru",
      "interests": ["trekking", "swimming", "photography"],
      "languages": ["English", "Hindi", "Kannada"]
    }
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

## 2.3 :: PATCH `/user/profile`

**Request Body:**

```json
{
  "country_code": "+91",
  "mobile_no": 9100010001,
  "name_first": "Pooja",
  "name_last": "Hegde",
  "gender": "Female",
  "dob": "1996-05-20",
  "city_id": "Bengaluru",
  "pincode": 560034,
  "bio": "Updated bio: Weekend cyclist & photographer.",
  "profession": "Senior Product Designer",
  "profile_photo_uri": "https://r2.buddyadda.com/avatars/new.png",
  "interests": ["cycling", "photography", "trekking"],
  "languages": ["English", "Hindi"]
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Profile Update Successful",
  "data": {
    "user_id": "#####",
    "bio": "Updated bio: Weekend cyclist & photographer.",
    "profession": "Senior Product Designer",
    "profile_photo_uri": "https://r2.buddyadda.com/avatars/new.png",
    "interests": ["cycling", "photography", "trekking"],
    "languages": ["English", "Hindi"]
  }
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Profile Update Failed",
  "error_message": "User doesn't exist"
}
```

---



## 2.4 :: PUT `/user/profile/deactivate`

**Request Body:**

```json
{
  "deactivate": true
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "User Deactivated Successfully"
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Unable to Deactivate User",
  "error_message": "Server Internal Error"
}
```

**Note:**

- `deactivate` accepts boolean values: `true` or `false`
## 2.5 :: DELETE `/user`

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "User Profile Deleted"
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Unable to Delete User Profile",
  "error_message": "Server Internal Error"
}
```
