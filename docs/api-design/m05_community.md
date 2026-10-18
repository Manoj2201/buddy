Notes:
-------
## Interests (provided by the system)
Social activities
Hobbies/Passions
Sports/Fitness
Travel/Outdoor
Career/Business
Technology
Community/Environment
Languages
Games
Dancing
Support/Coaching
Music
Health/Well-being
Art/Culture
Science/Education
Pets/Animals
Religion/Spirituality
Writing
Parents/Family
Movements/Politics
Reading
(Select atleast 1 and maximum 5)

Buddy Adda Community and Sub-Community architecture:
There are 2 concepts: 
1) interest-community (created by default for every interest)
2) sub-community 
(2-level hierarchy)

1) interest-community has ADMIN 
2) sub-community has MODERATOR. sub community is created by logged-in user subject to interest-community ADMIN approval.

1) interest-community cannot be generic like Bengaluru. It has to be interest specific. Like Outdoor/Travel Community.
2) sub-community has to be a focussed group. Like "Kodachadri Trekking"

1) interest-community cannot have a chat group. It prevents chatting with a very wide audience across sub-communities.
2) sub-community has a Chat group.

1) interest-community has a Broadcast-board (Read only).
2) sub-community has a Buletin-board (Interactive).

1) interest-community has Events
2) sub-community has Meetups 


# Community API

**Base URL:** `/api/v1`

## Associated Tables

### A. 
 
 ### 1. `communities`

| Column             | Data Type | Key / Constraint                   | Description                                                |
| ------------------ | --------- | ---------------------------------- | ---------------------------------------------------------- |
| `id`               | UUID      | PK                                 | Unique group identifier                                    |
| `name`             | VARCHAR   | —                                  | Name of the group                                          |
| `parent_id`        | UUID      | —                                  | Parent group identifier, if applicable                     |
| `is_admin_managed` | BOOLEAN   | —                                  | Indicates whether the group is managed by an administrator |
| `visibility`       | ENUM      | `public`, `private`, `invite_only` | Group visibility setting                                   |
| `description`      | VARCHAR   | —                                  | Description of the group                                   |
| `status`           | VARCHAR   | —                                  | Current status of the group                                |
| `created_by`       | UUID      | FK → `users.id`, ON DELETE CASCADE | User who created the group                                 |

| Column        | Purpose                                                                            |
| ------------- | ---------------------------------------------------------------------------------- |
| `interest_id` | Interest associated with the community                                             |
| `created_at`  | Creation timestamp                                                                 |

### 2. `group_interests`

| Column        | Data Type | Key / Constraint                         | Description                        |
| ------------- | --------- | ---------------------------------------- | ---------------------------------- |
| `group_id`    | UUID      | FK → `communities.id`, ON DELETE CASCADE      | Group associated with the interest |
| `interest_id` | VARCHAR   | FK → `interests.name`, ON DELETE CASCADE | Interest associated with the group |

Design note: Consider a composite primary key (`group_id`, `interest_id`) to prevent duplicate interest associations within a group.

### 3. `group_members`

| Column         | Data Type | Key / Constraint                    | Description                                     |
| -------------- | --------- | ----------------------------------- | ----------------------------------------------- |
| `user_id`      | UUID      | FK → `users.id`, ON DELETE CASCADE  | User who belongs to the group                   |
| `group_id`     | UUID      | FK → `communities.id`, ON DELETE CASCADE | Group the user belongs to                       |
| `is_moderator` | BOOLEAN   | —                                   | Indicates whether the user is a group moderator |
| `joined_at`    | TIMESTAMP | —                                   | Date and time the user joined the group         |

A group with "n" members can have "n" moderators.

## APIs

| No. | Method | API Endpoint                              | Purpose                                                  |
| --- | ------ | ----------------------------------------- | -------------------------------------------------------- |
| 1   | POST   | `/api/v1/communities`                          | Create a new community or sub-community                |
| 2   | GET    | `/api/v1/communities`                          | Discover active communities                                   |
| 3   | GET    | `/api/v1/communities/{id}`                     | Get specific community details                           |
| 4   | GET    | `/api/v1/communities/{id}/subcommunities`      | Get sub-communities for a specific community                  |
| 5   | PUT    | `/api/v1/communities/{id}`                     | Update community settings (Admin only)           |
| 6   | DELETE | `/api/v1/communities/{id}`                     | Delete a community (Admin only)                  |
| 7   | POST   | `/api/v1/communities/{id}/join`                | Join a community                                         |
| 8   | POST   | `/api/v1/communities/{id}/leave`               | Leave a community                                        |
| 9   | DELETE | `/api/v1/communities/{id}/members/{userId}`    | Moderator: Kick a user from the community                     |
| 10  | POST   | `/api/v1/communities/{id}/moderators/{userId}` | Moderator: Promote user to moderator (Admin only) |
| 11  | GET    | `/api/v1/admin/communities/pending`            | Admin: List user-created sub-communities awaiting approval        |
| 12  | POST   | `/api/v1/admin/communities/{id}/approve`       | Admin: Approve a pending sub-community                           |
| 13  | POST   | `/api/v1/admin/communities/{id}/reject`        | Admin: Reject a pending sub-community                            |

# BuddyAdda — communities API Specification

The following specifications cover the request body, response body, and HTTP status codes for all 13 communities APIs. They follow the two-level hierarchy: Interest Community → Sub-community.

## Common response conventions

- Success response: `success`, `data`, and `message`.
- Error response: `success: false` with an `error` object containing `code` and `message`.
- Use UUIDs for group and user identifiers.
- A `parent_id` of `null` identifies a default Interest Community; a non-null `parent_id` identifies a sub-community.
- All protected operations require authentication and appropriate role-based authorization.

Example error response:

```
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "You are not authorized to perform this action."
  }
}
```

## 1. Create a community or sub-community

`POST /api/v1/communities`

Purpose: Create a user-created sub-community, or create a community when authorized by platform administration. Default Interest Communities should normally be created automatically when an interest is created.

Request body — sub-community:

```
{
  "name": "Weekend Trekkers",
  "interest_id": "interest-uuid",
  "parent_id": "parent-group-uuid",
  "description": "A group for people who enjoy weekend treks.",
  "visibility": "public"
}
```

Response body — pending approval:

```
{
  "success": true,
  "data": {
    "id": "group-uuid",
    "name": "Weekend Trekkers",
    "interest_id": "interest-uuid",
    "parent_id": "parent-group-uuid",
    "created_by": "user-uuid",
    "status": "pending",
    "visibility": "public",
    "created_at": "2026-10-11T10:00:00Z"
  },
  "message": "Group created and submitted for approval."
}
```

Status codes: `201 Created` — created; `400 Bad Request` — invalid fields or hierarchy; `401 Unauthorized` — not authenticated; `403 Forbidden` — insufficient permissions; `409 Conflict` — duplicate group; `422 Unprocessable Entity` — validation failure.

## 2. Discover active communities

`GET /api/v1/communities`

Purpose: Return active communities that the requesting user is permitted to discover.

Request body: None.

Optional query parameters: `interest_id`, `parent_id`, `visibility`, `page`, `page_size`, `sort_by`, `sort_order`.

Example: `/api/v1/communities?interest_id=interest-uuid&page=1&page_size=10`

Response body:

```
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "group-uuid",
        "name": "Weekend Trekkers",
        "interest_id": "interest-uuid",
        "parent_id": "parent-group-uuid",
        "created_by": "user-uuid",
        "status": "active",
        "visibility": "public",
        "created_at": "2026-10-11T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 10,
      "total": 1,
      "total_pages": 1
    }
  },
  "message": "communities retrieved successfully."
}
```

Status codes: `200 OK` — success; `400 Bad Request` — invalid query parameters; `401 Unauthorized` — authentication required if the endpoint is protected.

## 3. Get specific community details

`GET /api/v1/communities/{id}`

Purpose: Retrieve details of a specific community or sub-community.

Request body: None.

Path parameter: `id` — UUID of the group.

Response body:

```
{
  "success": true,
  "data": {
    "id": "group-uuid",
    "name": "Weekend Trekkers",
    "interest_id": "interest-uuid",
    "parent_id": "parent-group-uuid",
    "created_by": "user-uuid",
    "description": "A group for people who enjoy weekend treks.",
    "status": "active",
    "visibility": "public",
    "created_at": "2026-10-11T10:00:00Z",
    "member_count": 25
  },
  "message": "Group details retrieved successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`.

## 4. Get sub-communities for a community

`GET /api/v1/communities/{id}/subcommunities`

Purpose: Retrieve the sub-communities belonging to a specific parent community.

Request body: None.

Path parameter: `id` — UUID of the parent community.

Response body:

```
{
  "success": true,
  "data": [
    {
      "id": "subgroup-uuid-1",
      "name": "Weekend Trekkers",
      "interest_id": "interest-uuid",
      "parent_id": "parent-group-uuid",
      "status": "active",
      "visibility": "public"
    },
    {
      "id": "subgroup-uuid-2",
      "name": "Beginner Trekkers",
      "interest_id": "interest-uuid",
      "parent_id": "parent-group-uuid",
      "status": "active",
      "visibility": "private"
    }
  ],
  "message": "Sub-communities retrieved successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`.

Private sub-communities should only be returned when the requesting user is authorized to discover them.

## 5. Update community settings

`PUT /api/v1/communities/{id}`

Purpose: Update community details and settings. Restricted to authorized group creators/admins.

Request body:

```
{
  "name": "Bengaluru Weekend Trekkers",
  "description": "A community for weekend trekking enthusiasts.",
  "visibility": "private"
}
```

Response body:

```
{
  "success": true,
  "data": {
    "id": "group-uuid",
    "name": "Bengaluru Weekend Trekkers",
    "description": "A community for weekend trekking enthusiasts.",
    "visibility": "private",
    "status": "active"
  },
  "message": "Group updated successfully."
}
```

Status codes: `200 OK`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`.

The client should not be allowed to change protected fields such as `created_by`, `status` or `parent_id` through this general update endpoint.

## 6. Delete a community

`DELETE /api/v1/communities/{id}`

Purpose: Soft-delete a community or sub-community, subject to authorization and applicable ownership rules.

Request body: None.

Path parameter: `id` — UUID of the group.

Response body:

```
{
  "success": true,
  "data": {
    "id": "group-uuid",
    "status": "deleted"
  },
  "message": "Group deleted successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`.

Business rule: Soft deletion should preserve historical records. Define whether deleting a parent community also soft-deletes its sub-communities or requires them to be handled separately.

## 7. Join a community

`POST /api/v1/communities/{id}/join`

Purpose: Allow an authenticated user to join a community or sub-community.

Request body: None. The user ID should be derived from the authenticated user's token.

Response body — successful public-group membership:

```
{
  "success": true,
  "data": {
    "group_id": "group-uuid",
    "user_id": "user-uuid",
    "membership_status": "active",
    "joined_at": "2026-10-11T10:00:00Z"
  },
  "message": "You have joined the group successfully."
}
```

Status codes: `200 OK` — already a member or idempotent success; `201 Created` — new membership; `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`.

For private communities requiring approval, return `202 Accepted` when a membership request has been submitted but not yet approved. For invite-only communities, require a valid invitation.

## 8. Leave a community

`POST /api/v1/communities/{id}/leave`

Purpose: Allow the authenticated user to leave a community or sub-community.

Request body: None.

Response body:

```
{
  "success": true,
  "data": {
    "group_id": "group-uuid",
    "user_id": "user-uuid",
    "membership_status": "left"
  },
  "message": "You have left the group successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`.

Business rule: A group should not be left without an eligible replacement if the user is its sole primary admin. The user must transfer ownership or appoint a replacement first.

## 9. Remove a member from a group

`DELETE /api/v1/communities/{id}/members/{userId}`

Purpose: Allow an authorized group admin or moderator to remove a member from the group.

Request body: None.

Path parameters:

- `id` — UUID of the group.
- `userId` — UUID of the member to remove.

Response body:

```
{
  "success": true,
  "data": {
    "group_id": "group-uuid",
    "user_id": "user-uuid",
    "membership_status": "removed"
  },
  "message": "Member removed from the group successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`.

Business rule: A moderator should not be able to remove the group owner or an admin with equal or higher privileges. Record the acting user's ID and the reason for removal in the audit log.

## 10. Promote a member to moderator

`POST /api/v1/communities/{id}/moderators/{userId}`

Purpose: Allow an authorized group creator/admin to promote an existing member to moderator.

Request body: None.

Path parameters:

- `id` — UUID of the group.
- `userId` — UUID of the member to promote.

Response body:

```
{
  "success": true,
  "data": {
    "group_id": "group-uuid",
    "user_id": "user-uuid",
    "role": "MODERATOR"
  },
  "message": "Member promoted to moderator successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`.

Business rule: The user must already be a member of the group. Promotion grants moderator permissions only within the specified group unless additional authorization is explicitly assigned.

## 11. List communities awaiting approval

`GET /api/v1/admin/communities/pending`

Purpose: Allow authorized platform administrators or designated approvers to retrieve user-created communities awaiting approval.

Request body: None.

Optional query parameters: `page`, `page_size`, `sort_by`, `sort_order`.

Response body:

```
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "group-uuid",
        "name": "Weekend Trekkers",
        "interest_id": "interest-uuid",
        "parent_id": "parent-group-uuid",
        "created_by": "user-uuid",
        "status": "pending",
        "visibility": "public",
        "created_at": "2026-10-11T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 10,
      "total": 1,
      "total_pages": 1
    }
  },
  "message": "Pending communities retrieved successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `400 Bad Request`.

## 12. Approve a pending group

`POST /api/v1/admin/communities/{id}/approve`

Purpose: Approve a pending user-created group and make it active.

Request body: None.

Path parameter: `id` — UUID of the group awaiting approval.

Response body:

```
{
  "success": true,
  "data": {
    "id": "group-uuid",
    "status": "active",
    "approved_by": "admin-uuid",
    "approved_at": "2026-10-11T10:00:00Z"
  },
  "message": "Group approved successfully."
}
```

Status codes: `200 OK`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`.

Business rule: Only a group in `pending` status can be approved. Approval should be recorded for auditing, and the group creator may be notified.

## 13. Reject a pending group

`POST /api/v1/admin/communities/{id}/reject`

Purpose: Reject a pending user-created group and record the reason for rejection.

Request body:

```
{
  "reason": "The proposed group duplicates an existing community."
}
```

Path parameter: `id` — UUID of the group awaiting approval.

Response body:

```
{
  "success": true,
  "data": {
    "id": "group-uuid",
    "status": "rejected",
    "rejected_by": "admin-uuid",
    "rejection_reason": "The proposed group duplicates an existing community.",
    "rejected_at": "2026-10-11T10:00:00Z"
  },
  "message": "Group rejected successfully."
}
```

Status codes: `200 OK`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`.

Business rule: Only a pending group can be rejected. The rejection reason should be retained and communicated to the creator where appropriate.

## Common error status codes

| Status                      | Meaning                         | Typical scenario                                  |
| --------------------------- | ------------------------------- | ------------------------------------------------- |
| `200 OK`                    | Request successful              | Fetch, update, join/leave or moderation operation |
| `201 Created`               | Resource created                | Group or membership created                       |
| `202 Accepted`              | Request accepted for processing | Membership awaiting approval                      |
| `400 Bad Request`           | Invalid request                 | Malformed parameters                              |
| `401 Unauthorized`          | Authentication required         | Missing or invalid token                          |
| `403 Forbidden`             | Permission denied               | Insufficient role or membership                   |
| `404 Not Found`             | Resource not found              | Group or user does not exist or is not accessible |
| `409 Conflict`              | State conflict                  | Duplicate membership or invalid status transition |
| `422 Unprocessable Entity`  | Validation failure              | Invalid field values or business-rule violation   |
| `500 Internal Server Error` | Unexpected server error         | Unhandled backend failure                         |

## Important design decisions before freezing the API

1. Default Interest Communities: create these automatically when interests are created. Users should create sub-communities under them, rather than creating duplicate default communities.
2. Approval ownership: decide whether the parent Interest Community Admin or a platform-level admin approves sub-community requests. The endpoint names under `/api/v1/admin/` currently imply platform-admin access.
3. Group roles: define `MEMBER`, `MODERATOR` and `ADMIN` as group-scoped roles. Keep platform-admin privileges separate.
4. Status values: use a consistent lifecycle, such as `PENDING`, `ACTIVE`, `REJECTED`, `SUSPENDED` and `DELETED`. Store approval/rejection metadata where required.
5. Response consistency: standardize timestamps, UUID formats, pagination, validation errors and response envelopes across all APIs.

These are proposed API contracts for your BuddyAdda LLD. The approval workflow, role boundaries and exact status codes should be agreed upon before implementation.




























