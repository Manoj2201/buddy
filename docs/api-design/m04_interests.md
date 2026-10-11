# Interests API

**Base URL:** `/api/v1`

## Associated Tables

### A. 

### 1. `interests`

| Column     | Data Type | Key / Constraint | Description              |
| ---------- | --------- | ---------------- | ------------------------ |
| `name`     | VARCHAR   | PK               | Name of the interest     |
| `category` | ENUM      | `Interest1`      | Category of the interest |

### 2. `user_interests`

| Column     | Data Type | Key / Constraint                                            | Description                          |
| ---------- | --------- | ----------------------------------------------------------- | ------------------------------------ |
| `id`       | UUID      | PK, FK → `users.id`, ON DELETE CASCADE                      | Unique identifier; references a user |
| `user_id`  | UUID      | FK → `users.id`, ON DELETE CASCADE                          | User associated with the interest    |
| `interest` | VARCHAR   | FK → `interests.name`, ON DELETE CASCADE, ON UPDATE CASCADE | Interest associated with the user    |

Table group_interests {
  group_id uuid [ ref: > groups.id, note: 'On delete cascade']
  interest_id varchar [ ref: > interests.name, note: 'On delete cascade']
}

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

## APIs

Endpoints

GET /interests 
:: List the interests 

**Endpoints:** 

```text
POST   /interests 
:: Create interest by admin 

PUT    /interests
:: Link interest with user(create, update, delete)

GET    /interests
:: Returns list of interests 

DELETE /interests/{interest-id}
:: Delete the provided interest by the admin from the system(Soft delete)

```


## Interests API Specification

### 1. Create Interest by Admin

API: `POST /interests`

Purpose: Creates a new interest in the system. Accessible only to an authorized admin.

Response Body:

```
{
  "interestId": "INT001",
  "name": "Travel",
  "description": "Explore new places and cultures",
  "isActive": true,
  "createdAt": "2026-10-11T10:00:00Z"
}
```

Status: 200 OK

### 2. Link Interest with User

API: `PUT /interests`

Purpose: Creates, updates, or removes the association between a user and their selected interests.

Response Body:

```
{
  "userId": "USR001",
  "interests": [
    {
      "interestId": "INT001",
      "name": "Travel"
    },
    {
      "interestId": "INT002",
      "name": "Photography"
    }
  ],
  "message": "User interests updated successfully"
}
```

Status: 200 OK

### 3. Get List of Interests

API: `GET /interests`

Purpose: Returns the list of active interests available in the system for users to explore and select.

Response Body:

```
{
  "interests": [
    {
      "interestId": "INT001",
      "name": "Travel",
      "description": "Explore new places and cultures"
    },
    {
      "interestId": "INT002",
      "name": "Photography",
      "description": "Capture moments through photography"
    }
  ],
  "totalCount": 2
}
```

Status: 200 OK

### 4. Soft Delete Interest by Admin

API: `DELETE /interests/{interest-id}`

Purpose: Soft-deletes the specified interest from the system by marking it as inactive without permanently removing its record. Accessible only to an authorized admin.

Response Body:

```
{
  "interestId": "INT001",
  "isActive": false,
  "message": "Interest deleted successfully"
}
```

Status: 200 OK

Design recommendations for Buddy Adda:

- Use `201 Created` instead of `200 OK` for successful interest creation via `POST /interests`.
- Use `204 No Content` for successful deletion if you do not need to return a response body; otherwise, `200 OK` is appropriate.
- For `PUT /interests`, define the request body clearly to distinguish between adding, updating, and removing user-interest associations.
- Enforce admin authorization for creating and soft-deleting interests.



