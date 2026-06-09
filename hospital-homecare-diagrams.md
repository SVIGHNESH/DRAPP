# Hospital Home-Care System — Diagrams

**Actors (2):**
- **End User / Relative** — logs in, browses services, books care for self or a family member, adds special demands.
- **Admin** — reviews booking requests, arranges a nurse **offline (outside the system)**, confirms the booking, records the assigned nurse.

> The **nurse is not a system user**. All admin ↔ nurse coordination happens outside this app; the system only stores the outcome.

---

## 1. Context Diagram (DFD Level 0)

```mermaid
flowchart TB
    User([End User / Relative])
    Admin([Admin])
    OAuth([OAuth Provider<br/>Google etc.])

    System(("Hospital Home-Care<br/>System"))

    User -->|Login, Book service, Special demands| System
    System -->|Service list, Booking status, Confirmation| User

    OAuth -->|Identity / token| System
    System -->|Auth request| OAuth

    System -->|Booking requests + demands| Admin
    Admin -->|Confirm booking, Assigned nurse info, Updates| System

    Admin -.->|Coordinates offline<br/>OUTSIDE SYSTEM| Ext([Nurse / Caregiver])
```

---

## 2. DFD Level 1 (processes + data stores)

```mermaid
flowchart TB
    User([End User / Relative])
    Admin([Admin])
    OAuth([OAuth Provider])

    P1[[1.0 Authenticate User]]
    P2[[2.0 Browse / Select Service]]
    P3[[3.0 Create Booking Request]]
    P4[[4.0 Review & Confirm Booking]]
    P5[[5.0 Handle Special Demands]]
    P6[[6.0 Track Service Status]]

    D1[(Users)]
    D2[(Services Catalog)]
    D3[(Bookings)]

    User -->|email / OAuth| P1
    OAuth <--> P1
    P1 <--> D1

    User --> P2
    P2 <--> D2
    P2 -->|selected service| P3

    P3 -->|member, time slot| D3
    P3 -->|new request| P4

    Admin --> P4
    P4 <--> D3
    P4 -->|assigned nurse info<br/>arranged offline| D3
    P4 -->|confirmation| User

    User <--> P5
    Admin <--> P5
    P5 -->|notes / requirements| D3

    Admin --> P6
    User --> P6
    P6 <--> D3
    P6 -->|status updates| User
```

---

## 3. Activity Diagram (booking flow, swimlanes)

```mermaid
flowchart TD
    subgraph USER[End User]
        Start([Start]) --> Open[Open App]
        Open --> Logged{Logged in?}
        Logged -- No --> Auth[Login via Email / OAuth]
        Auth --> AuthOK{Auth success?}
        AuthOK -- No --> Auth
        AuthOK -- Yes --> Home
        Logged -- Yes --> Home[View Service Catalog]
        Home --> Pick[Select service<br/>Post-op / Pre-op / Elderly care]
        Pick --> Member[Choose self / family member]
        Member --> Slot[Choose time slot e.g. 8am–8pm]
        Slot --> Demands[Add special demands optional]
        Demands --> Submit[Submit booking request]
    end

    subgraph ADMIN[Admin]
        Receive[Receive booking request] --> Review{Demands feasible?}
        Review -- No --> Discuss[Contact user to adjust]
        Discuss --> Review
        Review -- Yes --> Arrange[Arrange nurse OFFLINE<br/>outside system]
        Arrange --> Assign[Record assigned nurse<br/>+ confirm booking]
    end

    Submit --> Receive
    Assign --> Notify[User notified:<br/>booking confirmed]
    Notify --> Track[Track service status]
    Track --> Done([End])
```

---

## 4. Overall Workflow (Sequence diagram)

```mermaid
sequenceDiagram
    actor U as End User
    participant A as App / System
    participant O as OAuth
    actor AD as Admin
    participant N as Nurse (offline)

    U->>A: Open app
    A->>O: Request authentication
    O-->>A: Identity / token
    A-->>U: Logged in + show service catalog

    U->>A: Select service, member, time slot, demands
    A->>A: Save booking request
    A-->>AD: Notify: new booking request

    opt Special demands need clarification
        AD->>A: Message user
        A-->>U: Relay query
        U->>A: Respond
        A-->>AD: Relay response
    end

    AD-->>N: Arrange nurse (OUTSIDE system)
    N-->>AD: Confirms availability (offline)

    AD->>A: Confirm booking + record assigned nurse
    A-->>U: Booking confirmed + nurse details
    A-->>U: Service status updates
```

---

## 5. ER Diagram (database schema)

```mermaid
erDiagram
    USER ||--o{ FAMILY_MEMBER : "manages"
    USER ||--o{ BOOKING : "places"
    FAMILY_MEMBER ||--o{ BOOKING : "is care recipient of"
    SERVICE ||--o{ BOOKING : "is requested in"
    BOOKING ||--o{ BOOKING_NOTE : "has special demands"
    BOOKING ||--o| ASSIGNED_NURSE : "is fulfilled by"

    USER {
        int user_id PK
        string name
        string email
        string oauth_provider
        string oauth_id
        string role "user or admin"
        datetime created_at
    }

    FAMILY_MEMBER {
        int member_id PK
        int user_id FK
        string name
        int age
        string relation
        string medical_notes
    }

    SERVICE {
        int service_id PK
        string name "Post-op / Pre-op / Elderly care"
        string description
        decimal base_price
        boolean active
    }

    BOOKING {
        int booking_id PK
        int user_id FK
        int member_id FK
        int service_id FK
        datetime slot_start "e.g. 8am"
        datetime slot_end "e.g. 8pm"
        string status "requested / confirmed / in_progress / completed / cancelled"
        datetime created_at
    }

    BOOKING_NOTE {
        int note_id PK
        int booking_id FK
        string author "user or admin"
        string message
        datetime created_at
    }

    ASSIGNED_NURSE {
        int assignment_id PK
        int booking_id FK
        string nurse_name "arranged offline by admin"
        string nurse_contact
        datetime assigned_at
    }
```

> **Note on `ASSIGNED_NURSE`:** the nurse is not a system account. This table only records the details the admin enters after arranging the nurse offline, so the user can see who will arrive.
