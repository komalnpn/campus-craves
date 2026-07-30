# CampusCraves 🍽️

A campus food-sharing platform that reduces food waste by letting students post surplus food, reserve items, and complete hand-offs — built as a full-stack software engineering class project by a team of four at NYU Abu Dhabi.

**Stack:** React Native (Expo) · FastAPI · MongoDB · deployed on Render

> This fork adds architecture documentation and repo hygiene on top of the original team repo ([bipana06/campusCraves](https://github.com/bipana06/campusCraves)).

## Why it's interesting

- **A concurrency-safe reservation protocol.** Food items move through a traffic-light lifecycle (`green` available → `yellow` reserved → `red` completed). State transitions use MongoDB *compare-and-set* updates — the update filter includes the expected current status — so two users racing to reserve the same item can't both win.
- **Authorization at the transition level.** Only the user who reserved an item may complete the transaction; the check happens server-side in the update filter, not just in the UI.
- **A moderation subsystem.** Users can report bad actors per post (with server-side "can this user report this post?" checks to prevent duplicates), and admins review reports through a pending → reviewed workflow.
- **Three auth paths** (email/password signup, Google sign-in, NYU NetID registration) converging on one user model.

## System architecture

```mermaid
flowchart LR
    subgraph Client["📱 React Native app (Expo)"]
        UI["Screens<br/>Marketplace · FoodPost · Report<br/>NetID · Notifications"]
        API["apiService.js<br/>(central API client)"]
        UI --> API
    end

    subgraph Server["⚙️ FastAPI (Render)"]
        FR["food router<br/>/api/food"]
        UR["users router<br/>/api/users"]
        RR["reports router<br/>/api/report"]
    end

    subgraph DB["🗄️ MongoDB"]
        FC[("food")]
        UC[("users")]
        RC[("reports")]
    end

    API -->|REST / JSON| FR & UR & RR
    FR --> FC
    UR --> UC
    RR --> RC
    RR -.->|"validates postId"| FC
```

## Domain model

```mermaid
erDiagram
    USER ||--o{ FOODPOST : "posts"
    USER ||--o{ FOODPOST : "reserves"
    USER ||--o{ REPORT : "files"
    FOODPOST ||--o{ REPORT : "is subject of"

    USER {
        string netId
        string email
        string googleId "optional"
        string role "user | admin"
        int postCount
        int reservationCount
    }
    FOODPOST {
        string foodName
        string category
        string dietaryInfo
        string pickupLocation
        datetime expirationTime
        string status "green | yellow | red"
        string reservedBy
        string photo
    }
    REPORT {
        string postId
        string user1ID "reporter"
        string user2ID "reported"
        string message
        string reviewStatus "pending | reviewed"
        string reviewedBy
    }
```

## Reservation lifecycle

```mermaid
stateDiagram-v2
    [*] --> green : POST /api/food
    green --> yellow : POST /api/food/reserve<br/>(atomic CAS on status green)
    yellow --> red : POST /api/food/complete<br/>(atomic CAS, reserver only)
    red --> [*]

    note right of yellow
        Update filter matches
        {_id, status "green"} — a lost
        race returns 400, never a
        double reservation
    end note
```

## Reserve flow, end to end

```mermaid
sequenceDiagram
    actor U as User
    participant App as RN App
    participant F as FastAPI /api/food
    participant M as MongoDB

    U->>App: Tap "Reserve"
    App->>F: POST /reserve {food_id, user}
    F->>M: find_one({_id})
    M-->>F: item (status green)
    F->>M: update_one({_id, status green},<br/>{$set: {status yellow, reservedBy: user}})
    alt update matched
        M-->>F: modified 1
        F-->>App: 200 reserved
        App-->>U: Confirmation
    else raced by another user
        M-->>F: modified 0
        F->>M: find_one({_id}) — re-check
        F-->>App: 400 already reserved
        App-->>U: "No longer available"
    end
```

## API surface

| Area | Endpoint | Purpose |
|---|---|---|
| Food | `POST /api/food` | Create a post (multipart, photo upload) |
| | `GET /api/food` | List marketplace posts |
| | `GET /api/food/search` | Filtered search |
| | `POST /api/food/reserve` | Reserve (green → yellow, atomic) |
| | `POST /api/food/complete` | Complete hand-off (yellow → red, reserver only) |
| | `GET /api/food/poster-netid/{id}` | Look up poster's NetID |
| Users | `POST /api/users/signup` · `/email-login` · `/auth-check` | Email auth |
| | `POST /api/users/register` | Google / NetID registration |
| | `GET /api/users/profile/{net_id}` | Profile with post & pickup history |
| Reports | `POST /api/report` | File a report |
| | `GET /api/report/can-report/{post}/{user}` | Duplicate-report guard |
| | `PUT /api/report/{id}` | Admin review (pending → reviewed) |
| | `GET /api/reports` | Admin list |

## Testing

The backend ships with a pytest suite (`CC-backend/tests/`) covering the food, user, and report routers plus database and utility layers, with fixtures in `conftest.py`.

```bash
cd CC-backend && pip install -r requirements.txt && pytest
```

## Running locally

**Frontend** (Expo):

```bash
cd frontend && npm install && npx expo start
```

**Backend** (FastAPI + MongoDB):

```bash
cd CC-backend
python -m venv campusenv && source campusenv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

Set the MongoDB connection string in `CC-backend/.env` (see `database.py`).

## Team

Built by **Manoj Dhakal**, **Komal Neupane**, **Bipana Bastola**, and **Aabaran Paudel** for a Software Engineering course at NYU Abu Dhabi.
