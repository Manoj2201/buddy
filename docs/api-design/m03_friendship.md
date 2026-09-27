# Friendship API

**Base URL:** `/api/v1`

## Associated Tables

### A. `friend_requests`

| Column        | Data Type | Key / Constraint                  | Description               |
| ------------- | --------- | --------------------------------- | ------------------------- |
| `id`          | VARCHAR   | PK                                | UUID                      |
| `sender_id`   | UUID      | FK → `users.id`                   | Friendship requester      |
| `receiver_id` | UUID      | FK → `users.id`                   | Friendship recipient      |
| `status`      | ENUM      | `pending`, `accepted`, `declined` | Friendship Request status |
| `sent_at`     | TIMESTAMP | —                                 | Request sent time         |
| `accepted_at` | TIMESTAMP | —                                 | Request accepted time     |

### Lifecycle / Transition Flow

```text
+-------------------+        Accept Request        +-------------------+
|  friend_requests  | ---------------------------> |    friendships    |
+-------------------+                              +-------------------+
| sender_id (FK)    |    Deletes request row       | user_id_1 (FK)    |
| receiver_id (FK)  |    and inserts active pair   | user_id_2 (FK)    |
| status            |                              | created_at        |
| created_at        |                              +-------------------+
+-------------------+
```

### B. `friendships`

| Column              | Data Type | Key / Constraint                         | Description                 |
| ------------------- | --------- | ---------------------------------------- | --------------------------- |
| `id`                | VARCHAR   | PK                                       | UUID                        |
| `from_user`         | UUID      | FK → `users.id`                          | User 1                      |
| `to_user`           | UUID      | FK → `users.id`                          | User 2                      |
| `status`            | ENUM      | `muted`, `blocked`, `friend`, `un-friend` | Friendship status           |
| `friend_request_id` | VARCHAR   | FK → `friend_requests.id`                | Reference to friend request |

---

## APIs

**Endpoints:** `friend-request`, `friendships`

```text
POST   /friend-request
:: Send friendship request

PUT    /friend-request/{id}
:: Status : Accept/Decline/Cancel friendship request

GET    /friend-request?status={...}
:: Returns friendship request status (query param)

GET    /friend-request/{id}
:: Returns the status of the respective friendship request

PUT    /friendships/{id}
:: Status : Block/Unblock/un-friend/mute/un-mute

GET    /friendships
:: Returns all the friends
```

**Header:**

- **Access Token:** JWT Structure: `user-id`, `role`, `expiry-time`, `JTI`, `subject`
- **Request-Id**

**Note:**

- Current logged-in `user-id` should be taken from the JWT.
- Do not pass `sender-user-id` from frontend. Backend should derive it from the token.
- `friend-request` is used before friendship is created.
- `friendships` is used after the request is accepted.

---

## 3.1 :: POST `/friend-request`

**Purpose:**

- Send a friendship request to another user.

**Request Body:**

```json
{
  "receiver-user-id": "USR987654",
  "message": "Hi, let's connect."
}
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Friend Request Sent Successfully",
  "data": {
    "friend-request-id": "FRQ123456",
    "sender-user-id": "USR123456",
    "receiver-user-id": "USR987654",
    "status": "Pending"
  }
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Friend Request Failed",
  "error-message": "Friend request already exists"
}
```

**Note:**

- One user cannot send a friend request to themselves.
- Duplicate pending requests should not be allowed.
- If both users are already friends, return a clear error message.

---

## 3.2 :: PUT `/friend-request/{id}`

**Purpose:**

- Update the friendship request status.
- Used for `Accept`, `Decline`, or `Cancel`.

**Path Parameter:**

| Parameter | Type | Description                   |
| --------- | ---- | ----------------------------- |
| `id`      | UUID | Friendship request identifier |

**Request Body:**

```json
{
  "status": "Accepted"
}
```

**Allowed Status Values:**

```text
Accepted
Declined
Cancelled
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Friend Request Updated Successfully",
  "data": {
    "friend-request-id": "FRQ123456",
    "status": "Accepted",
    "friendship-id": "FRD123456"
  }
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Friend Request Update Failed",
  "error-message": "Invalid status update"
}
```

**Note:**

- Receiver can `Accept` or `Decline` the request.
- Sender can `Cancel` the request.
- When status becomes `Accepted`, create a record in `friendships`.

---

## 3.3 :: GET `/friend-request?status={status}`

**Purpose:**

- Get friendship requests filtered by status.

**Query Parameter:**

| Parameter | Type   | Required | Description                                       |
| --------- | ------ | -------- | ------------------------------------------------- |
| `status`  | STRING | Optional | `Pending`, `Accepted`, `Declined`, or `Cancelled` |

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Friend Requests Retrieved Successfully",
  "data": [
    {
      "friend-request-id": "FRQ123456",
      "sender-user-id": "USR123456",
      "receiver-user-id": "USR987654",
      "status": "Pending",
      "created-at": "2026-09-27T10:30:00Z"
    }
  ]
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Unauthorized",
  "error-message": "Access Token is invalid or expired"
}
```

**Note:**

- If `status` is not passed, backend can return all request statuses for the logged-in user.
- This API should return requests where the logged-in user is either sender or receiver.

---

## 3.4 :: GET `/friend-request/{id}`

**Purpose:**

- Get the status/details of one friendship request.

**Path Parameter:**

| Parameter | Type | Description                   |
| --------- | ---- | ----------------------------- |
| `id`      | UUID | Friendship request identifier |

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Friend Request Status Retrieved Successfully",
  "data": {
    "friend-request-id": "FRQ123456",
    "sender-user-id": "USR123456",
    "receiver-user-id": "USR987654",
    "status": "Pending",
    "created-at": "2026-09-27T10:30:00Z",
    "updated-at": "2026-09-27T10:30:00Z"
  }
}
```

**Status: 404**

```json
{
  "success": false,
  "message": "Friend Request Not Found",
  "error-message": "Invalid friend request id"
}
```

---

## 3.5 :: PUT `/friendships/{id}`

**Purpose:**

- Update an existing friendship.
- Used for `Block`, `Unblock`, `Unfriend`, `Mute`, or `Unmute`.

**Path Parameter:**

| Parameter | Type | Description           |
| --------- | ---- | --------------------- |
| `id`      | UUID | Friendship identifier |

**Request Body:**

```json
{
  "action": "Mute"
}
```

**Allowed Action Values:**

```text
Block
Unblock
Unfriend
Mute
Unmute
```

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Friendship Updated Successfully",
  "data": {
    "friendship-id": "FRD123456",
    "status": "Active",
    "is-muted": true
  }
}
```

**Status: 400**

```json
{
  "success": false,
  "message": "Friendship Update Failed",
  "error-message": "Invalid friendship action"
}
```

**Note:**

- `Block` changes friendship status to `Blocked`.
- `Unblock` changes friendship status back to `Active`.
- `Unfriend` changes friendship status to `Unfriended`.
- `Mute` and `Unmute` only update the `is-muted` flag.

---

## 3.6 :: GET `/friendships`

**Purpose:**

- Get all active friends of the logged-in user.

**Request Body:** `None`

**Response Body:**

**Status: 200**

```json
{
  "success": true,
  "message": "Friends Retrieved Successfully",
  "data": [
    {
      "friendship-id": "FRD123456",
      "friend-user-id": "USR987654",
      "name-full": "Aarav Sharma",
      "profile-photo-uri": "https://r2.buddyadda.com/avatars/aarav.png",
      "status": "Active",
      "is-muted": false
    }
  ]
}
```

**Status: 401**

```json
{
  "success": false,
  "message": "Unauthorized",
  "error-message": "Access Token is invalid or expired"
}
```

**Note:**

- By default, return only `Active` friends.
- Blocked or unfriended users can be returned only if a separate filter is added later.
