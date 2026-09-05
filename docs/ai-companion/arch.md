# AI Wellness Companion — Architecture

## 1. Objective

Build a small AI wellness companion inside BuddyAdda that can:

* record and read mood entries
* create and recall journal entries
* record and read wellness activities
* provide simple, non-clinical wellness guidance
* reflect on the user's own stored wellness data

The companion should remain a module inside the existing backend and should not be deployed as a separate AI microservice for the MVP.

---

## 2. Tech Stack

| Layer             | Technology                           | Purpose                                                 |
| ----------------- | ------------------------------------ | ------------------------------------------------------- |
| API               | FastAPI                              | Expose the companion endpoint                           |
| AI / Tool Calling | LangChain                            | Bind the LLM to approved Python tools                   |
| LLM               | `gpt-oss-120b`                       | Intent understanding and response generation            |
| Validation        | Pydantic                             | Validate tool arguments                                 |
| Database          | PostgreSQL                           | Store BuddyAdda user and wellness data                  |
| ORM / DB Access   | SQLAlchemy / async PostgreSQL driver | Read and write application data                         |
| Backend Language  | Python                               | Companion, services, repositories                       |
| Optional Later    | LangGraph                            | Add only if flows become multi-step or stateful         |
| Optional Later    | Redis                                | Add only for rate limiting, cache, or short-lived state |

### MVP Decision

Do **not** require:

* vector database
* RAG pipeline
* message queue
* separate AI database
* separate AI deployment
* Redis for core functionality
* LangGraph for simple one-tool flows

---

## 3. Core Data Used

The companion only needs these existing BuddyAdda tables:

```text
users
user_profile
mood_entries
journal_entries
user_wellness_activities
```

Relationship:

```text
users.id
   │
   ├── user_profile.user_id
   ├── mood_entries.user_id
   ├── journal_entries.user_id
   └── user_wellness_activities.user_id
```

`user_id` must always come from the authenticated application session.

The LLM must never choose or override the user identity.

---

## 4. High-Level Architecture

```text
                    BuddyAdda User
                          │
                          ▼
                 POST /companion/message
                          │
                          ▼
                       FastAPI
                          │
                          ▼
                 Companion Controller
                          │
                          ▼
                 LangChain + LLM
                          │
               selects approved tool
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
     Mood Tools      Journal Tools   Wellness Tools
          │               │               │
          ▼               ▼               ▼
     Python Service   Python Service   Python Service
          │               │               │
          └───────────────┼───────────────┘
                          ▼
                     Repository
                          │
                          ▼
                     PostgreSQL
```

### End-to-End Architecture

```text
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                      BUDDYADDA                                           │
│                                                                                          │
│   ┌───────────────────┐                                                                  │
│   │       USER        │                                                                  │
│   │                   │                                                                  │
│   │ "I feel anxious"  │                                                                  │
│   │ "Save my journal" │                                                                  │
│   │ "How was my week?"│                                                                  │
│   └─────────┬─────────┘                                                                  │
│             │                                                                            │
│             │ message                                                                    │
│             ▼                                                                            │
│   ┌─────────────────────────────────────────────┐                                        │
│   │              FASTAPI BACKEND                │                                        │
│   │                                             │                                        │
│   │        POST /companion/message              │                                        │
│   └──────────────────────┬──────────────────────┘                                        │
│                          │                                                               │
│                          ▼                                                               │
│   ┌─────────────────────────────────────────────┐                                        │
│   │            AUTHENTICATION LAYER             │                                        │
│   │                                             │                                        │
│   │     Resolve authenticated application user │                                        │
│   │                                             │                                        │
│   │     user_id = authenticated_user_id         │                                        │
│   └──────────────────────┬──────────────────────┘                                        │
│                          │                                                               │
│                          │ user_id + message                                              │
│                          ▼                                                               │
│   ┌─────────────────────────────────────────────┐                                        │
│   │           AI COMPANION CONTROLLER           │                                        │
│   │                                             │                                        │
│   │  Builds request context and passes the      │                                        │
│   │  user's message to the companion            │                                        │
│   └──────────────────────┬──────────────────────┘                                        │
│                          │                                                               │
│                          ▼                                                               │
│   ┌─────────────────────────────────────────────┐                                        │
│   │          LANGCHAIN + gpt-oss-120b           │                                        │
│   │                                             │                                        │
│   │  Understand request                         │                                        │
│   │  Extract required arguments                 │                                        │
│   │  Select approved tool when required         │                                        │
│   └──────────────────────┬──────────────────────┘                                        │
│                          │                                                               │
│                          ▼                                                               │
│                  ┌─────────────────┐                                                     │
│                  │ TOOL REQUIRED ? │                                                     │
│                  └───────┬─────────┘                                                     │
│                          │                                                               │
│              ┌───────────┴────────────┐                                                  │
│              │                        │                                                  │
│             NO                       YES                                                 │
│              │                        │                                                  │
│              ▼                        ▼                                                  │
│   ┌─────────────────────┐    ┌──────────────────────────────────────────────────────┐   │
│   │ GENERAL WELLNESS    │    │               APPROVED TOOL REGISTRY                 │   │
│   │ RESPONSE            │    │                                                      │   │
│   │                     │    │  Mood               Journal             Wellness     │   │
│   │ No DB access needed │    │  ──────────         ───────────         ──────────    │   │
│   │ for simple guidance │    │  record_mood        create_journal      record_       │   │
│   │                     │    │  get_mood_history   get_journal_history wellness_     │   │
│   │                     │    │  get_mood_summary   get_last_journal    activity      │   │
│   │                     │    │                                        get_wellness_ │   │
│   │                     │    │                                        history       │   │
│   └──────────┬──────────┘    └────────────────────────┬─────────────────────────────┘   │
│              │                                        │                                 │
│              │                                        ▼                                 │
│              │                         ┌──────────────────────────────┐                  │
│              │                         │     PYDANTIC VALIDATION      │                  │
│              │                         │                              │                  │
│              │                         │ Validate mood, scale, dates, │                  │
│              │                         │ title, content, type, etc.   │                  │
│              │                         └──────────────┬───────────────┘                  │
│              │                                        │                                 │
│              │                                        ▼                                 │
│              │                         ┌──────────────────────────────┐                  │
│              │                         │      APPLICATION CONTEXT     │                  │
│              │                         │                              │                  │
│              │                         │ Inject trusted user_id       │                  │
│              │                         │                              │                  │
│              │                         │ LLM cannot provide user_id   │                  │
│              │                         └──────────────┬───────────────┘                  │
│              │                                        │                                 │
│              │                                        ▼                                 │
│              │                         ┌──────────────────────────────┐                  │
│              │                         │       PYTHON SERVICES        │                  │
│              │                         │                              │                  │
│              │                         │ Wellness business logic      │                  │
│              │                         │ Read / Write orchestration   │                  │
│              │                         └──────────────┬───────────────┘                  │
│              │                                        │                                 │
│              │                                        ▼                                 │
│              │                         ┌──────────────────────────────┐                  │
│              │                         │          REPOSITORY          │                  │
│              │                         │                              │                  │
│              │                         │ SQLAlchemy / async driver    │                  │
│              │                         │ Parameterised DB operations  │                  │
│              │                         └──────────────┬───────────────┘                  │
│              │                                        │                                 │
│              │                                        ▼                                 │
│              │          ┌───────────────────────────────────────────────────────────┐    │
│              │          │                      POSTGRESQL                           │    │
│              │          │                                                           │    │
│              │          │   ┌───────────────┐        ┌──────────────────────────┐   │    │
│              │          │   │     users     │───────▶│      user_profile        │   │    │
│              │          │   └───────┬───────┘        └──────────────────────────┘   │    │
│              │          │           │                                               │    │
│              │          │           ├────────▶ mood_entries                         │    │
│              │          │           │                                               │    │
│              │          │           ├────────▶ journal_entries                      │    │
│              │          │           │                                               │    │
│              │          │           └────────▶ user_wellness_activities             │    │
│              │          │                                                           │    │
│              │          │     Every query scoped by authenticated user_id           │    │
│              │          └─────────────────────────────┬─────────────────────────────┘    │
│              │                                        │                                 │
│              │                                        │ tool result                     │
│              │                                        ▼                                 │
│              │                         ┌──────────────────────────────┐                  │
│              │                         │     GROUNDED AI RESPONSE     │                  │
│              │                         │                              │                  │
│              │                         │ LLM receives only the data   │                  │
│              │                         │ returned by approved tools   │                  │
│              │                         └──────────────┬───────────────┘                  │
│              │                                        │                                 │
│              └───────────────────────┬────────────────┘                                 │
│                                      │                                                  │
│                                      ▼                                                  │
│                         ┌──────────────────────────────┐                                 │
│                         │       SAFETY BOUNDARY        │                                 │
│                         │                              │                                 │
│                         │ General wellness     → allow │                                 │
│                         │ Reflection           → allow │                                 │
│                         │ Diagnosis            → block │                                 │
│                         │ Medication advice    → block │                                 │
│                         │ Immediate danger     → safety│                                 │
│                         │                        path  │                                 │
│                         └──────────────┬───────────────┘                                 │
│                                        │                                                 │
│                                        ▼                                                 │
│                         ┌──────────────────────────────┐                                 │
│                         │        API RESPONSE          │                                 │
│                         └──────────────┬───────────────┘                                 │
│                                        │                                                 │
│                                        ▼                                                 │
│                         ┌──────────────────────────────┐                                 │
│                         │            USER              │                                 │
│                         │                              │                                 │
│                         │ Grounded wellness response   │                                 │
│                         └──────────────────────────────┘                                 │
│                                                                                          │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Request Flow

```text
1. User sends a message
        ↓
2. FastAPI resolves authenticated user_id
        ↓
3. Companion sends message to LLM
        ↓
4. LLM decides:
      - answer directly
      - or call an approved tool
        ↓
5. Python validates tool arguments
        ↓
6. Application injects user_id
        ↓
7. Service / repository reads or writes PostgreSQL
        ↓
8. Result is returned to the LLM
        ↓
9. LLM produces a grounded response
        ↓
10. FastAPI returns the response
```

---

## 6. Companion Tools

The MVP needs only eight tools.

### Mood

```text
record_mood()
get_mood_history()
get_mood_summary()
```

### Journal

```text
create_journal_entry()
get_journal_history()
get_last_journal()
```

### Wellness

```text
record_wellness_activity()
get_wellness_history()
```

No generic SQL tool should be exposed to the LLM.

---

## 7. Security and Privacy

### Identity

All database access must use:

```text
user_id = authenticated_user_id
```

### LLM Boundary

The LLM must not:

* generate unrestricted SQL
* provide `user_id`
* access another user's wellness records
* decide which database tables exist
* call tools outside the approved registry

### Private Data

The MVP should not read:

```text
user-chats (with other users)
friendships
groups
posts
events
notifications
user_reports
```

They are outside the wellness MVP.

---

## 8. Safety Boundary

The companion can provide:

* simple reflection
* journaling prompts
* breathing / pause suggestions
* hydration / walking / rest suggestions
* encouragement to talk to someone they trust

The companion must not:

* diagnose mental-health conditions
* recommend prescription medication
* claim to be a therapist
* make medical conclusions from mood history

A separate safety response path should handle messages indicating immediate danger or self-harm risk.
