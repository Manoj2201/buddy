# AI Wellness Companion — MVP Usage & Scenarios

## 1. Product Objective

The BuddyAdda AI Wellness Companion provides a conversational interface for lightweight wellness tracking and reflection.

The MVP supports four core user needs:

* record how they feel
* maintain short journal entries
* log simple wellness activities
* receive grounded, non-clinical guidance based on their own history

The companion uses BuddyAdda's existing relational data as the source of truth and does not maintain a separate wellness profile.

---

# 2. MVP Scope

| Capability                      | In Scope | Data Source                 |
| ------------------------------- | -------: | --------------------------- |
| Mood logging                    |      Yes | `mood_entries`              |
| Mood history                    |      Yes | `mood_entries`              |
| Mood trend summary              |      Yes | `mood_entries`              |
| Journal creation                |      Yes | `journal_entries`           |
| Journal history                 |      Yes | `journal_entries`           |
| Wellness activity logging       |      Yes | `user_wellness_activities`  |
| Wellness activity history       |      Yes | `user_wellness_activities`  |
| Simple wellness guidance        |      Yes | LLM + optional user history |
| Medical diagnosis               |       No | —                           |
| Medication advice               |       No | —                           |
| Chat history access             |       No | —                           |
| Community/event recommendations |       No | —                           |

The MVP intentionally uses only the five core tables already identified in the existing design.

---

# 3. Data Model Used by the Companion

## 3.1 Core Schema

```text
users
  │
  ├── user_profile
  ├── mood_entries
  ├── journal_entries
  └── user_wellness_activities
```

### `users`

Primary identity source.

```text
id
status
```

### `user_profile`

Used only for light personalisation.

```text
user_id
name_first
name_full
bio
profession
```

### `mood_entries`

```text
id
user_id
entry_time
mood
scale
```

### `journal_entries`

```text
id
user_id
entry_date
title
content
```

### `user_wellness_activities`

```text
id
user_id
entry_time
content
type
```

All private wellness data is scoped through:

```text
user_id = authenticated_user_id
```

The LLM must never provide or override `user_id`.

---

# 4. System Interaction Model

The AI layer is responsible for understanding intent and generating responses.

Python services are responsible for data access.

```text
User
  ↓
FastAPI Endpoint
  ↓
AI Companion
  ↓
Intent / Tool Selection
  ↓
Python Service
  ↓
Repository
  ↓
PostgreSQL
```

The LLM does not generate SQL.

The LLM invokes predefined application functions such as:

```text
record_mood
get_mood_history
get_mood_summary
create_journal_entry
get_journal_history
get_last_journal
record_wellness_activity
get_wellness_history
```

---

# 5. User Stories

## US-01 — Record Current Mood

**As a user,**
I want to tell the companion how I am feeling
**so that** my mood is recorded without opening a separate form.

### Example

```text
User:
"I'm feeling anxious today."
```

### Expected system behaviour

1. Identify intent as mood logging.
2. Extract `mood = anxious`.
3. Use authenticated `user_id`.
4. Insert a new row into `mood_entries`.
5. Confirm the write to the user.

### Data operation

```text
INSERT mood_entries
    user_id
    entry_time
    mood
    scale
```

### Acceptance criteria

* A mood entry is created for the authenticated user.
* `entry_time` is system generated.
* `scale` may be null.
* No other user's data can be written.

---

# 6. US-02 — Record Mood With Intensity

**As a user,**
I want to record how strongly I feel a mood
**so that** I can track intensity over time.

### Example

```text
"I'm stressed, around 4 out of 5."
```

### Tool input

```json
{
  "mood": "stressed",
  "scale": 4
}
```

### Expected database record

```text
mood = stressed
scale = 4
```

### Acceptance criteria

* `scale` is stored when provided.
* Missing intensity must not block mood logging.
* Invalid scale values must be rejected by application validation.

---

# 7. US-03 — View Mood History

**As a user,**
I want to ask how I have been feeling recently
**so that** I can understand simple patterns in my mood logs.

### Example

```text
"How have I been feeling this week?"
```

### Application flow

```text
get_mood_history(
    user_id=current_user,
    start_date=start_of_week,
    end_date=today
)
```

### Example result

```text
anxious    2
relaxed    2
happy      1
```

### Response rule

The assistant must describe the records, not infer a diagnosis.

Good:

```text
"Based on your entries this week, anxious and relaxed were each logged twice."
```

Avoid:

```text
"You are becoming emotionally unstable."
```

---

# 8. US-04 — Compare Mood Over Time

**As a user,**
I want to compare my recent mood history
**so that** I can see whether my recorded pattern is changing.

### Example

```text
"Do you think I've been doing better?"
```

### Required grounding

```text
Current period
vs
Previous period
```

Example:

```text
Previous week
anxious = 4
relaxed = 1

Current week
anxious = 1
relaxed = 3
```

### Expected response

```text
"Based on what you've logged, anxious appeared less often this week and relaxed appeared more often."
```

### Acceptance criteria

* Response must use stored mood data.
* No clinical conclusion is allowed.
* The response must state the evidence boundary using wording such as "based on what you've logged."

---

# 9. US-05 — Create Journal Entry

**As a user,**
I want to write a journal entry through conversation
**so that** journaling feels natural and quick.

### Example

```text
"Today was stressful at work, but I felt better after going outside."
```

### Application call

```text
create_journal_entry(
    user_id=current_user,
    title="Work stress",
    content="Today was stressful at work, but I felt better after going outside."
)
```

### Data operation

```text
journal_entries
    user_id
    entry_date
    title
    content
```

### Acceptance criteria

* Entry is saved against the authenticated user.
* Journal content is preserved.
* The companion confirms that the entry was saved.

---

# 10. US-06 — Recall Previous Journal

**As a user,**
I want to retrieve my previous journal entry
**so that** I can remember what I wrote.

### Example

```text
"What did I write last time?"
```

### Application call

```text
get_last_journal(user_id=current_user)
```

### Query behaviour

```text
WHERE user_id = authenticated_user_id
ORDER BY entry_date DESC
LIMIT 1
```

### Expected response

Return the most recent entry in plain language.

---

# 11. US-07 — Summarise Recent Journal Themes

**As a user,**
I want to understand what I have been writing about recently
**so that** I can reflect on recurring topics.

### Example

```text
"What have I been writing about lately?"
```

### Application flow

```text
get_journal_history(limit=5)
```

The LLM may summarise themes only from the returned journal entries.

Example:

```text
Work pressure
Poor sleep
Walking
Meeting friends
Productive days
```

### Acceptance criteria

* Summary must be grounded in retrieved entries.
* The assistant must not invent topics that were not present.

---

# 12. US-08 — Log Wellness Activity

**As a user,**
I want to record simple wellness activities
**so that** I can build a basic history of habits.

### Example

```text
"I walked for 30 minutes."
```

### Application call

```text
record_wellness_activity(
    user_id=current_user,
    type="walking",
    content="Walked for 30 minutes"
)
```

### Database

```text
user_wellness_activities
```

### Acceptance criteria

* Activity is recorded with timestamp.
* Activity is linked to the authenticated user.
* `type` and `content` are stored.

---

# 13. US-09 — View Wellness Activity History

**As a user,**
I want to see what wellness activities I have completed recently
**so that** I can reflect on consistency.

### Example

```text
"What healthy things have I done this week?"
```

### Application flow

```text
get_wellness_history(
    start_date=start_of_week
)
```

### Example response

```text
"You logged three walks, two meditation sessions, and one exercise session this week."
```

---

# 14. US-10 — Receive Simple Wellness Guidance

**As a user,**
I want simple support when I feel stressed or overwhelmed
**so that** I can take a practical next step.

### Example

```text
"I'm feeling stressed. What should I do?"
```

### Expected behaviour

The companion may provide simple, low-risk suggestions such as:

* take a short walk
* pause and breathe slowly
* write down what is causing stress
* drink water
* take a short break
* create a journal entry

No database read is required unless personalisation is needed.

---

# 15. US-11 — Personalised Guidance From History

**As a user,**
I want guidance that considers what has helped me before
**so that** the response feels relevant.

### Example

```text
"What usually helps me when I'm stressed?"
```

### Data sources

```text
mood_entries
journal_entries
user_wellness_activities
```

### Example evidence

```text
Journal:
"I felt better after walking."

Wellness activity:
walking logged several times
```

### Expected response

```text
"From your recent entries, walking appears several times around days when you later described feeling better. A short walk may be worth trying again."
```

### Important rule

The system should describe correlation in the data, not claim causation.

---

# 16. US-12 — Open Supportive Conversation

**As a user,**
I want to talk when I am having a difficult day
**so that** I can decide whether to reflect, log my mood, or journal.

### Example

```text
"I had a difficult day."
```

### Expected response behaviour

The assistant may continue the conversation without immediately writing data.

Possible next actions:

```text
Talk
Log Mood
Create Journal Entry
```

A write should happen only when the user expresses a clear intent to save something.

---

# 17. Functional Rules

## FR-01 — Authentication

All reads and writes must use the authenticated application user.

```text
user_id = authenticated_user_id
```

---

## FR-02 — LLM Data Boundary

The LLM must not:

```text
generate SQL
select arbitrary user IDs
query unrestricted tables
access another user's wellness data
```

---

## FR-03 — Grounded Reflection

Any statement about the user's historical wellbeing must be based on stored data.

---

## FR-04 — Explicit Writes

Mood, journal, and activity records are created only when user intent is sufficiently clear.

---

## FR-05 — Limited Scope

The MVP should not use:

```text
chats
groups
events
friendships
notifications
user_reports
```

---

# 18. Safety Boundaries

The companion supports general wellness.

It must not:

* diagnose mental-health conditions
* claim a user has depression, anxiety disorder, or another condition
* recommend prescription medication
* present itself as a therapist or medical professional
* make strong conclusions from limited wellness data

If the conversation indicates immediate danger or self-harm risk, the normal wellness flow should stop and the application should move to a dedicated safety response path.

---

# 19. Minimal Technical Surface

The MVP requires only eight application tools.

```text
Mood
├── record_mood
├── get_mood_history
└── get_mood_summary

Journal
├── create_journal_entry
├── get_journal_history
└── get_last_journal

Wellness
├── record_wellness_activity
└── get_wellness_history
```

This is the complete initial AI-to-database surface.

---

# 20. MVP Request Flow

```text
POST /companion/message
        │
        ▼
Authenticate User
        │
        ▼
Classify Request
        │
        ├── Mood
        ├── Journal
        ├── Wellness Activity
        └── General Wellness
        │
        ▼
Invoke Allowed Python Tool
        │
        ▼
Read / Write PostgreSQL
        │
        ▼
Generate Grounded Response
        │
        ▼
Return Response
```

---

# 21. Product Principle

The MVP should remain small enough that every response can be explained by one of three sources:

```text
1. User's current message
2. User's BuddyAdda wellness data
3. General low-risk wellness guidance
```

Anything outside those three sources is outside the initial product scope.
