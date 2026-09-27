## Data Dictionary

### 1. `users`

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

### 2. `features`

| Column | Data Type | Key / Constraint | Description        |
| ------ | --------- | ---------------- | ------------------ |
| `id`   | UUID      | PK               | Feature identifier |
| `name` | VARCHAR   | —                | Feature name       |

### 3. `user_profile_features`

| Column         | Data Type | Key / Constraint    | Description                        |
| -------------- | --------- | ------------------- | ---------------------------------- |
| `user_id`      | UUID      | PK, FK → `users.id` | User identifier; cascade on delete |
| `feature_type` | VARCHAR   | —                   | Type/name of feature               |
| `is_active`    | BOOL      | —                   | Whether feature is active for user |

### 4. `user_profile`

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

### 5. `interests`

| Column     | Data Type | Key / Constraint | Description       |
| ---------- | --------- | ---------------- | ----------------- |
| `name`     | VARCHAR   | PK               | Interest name     |
| `category` | ENUM      | `Interest1`      | Interest category |

### 6. `user_interests`

| Column     | Data Type | Key / Constraint      | Description                                               |
| ---------- | --------- | --------------------- | --------------------------------------------------------- |
| `id`       | UUID      | PK, FK → `users.id`   | Record identifier; schema currently also references users |
| `user_id`  | UUID      | FK → `users.id`       | User identifier; cascade on delete                        |
| `interest` | VARCHAR   | FK → `interests.name` | Selected interest; cascade/update relationship            |

### 7. `languages`

| Column | Data Type | Key / Constraint | Description   |
| ------ | --------- | ---------------- | ------------- |
| `name` | VARCHAR   | PK               | Language name |

### 8. `user_languages`

| Column     | Data Type | Key / Constraint      | Description                                |
| ---------- | --------- | --------------------- | ------------------------------------------ |
| `id`       | UUID      | PK                    | Record identifier                          |
| `user_id`  | UUID      | FK → `users.id`       | User identifier; cascade on delete         |
| `language` | VARCHAR   | FK → `languages.name` | User language; cascade/update relationship |

### 9A. `friend_requests`

| Column        | Data Type | Key / Constraint                  | Description               |
| ------------- | --------- | --------------------------------- | ------------------------- |
| `id`          | VARCHAR   | PK                                | UUID                      |
| `sender_id`   | UUID      | FK → `users.id`                   | Friendship requester      |
| `receiver_id` | UUID      | FK → `users.id`                   | Friendship recipient      |
| `status`      | ENUM      | `pending`, `accepted`, `declined` | Friendship Request status |
| `sent_at`     | TIMESTAMP | —                                 | Request sent time         |
| `accepted_at` | TIMESTAMP | —                                 | Request accepted time     |

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

### 9B. `friendships`

| Column              | Data Type | Key / Constraint                         | Description                 |
| ------------------- | --------- | ---------------------------------------- | --------------------------- |
| `id`                | VARCHAR   | PK                                       | UUID                        |
| `from_user`         | UUID      | FK → `users.id`                          | User 1                      |
| `to_user`           | UUID      | FK → `users.id`                          | User 2                      |
| `status`            | ENUM      | `muted`, `blocked`, `friend`, `un-friend` | Friendship status           |
| `friend_request_id` | VARCHAR   | FK → `friend_requests.id`                | Reference to friend request |

### 10. `groups`

| Column             | Data Type | Key / Constraint                   | Description                |
| ------------------ | --------- | ---------------------------------- | -------------------------- |
| `id`               | UUID      | PK                                 | Group identifier           |
| `name`             | VARCHAR   | —                                  | Group name                 |
| `parent_id`        | UUID      | —                                  | Parent group identifier    |
| `is_admin_managed` | BOOL      | —                                  | Whether managed by admin   |
| `visibility`       | ENUM      | `public`, `private`, `invite_only` | Group visibility           |
| `description`      | VARCHAR   | —                                  | Group description          |
| `status`           | VARCHAR   | —                                  | Group status               |
| `created_by`       | UUID      | FK → `users.id`                    | User who created the group |

### 11. `group_interests`

| Column        | Data Type | Key / Constraint      | Description                    |
| ------------- | --------- | --------------------- | ------------------------------ |
| `group_id`    | UUID      | FK → `groups.id`      | Group identifier               |
| `interest_id` | VARCHAR   | FK → `interests.name` | Interest associated with group |

### 12. `group_members`

| Column         | Data Type | Key / Constraint | Description                   |
| -------------- | --------- | ---------------- | ----------------------------- |
| `user_id`      | UUID      | FK → `users.id`  | Group member                  |
| `group_id`     | UUID      | FK → `groups.id` | Group                         |
| `is_moderator` | BOOL      | —                | Whether member is a moderator |
| `joined_at`    | TIMESTAMP | —                | Date/time user joined         |

### 13. `posts`

| Column       | Data Type | Key / Constraint | Description                     |
| ------------ | --------- | ---------------- | ------------------------------- |
| `id`         | UUID      | PK               | Post identifier                 |
| `group_id`   | UUID      | FK → `groups.id` | Group containing post           |
| `content`    | VARCHAR   | —                | Post content                    |
| `type`       | ENUM      | —                | Post type                       |
| `extra_data` | JSONB     | —                | Additional structured post data |
| `user_id`    | UUID      | FK → `users.id`  | User who created post           |
| `visibility` | VARCHAR   | —                | Post visibility                 |

### 14. `post_interactions`

| Column          | Data Type | Key / Constraint | Description                 |
| --------------- | --------- | ---------------- | --------------------------- |
| `user_id`       | UUID      | FK → `users.id`  | User performing interaction |
| `post_id`       | UUID      | FK → `posts.id`  | Post being interacted with  |
| `type`          | VARCHAR   | —                | Interaction type            |
| `interaction`   | VARCHAR   | —                | Interaction value/action    |
| `longitude_dim` | DOUBLE    | —                | Auto-fetched longitude      |
| `latitude_dim`  | DOUBLE    | —                | Auto-fetched latitude       |

### 15. `events`

| Column          | Data Type | Key / Constraint | Description                 |
| --------------- | --------- | ---------------- | --------------------------- |
| `id`            | UUID      | PK               | Event identifier            |
| `user_id`       | UUID      | FK → `users.id`  | User associated with event  |
| `group_id`      | UUID      | FK → `groups.id` | Group associated with event |
| `starts_at`     | TIMESTAMP | —                | Event start time            |
| `ends_at`       | TIMESTAMP | —                | Event end time              |
| `location`      | VARCHAR   | —                | Event location              |
| `title`         | VARCHAR   | —                | Event title                 |
| `description`   | VARCHAR   | —                | Event description           |
| `longitude_dim` | DOUBLE    | —                | Auto-fetched longitude      |
| `latitude_dim`  | DOUBLE    | —                | Auto-fetched latitude       |

### 16. `events_interactions`

| Column        | Data Type | Key / Constraint | Description                 |
| ------------- | --------- | ---------------- | --------------------------- |
| `id`          | UUID      | PK               | Interaction identifier      |
| `user_id`     | UUID      | FK → `users.id`  | User interacting with event |
| `event_id`    | UUID      | FK → `events.id` | Event identifier            |
| `type`        | VARCHAR   | —                | Interaction type            |
| `interaction` | VARCHAR   | —                | Interaction/action value    |

### 17. `chats`

| Column        | Data Type | Key / Constraint | Description                         |
| ------------- | --------- | ---------------- | ----------------------------------- |
| `id`          | UUID      | PK               | Chat/message identifier             |
| `target_type` | VARCHAR   | —                | Type of chat target                 |
| `target_id`   | UUID      | —                | Target user/group/entity identifier |
| `sender_id`   | UUID      | FK → `users.id`  | Message sender                      |
| `content`     | VARCHAR   | —                | Message content                     |
| `sent_at`     | TIMESTAMP | —                | Message sent time                   |
| `ref_type`    | VARCHAR   | —                | Referenced entity type              |
| `ref_id`      | UUID      | —                | Referenced entity identifier        |

### 18. `assets`

| Column        | Data Type     | Key / Constraint | Description                        |
| ------------- | ------------- | ---------------- | ---------------------------------- |
| `id`          | UUID          | PK               | Asset identifier                   |
| `target_type` | VARCHAR       | —                | Entity type asset belongs to       |
| `target_id`   | UUID          | —                | Entity identifier                  |
| `url`         | VARCHAR(2048) | —                | Asset URL; stored in Cloudflare R2 |
| `type`        | VARCHAR       | —                | Asset type                         |

### 19. `notifications`

| Column        | Data Type | Key / Constraint | Description                       |
| ------------- | --------- | ---------------- | --------------------------------- |
| `id`          | UUID      | PK               | Notification identifier           |
| `user_id`     | UUID      | FK → `users.id`  | Notification recipient            |
| `type`        | VARCHAR   | —                | Notification type                 |
| `content`     | JSONB     | —                | Structured notification content   |
| `is_read`     | BOOL      | —                | Whether notification was read     |
| `is_archived` | BOOL      | —                | Whether notification was archived |

### 20. `journal_entries`

| Column       | Data Type | Key / Constraint | Description                      |
| ------------ | --------- | ---------------- | -------------------------------- |
| `id`         | UUID      | PK               | Journal entry identifier         |
| `user_id`    | UUID      | FK → `users.id`  | Journal owner                    |
| `entry_date` | DATE      | —                | Journal date; can be a past date |
| `title`      | VARCHAR   | —                | Journal title                    |
| `content`    | TEXT      | —                | Journal content                  |

### 21. `mood_entries`

| Column       | Data Type | Key / Constraint | Description                                |
| ------------ | --------- | ---------------- | ------------------------------------------ |
| `id`         | UUID      | PK               | Mood entry identifier                      |
| `user_id`    | UUID      | FK → `users.id`  | User                                       |
| `entry_time` | TIMESTAMP | —                | Mood entry time; can represent a past date |
| `mood`       | VARCHAR   | —                | Mood value; may later become ENUM          |
| `scale`      | NUMBER    | —                | Numeric mood scale                         |

### 22. `user_wellness_activities`

| Column       | Data Type | Key / Constraint | Description                  |
| ------------ | --------- | ---------------- | ---------------------------- |
| `id`         | UUID      | PK               | Wellness activity identifier |
| `user_id`    | UUID      | FK → `users.id`  | User                         |
| `entry_time` | TIMESTAMP | —                | Activity timestamp           |
| `content`    | VARCHAR   | —                | Activity details             |
| `type`       | VARCHAR   | —                | Wellness activity type       |

### 23. `user_reports`

| Column          | Data Type | Key / Constraint | Description                         |
| --------------- | --------- | ---------------- | ----------------------------------- |
| `id`            | UUID      | PK               | Report identifier                   |
| `created_by`    | UUID      | FK → `users.id`  | User submitting report              |
| `reported_user` | UUID      | FK → `users.id`  | User being reported                 |
| `reported_from` | VARCHAR   | —                | Source UI, e.g. chat or post        |
| `incident_time` | TIMESTAMP | —                | Time of reported incident           |
| `extra_remarks` | VARCHAR   | —                | Additional remarks                  |
| `report_type`   | VARCHAR   | —                | Report category; candidate for ENUM |
| `reviewed_by`   | UUID      | FK → `users.id`  | User/admin reviewing report         |
| `verdict`       | VARCHAR   | —                | Review outcome                      |

**Total: 24 active tables.**

One schema issue worth flagging: in `user_interests`, `id` is defined as both the **primary key and a foreign key to `users.id`**. That looks accidental because `user_id` already references `users.id`; normally `id` would just be the independent UUID PK.
