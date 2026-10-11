# BuddyAdda — High-Level Design (Java + Spring Boot)

Since BuddyAdda has committed to Java, the recommendation is to retain the **Modular Monolith architecture** and replace the Python/FastAPI stack with Java 21 and Spring Boot. The frozen API design and ER diagram remain the foundation; the main change is how the backend is implemented.

## 1. High-Level Architecture

```text
                        ┌─────────────────────────┐
                        │          USERS          │
                        │   Web / Mobile Browser  │
                        └────────────┬────────────┘
                                     │ HTTPS
                                     ▼
                        ┌─────────────────────────┐
                        │      React Frontend     │
                        │ Login / Profile         │
                        │ Friends / Communities   │
                        │ Meetups / Wellness      │
                        │ Chat / Notifications    │
                        └────────────┬────────────┘
                                     │ REST APIs
                                     ▼
             ┌────────────────────────────────────────────┐
             │        JAVA 21 + SPRING BOOT BACKEND       │
             │                                            │
             │ Authentication │ User/Profile │ Friendship │
             │ Interest       │ Community    │ Meetups    │
             │ Wellness       │ Chat         │ Search     │
             │ Notifications  │ AI            │ Admin      │
             │ Moderation     │ Common       │ Config     │
             │                                            │
             │       Controller → Service → Repository   │
             └───────────────┬──────────────┬─────────────┘
                             │              │
                  ┌──────────▼─────┐ ┌──────▼──────────┐
                  │   PostgreSQL   │ │      Redis      │
                  │ Persistent data│ │ Optional cache, │
                  │                │ │ rate limits,     │
                  │                │ │ presence         │
                  └────────────────┘ └─────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   AWS S3         │
                    │ Profile images, │
                    │ community media │
                    │ and attachments │
                    └─────────────────┘
```

**Architecture note:** Redis, S3 and AWS deployment services can be introduced when required; they do not all need to be operational in the first development phase.

## 2.  BuddyAdda Technology Stack

| Layer                | Recommended Technology                                                   |
| -------------------- | ------------------------------------------------------------------------ |
| Frontend             | React.js                                                                 |
| Programming Language | Java 21 LTS                                                              |
| Backend Framework    | Spring Boot 3.x or a compatible current release                          |
| API Style            | REST                                                                     |
| Authentication       | Spring Security + JWT                                                    |
| ORM                  | Spring Data JPA + Hibernate                                              |
| Validation           | Jakarta Bean Validation                                                  |
| Database             | PostgreSQL                                                               |
| Database Migrations  | Flyway                                                                   |
| API Documentation    | Springdoc OpenAPI / Swagger UI                                           |
| Testing              | JUnit 5, Mockito, Spring Boot Test                                       |
| Build Tool           | Maven                                                                    |
| Cache                | Redis (when needed)                                                      |
| Containerization     | Docker                                                                   |
| Source Control       | GitHub                                                                   |
| CI/CD                | GitHub Actions                                                           |
| Web Server           | Embedded Tomcat                                                          |
| Cloud                | AWS                                                                      |
| Object Storage       | AWS S3                                                                   |
| Monitoring           | Spring Boot Actuator + CloudWatch                                        |
| AI Integration       | Java-based integration initially; separate AI service later if justified |

Flyway is a suitable alternative for managing database schema changes in a Java/Spring Boot project.

## 3. Backend Module Structure

Organize the application by **business module**, rather than creating one enormous collection of controllers, services and repositories.

```text
buddyadda-backend/
├── pom.xml
├── src/main/java/com/buddyadda/
│   ├── BuddyAddaApplication.java
│   ├── common/
│   ├── configuration/
│   ├── authentication/
│   ├── user/
│   ├── profile/
│   ├── friendship/
│   ├── interest/
│   ├── community/
│   ├── meetup/
│   ├── wellness/
│   ├── chat/
│   ├── notification/
│   ├── search/
│   ├── ai/
│   ├── admin/
│   └── moderation/
└── src/main/resources/
    ├── application.yml
    └── db/migration/
```

Inside a typical module:

```text
friendship/
├── controller/
├── service/
├── repository/
├── entity/
├── dto/
├── mapper/
├── exception/
└── event/
```

Each module owns its business logic. Other modules should interact through defined interfaces or application services, rather than directly modifying another module's entities or tables.

## 4. API Request Flow

For example, when a user sends a friend request:

```text
React frontend
      |
      v
FriendshipController
      |
      v
FriendshipService
(business rules and transaction)
      |
      v
FriendshipRepository
(Spring Data JPA)
      |
      v
PostgreSQL
```

The service checks relevant business rules, such as whether the target user exists, a block prevents the request, or a pending request already exists. The controller returns a response matching the frozen API contract.

## 5. Recommended Implementation Sequence

Since the API design and ER diagram are frozen, follow this sequence:

1. **Finalize the technology decision:** Java 21, Spring Boot, PostgreSQL, Maven and Flyway.
2. **Set up the project foundation:** application structure, configuration profiles, database connectivity, common responses, exception handling and logging.
3. **Translate the ER diagram:** define JPA entities, relationships, constraints and Flyway migrations. Validate that they match the approved schema.
4. **Complete LLD module by module:** classes, methods, service flows, validations, error cases, security and inter-module dependencies.
5. **Implement and test incrementally:** begin with Authentication, User, Profile, Interest and Friendship, then proceed to Community, Meetups and the remaining modules.
6. **Automate integration:** GitHub pull requests, automated tests, CI/CD and a shared development environment.

## 6. Scalability and Architecture Guidance

For a target of approximately **1,000 users**, a well-designed Spring Boot modular monolith with PostgreSQL is a sensible starting point. Add caching, background processing or independently deployed services later when measured requirements justify them.

The key is to preserve the existing API contracts and data model while ensuring that the Java implementation maintains the same business behavior.
