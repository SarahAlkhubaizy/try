
## Overview

This document defines the high-level system architecture for **Qimmah (قمة)**, a mobile application for discovering officially approved hiking trails across Saudi Arabia.

The architecture follows a standard 3-tier model — Frontend, Backend, and Database — with integration to external third-party services for maps and image hosting.

## Architecture Diagram

```mermaid
flowchart TB
    Roles[User Roles]

    Roles --> actor1[Guest]
    Roles --> actor2[Registered User]
    Roles --> actor3[Admin]

    actor1 --> APP
    actor2 --> APP
    actor3 --> ADMIN

    APP["Mobile App<br/>Flutter<br/>Browse · Search · Filter · Trail Details · Map · My Location · Reviews · Saved · Completed · Profile"]
    ADMIN["Admin Dashboard<br/>Flutter Web<br/>Manage Trails · Upload GeoJSON Routes · Delete Reviews · Suspend Users"]

    GPS["Device Location<br/>geolocator · Updated periodically while app is open"]

    GPS -->|Current Location| APP
    APP -->|REST API| BE
    ADMIN -->|REST API| BE
    APP -->|Route · Start Point · My Location| MAPS

    BE["Backend<br/>Flask (Python) + SQLAlchemy<br/>Auth JWT · Users · Trails · Reviews · Saved · Completed · Admin"]

    BE -->|SQL Queries| DB
    BE -->|Upload Photos| CLOUD

    DB["MySQL<br/>Users · Trails (GeoJSON Routes) · Reviews · Saved Trails · Completed Trails · Token Blocklist"]

    CLOUD["Cloudinary<br/>Trail Photos · CDN"]

    MAPS["Google Maps SDK<br/>Trail Routes · Starting Points · Current Location"]

    CLOUD -->|Image URLs| APP
```

## Component Descriptions

| Component       | Technology            | Role                                                                                                                                                                                                                  |
| --------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mobile App      | Flutter               | Renders the hiker-facing interface in Arabic (right-to-left). Handles browsing, search and filters, the trails map, trail details, saving trails, and reviews. Communicates with the backend via REST API calls.      |
| Admin Dashboard | Flutter (Web)         | Lets the admin add, edit, and deactivate trails, upload trail photos and GeoJSON route files, and moderate reviews. Uses the same backend API with admin-only routes.                                                 |
| Backend         | Flask (Python)        | Processes all business logic: authentication, data validation, filtering, rating calculation, and communication between the frontend and the database. Exposes RESTful API endpoints organized into Flask Blueprints. |
| ORM             | SQLAlchemy            | Maps Python classes to MySQL tables and builds safe, parameterized SQL queries.                                                                                                                                       |
| Database        | MySQL                 | Stores all application data: users, trails, reviews, and saved trails. Uses relational tables with foreign key constraints to enforce data integrity, and the utf8mb4 character set for Arabic text.                  |
| Route Data      | GeoJSON               | Stores each trail's route as a LineString with start and end Points, saved in a MySQL JSON column.                                                                                                                    |
| Auth Layer      | JWT (JSON Web Tokens) | Manages stateless authentication. Issues tokens on login, verifies identity on protected routes. Supports 3 roles: Visitor, Registered User, Admin.                                                                   |
| Image Storage   | Cloudinary            | Stores and serves trail photos. Provides CDN delivery and automatic optimization.                                                                                                                                     |
| Map Service     | Google Maps SDK       | Displays the trails map with markers and draws each trail's GeoJSON route between its start and end points.                                                                                                           |

## Data Flow

Steps describing how data moves through the system, covering the three key use cases defined in the sequence diagrams (Task 3):

### Use Case 1: User Login

| # | Step                                                                                                  |
| - | ----------------------------------------------------------------------------------------------------- |
| 1 | The user enters their email and password in the Flutter app.                                          |
| 2 | The app sends a login request to the Backend (Flask) over HTTPS.                                      |
| 3 | The backend checks the user's credentials against MySQL.                                              |
| 4 | MySQL returns the matching user record to the backend.                                                |
| 5 | The backend verifies the password hash and returns an authentication response (JWT token) to the app. |
| 6 | The app stores the token securely and displays a login success or error message to the user.          |

### Use Case 2: Filter Trails and View Trail Details

| # | Step                                                                                                        |
| - | ----------------------------------------------------------------------------------------------------------- |
| 1 | The user selects a region, difficulty level, and sort option in the app.                                    |
| 2 | The app requests the filtered trails from the Backend.                                                      |
| 3 | The backend queries MySQL for active trails matching the filters.                                           |
| 4 | The backend sends a trails summary list to the app as a JSON response, and the app renders the trail cards. |
| 5 | When the user taps a trail, the app requests its details from the Backend.                                  |
| 6 | The backend returns the trail data, its GeoJSON route, reviews, and rating summary.                         |
| 7 | The app passes the route coordinates to Google Maps SDK, which draws the route on the map.                  |
| 8 | Trail photos are fetched directly from Cloudinary via CDN URLs.                                             |

### Use Case 3: Add a Review

| # | Step                                                                                                           |
| - | -------------------------------------------------------------------------------------------------------------- |
| 1 | The user taps "Add your review" on the trail details screen.                                                   |
| 2 | If the user is not logged in, the app asks them to log in first.                                               |
| 3 | The logged-in user selects a rating (1–5), writes a comment, and the app sends it to the Backend with the JWT. |
| 4 | The backend verifies the token and validates the input.                                                        |
| 5 | The backend saves the review in MySQL and updates the trail's average rating.                                  |
| 6 | The backend confirms the review to the app, which displays it on the trail page.                               |

## Deployment Architecture

| Environment | Description                                                                                                                                                                                       |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Development | Local machines — each developer runs the Flask API and MySQL locally, and runs the Flutter app on an emulator or device.                                                                          |
| Staging     | Pre-production environment used for testing before release.                                                                                                                                       |
| Production  | Deployed on a cloud platform (e.g., Render or Railway for the backend and MySQL). The mobile app is distributed as an APK / test build, and the admin dashboard is deployed as a Flutter web app. |

## Technical Justifications

Every technology in this architecture was chosen based on the team's functional requirements, non-functional requirements, and project constraints.

| Technology      | Decision           | Justification                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Flutter         | Frontend Framework | One codebase builds the Android and iOS app and the web admin dashboard, which suits a small student team. Flutter supports right-to-left layouts for an Arabic-first interface and has an official Google Maps package.                                                                                                                                                                                                                                                                                                                                                                    |
| Flask (Python)  | Backend Framework  | Builds on the team's Python foundation from the Holberton program. Flask is lightweight and well-suited for building RESTful APIs quickly.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| MySQL           | Database           | A relational database was chosen over a non-relational one for Qimmah's core data model.<br><br>MySQL (relational — interconnected tables) enforces strong relationships between users, trails, reviews, and saved trails, and guarantees data integrity (e.g., a review cannot exist without a valid user and trail). Foreign key constraints prevent orphaned or invalid records.<br><br>MongoDB (non-relational — JSON documents) was considered but not selected, as it does not enforce relational integrity by default. MySQL's JSON column type still allows storing GeoJSON routes. |
| GeoJSON         | Route Data Format  | An open standard for geographic data. Routes recorded with GPS tools can be uploaded as files and sent to the app without conversion.                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| JWT             | Authentication     | Stateless authentication eliminates the need for session management on the server. JWT tokens support role-based access control for 3 user types: Visitor, Registered User, Admin.                                                                                                                                                                                                                                                                                                                                                                                                          |
| Google Maps SDK | Map Service        | The map is a core feature of Qimmah. Google Maps provides reliable coverage of Saudi Arabia, custom markers, and route drawing, with an official Flutter package.                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Cloudinary      | Image Hosting      | Provides a free-tier CDN for image storage and delivery, with no credit card required. Eliminates the need to manage file storage infrastructure. Supports automatic image optimization and resizing — essential for large trail photos on mobile.                                                                                                                                                                                                                                                                                                                                          |

## Non-Functional Requirements Addressed

| Requirement     | How the Architecture Addresses It                                                                                                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Performance     | Flutter compiles to native code for smooth scrolling and map interaction. Trail lists return summaries only, and full GeoJSON routes load on the details screen. Cloudinary CDN reduces image load times. |
| Scalability     | MySQL supports indexing and read replicas. The stateless JWT backend allows adding more Flask instances as usage grows.                                                                                   |
| Security        | JWT ensures only authenticated users access protected routes. HTTPS encrypts all client-server communication. Passwords are hashed using Werkzeug, and SQLAlchemy prevents SQL injection.                 |
| Maintainability | Separation of concerns: frontend, backend, and database are fully decoupled. Each layer can be updated independently.                                                                                     |
| Usability       | Arabic-first, right-to-left interface with simple navigation. Guests can browse all trails without an account.                                                                                            |



#2. Components, Classes, and Database Design

## 1. Back-end Classes

### User

**Attributes:**

* `id`
* `name`
* `email`
* `password`
* `role`
* `language`
* `currentLocation`
* `isSuspended`

**Methods:**

* `register()`
* `login()`
* `logout()`
* `updateProfile()`
* `viewFavorites()`
* `addFavorite()`
* `removeFavorite()`
* `viewMyReviews()`
* `markTrailAsCompleted()`
* `viewCompletedTrails()`
* `getCurrentLocation()`

### Trail

**Attributes:**

* `id`
* `name`
* `description`
* `region`
* `difficulty`
* `distance`
* `estimatedDuration`
* `images`
* `startLatitude`
* `startLongitude`
* `endLatitude`
* `endLongitude`
* `routeCoordinates`
* `safetyTips`
* `status`
* `createdAt`

**Methods:**

* `getDetails()`
* `getLocation()`
* `getRoute()`
* `getSafetyTips()`
* `isActive()`

### Review

**Attributes:**

* `id`
* `userId`
* `trailId`
* `rating`
* `comment`
* `createdAt`

**Methods:**

* `addReview()`
* `updateReview()`
* `deleteReview()`
* `getReview()`
* `validateRating()`

### Favorite

**Attributes:**

* `userId`
* `trailId`
* `createdAt`

**Methods:**

* `addFavorite()`
* `removeFavorite()`
* `isFavorite()`
* `getUserFavorites()`

### CompletedTrail

**Attributes:**

* `userId`
* `trailId`
* `completedAt`

**Methods:**

* `markAsCompleted()`
* `removeCompletedTrail()`
* `getCompletedTrails()`
* `isCompleted()`

### TrailService

**Attributes:**

* None

**Methods:**

* `searchByName()`
* `filterByRegion()`
* `filterByDifficulty()`
* `clearFilters()`

### ReviewService

**Attributes:**

* None

**Methods:**

* `getReviews()`
* `calculateAverageRating()`
* `deleteInappropriateReview()`

### Admin

Admin inherits from the `User` class and provides additional administrative functions.

**Methods:**

* `addTrail()`
* `updateTrail()`
* `deleteTrail()`
* `uploadTrailImages()`
* `updateTrailRoute()`
* `suspendUser()`

---

# 2. UML Class Diagram

```mermaid
classDiagram

class User {
    +int id
    +string name
    +string email
    +string password
    +Role role
    +Language language
    +Location currentLocation
    +boolean isSuspended
    +register()
    +login()
    +logout()
    +updateProfile()
    +viewFavorites()
    +addFavorite()
    +removeFavorite()
    +viewMyReviews()
    +markTrailAsCompleted()
    +viewCompletedTrails()
    +getCurrentLocation()
}

class Admin {
    +addTrail()
    +updateTrail()
    +deleteTrail()
    +uploadTrailImages()
    +updateTrailRoute()
    +suspendUser()
}

class Trail {
    +int id
    +string name
    +string description
    +string region
    +Difficulty difficulty
    +decimal distance
    +string estimatedDuration
    +string images
    +decimal startLatitude
    +decimal startLongitude
    +decimal endLatitude
    +decimal endLongitude
    +string routeCoordinates
    +string safetyTips
    +Status status
    +datetime createdAt
    +getDetails()
    +getLocation()
    +getRoute()
    +getSafetyTips()
    +isActive()
}

class Review {
    +int id
    +int userId
    +int trailId
    +int rating
    +string comment
    +datetime createdAt
    +addReview()
    +updateReview()
    +deleteReview()
    +getReview()
    +validateRating()
}

class Favorite {
    +int userId
    +int trailId
    +datetime createdAt
    +addFavorite()
    +removeFavorite()
    +isFavorite()
    +getUserFavorites()
}

class CompletedTrail {
    +int userId
    +int trailId
    +datetime completedAt
    +markAsCompleted()
    +removeCompletedTrail()
    +getCompletedTrails()
    +isCompleted()
}

class TrailService {
    +searchByName()
    +filterByRegion()
    +filterByDifficulty()
    +clearFilters()
}

class ReviewService {
    +getReviews()
    +calculateAverageRating()
    +deleteInappropriateReview()
}

User <|-- Admin

User "1" --> "0..*" Review : writes
Trail "1" --> "0..*" Review : receives

User "1" --> "0..*" Favorite : saves
Trail "1" --> "0..*" Favorite : has

User "1" --> "0..*" CompletedTrail : completes
Trail "1" --> "0..*" CompletedTrail : has

TrailService ..> Trail : searches and filters
ReviewService ..> Review : manages

Admin --> Trail : manages
Admin --> Review : moderates
```

---

# 3. Database Design

The system uses a relational database with the following tables:

### Users

| Field           | Type    | Key    |
| --------------- | ------- | ------ |
| id              | INT     | PK     |
| name            | VARCHAR |        |
| email           | VARCHAR | UNIQUE |
| password        | VARCHAR |        |
| role            | ENUM    |        |
| language        | ENUM    |        |
| currentLocation | VARCHAR |        |
| isSuspended     | BOOLEAN |        |

### Trails

| Field             | Type     | Key |
| ----------------- | -------- | --- |
| id                | INT      | PK  |
| name              | VARCHAR  |     |
| description       | TEXT     |     |
| region            | VARCHAR  |     |
| difficulty        | ENUM     |     |
| distance          | DECIMAL  |     |
| estimatedDuration | VARCHAR  |     |
| images            | TEXT     |     |
| startLatitude     | DECIMAL  |     |
| startLongitude    | DECIMAL  |     |
| endLatitude       | DECIMAL  |     |
| endLongitude      | DECIMAL  |     |
| routeCoordinates  | TEXT     |     |
| safetyTips        | TEXT     |     |
| status            | ENUM     |     |
| createdAt         | DATETIME |     |

### Reviews

| Field     | Type     | Key            |
| --------- | -------- | -------------- |
| id        | INT      | PK             |
| userId    | INT      | FK → Users.id  |
| trailId   | INT      | FK → Trails.id |
| rating    | INT      |                |
| comment   | TEXT     |                |
| createdAt | DATETIME |                |

### Favorites

| Field     | Type     | Key                |
| --------- | -------- | ------------------ |
| userId    | INT      | PK, FK → Users.id  |
| trailId   | INT      | PK, FK → Trails.id |
| createdAt | DATETIME |                    |

### CompletedTrails

| Field       | Type     | Key                |
| ----------- | -------- | ------------------ |
| userId      | INT      | PK, FK → Users.id  |
| trailId     | INT      | PK, FK → Trails.id |
| completedAt | DATETIME |                    |

---

# 4. ER Diagram

```mermaid
erDiagram

    USERS {
        INT id PK
        VARCHAR name
        VARCHAR email UK
        VARCHAR password
        ENUM role
        ENUM language
        VARCHAR currentLocation
        BOOLEAN isSuspended
    }

    TRAILS {
        INT id PK
        VARCHAR name
        TEXT description
        VARCHAR region
        ENUM difficulty
        DECIMAL distance
        VARCHAR estimatedDuration
        TEXT images
        DECIMAL startLatitude
        DECIMAL startLongitude
        DECIMAL endLatitude
        DECIMAL endLongitude
        TEXT routeCoordinates
        TEXT safetyTips
        ENUM status
        DATETIME createdAt
    }

    REVIEWS {
        INT id PK
        INT userId FK
        INT trailId FK
        INT rating
        TEXT comment
        DATETIME createdAt
    }

    FAVORITES {
        INT userId PK, FK
        INT trailId PK, FK
        DATETIME createdAt
    }

    COMPLETED_TRAILS {
        INT userId PK, FK
        INT trailId PK, FK
        DATETIME completedAt
    }

    USERS ||--o{ REVIEWS : writes
    TRAILS ||--o{ REVIEWS : receives

    USERS ||--o{ FAVORITES : saves
    TRAILS ||--o{ FAVORITES : has

    USERS ||--o{ COMPLETED_TRAILS : completes
    TRAILS ||--o{ COMPLETED_TRAILS : has
```

---

# 5. Front-end Components

The main front-end components are:

* **Home / Trail List**

  * Displays available hiking trails.
  * Provides access to search and filters.

* **Search Bar**

  * Searches trails by name.

* **Filter Component**

  * Filters trails by region.
  * Filters trails by difficulty.

* **Saudi Map**

  * Displays hiking trails across Saudi Arabia.
  * Allows users to select a trail marker.
  * Displays the trail starting point.
  * Displays the user's current location with periodic updates.

* **Trail Details**

  * Displays trail description, region, difficulty, distance, duration, images, and safety tips.

* **Trail Route Map**

  * Displays the trail route with start and end points.

* **Authentication**

  * Registration.
  * Login.
  * Logout.
  * Prompts guests to sign up when they try to save, rate, or review a trail.

* **Favorites**

  * Allows registered users to save and remove favorite trails.

* **Reviews & Ratings**

  * Displays reviews and average ratings.
  * Allows registered users to submit ratings and comments.
  * Allows users to edit or delete their own reviews.

* **Completed Trails*

# 3. High-Level Sequence Diagrams
## 1. User Login
<img width="2764" height="2316" alt="User Login" src="https://github.com/user-attachments/assets/c07c8f5b-81d0-4fc1-b239-8c1de3930eb5" />
## 2. browse-search-filter-trails.png
<img width="2936" height="2658" alt="Browse, Search, and Filter Trails (Guest or Registered User)" src="https://github.com/user-attachments/assets/21e4106f-e241-4fd3-8ed2-a3be59894b4f" />
## 3. current-location-trail-map.png
<img width="3165" height="2384" alt="View Current Location on the Trail Map" src="https://github.com/user-attachments/assets/58f7991e-6a1d-421b-ba10-8328bd77eb65" />
## 4. rate-review-trail.png
<img width="3164" height="2402" alt="Rate and Review a Trail" src="https://github.com/user-attachments/assets/239e3202-028c-4837-804a-d609d0022d17" />








# 4. API Specifications

## External APIs

| API | Used By | Purpose | Why It Was Chosen |
|---|---|---|---|
| Google Maps SDK for Android / iOS (via `google_maps_flutter`) | Flutter mobile app | Displays trails on the map, draws each trail's GeoJSON route with its starting point, and shows the user's current location (My Location layer). | Reliable map coverage of Saudi Arabia, custom markers, route lines, and a built-in current-location layer. Has an official Flutter package, and its monthly free usage covers the MVP. |
| Google Maps JavaScript API (via `google_maps_flutter_web`) | Admin dashboard (Flutter Web) | Previews an uploaded GeoJSON route on the map before the admin saves a trail. | Same map provider as the mobile app, so routes look the same in both. |
| Cloudinary Upload API (via `cloudinary` Python SDK) | Flask backend | Uploads trail photos and returns their URLs. Photos are served to the app via Cloudinary's CDN. | Free plan with no credit card required. Images are not lost when the server redeploys. Automatic resizing and CDN delivery make photos load fast on mobile. |

> **Device services (not external APIs):** the user's current location is read on the device with the `geolocator` Flutter package, only with the user's permission and only while the app is open. It is never sent to the server.
>
> **API keys** are never shared publicly: the Google Maps key is restricted to Qimmah's app, and Cloudinary keys are stored only on the server as environment variables.

## Internal API (Flask REST API)

### General Rules

- **Base URL:** `/api`
- **Format:** All requests and responses use JSON, except admin trail uploads, which use `multipart/form-data` (to send files).
- **Authentication:** Protected endpoints require a JWT in the header: `Authorization: Bearer <token>`
- **Access levels:** 🌐 Guest (no account) · 🔒 Registered user · 🛠️ Admin only
- **Error format:** `{ "error": "Error message" }`

### Endpoints Overview

| # | Method | URL Path | Access | Description | User Story |
|---|---|---|---|---|---|
| 1 | POST | `/api/auth/register` | 🌐 | Create a new account | 1 |
| 2 | POST | `/api/auth/login` | 🌐 | Log in and receive a JWT | 2 |
| 3 | POST | `/api/auth/logout` | 🔒 | Log out and revoke the token | 2 |
| 4 | GET | `/api/users/me` | 🔒 | Get the user's account information | 17 |
| 5 | PUT | `/api/users/me` | 🔒 | Edit account information | 17 |
| 6 | GET | `/api/trails` | 🌐 | List trails with filters and search | 3, 4, 5, 8 |
| 7 | GET | `/api/trails/{id}` | 🌐 | Get trail details, GeoJSON route, starting point, and safety tips | 6, 7, 18, 22 |
| 8 | GET | `/api/trails/{id}/reviews` | 🌐 | List a trail's reviews | 9 |
| 9 | POST | `/api/trails/{id}/reviews` | 🔒 | Rate and review a trail | 12 |
| 10 | PUT | `/api/reviews/{id}` | 🔒 | Edit own review | 13 |
| 11 | DELETE | `/api/reviews/{id}` | 🔒 / 🛠️ | Delete own review, or any review (admin) | 13, 15 |
| 12 | GET | `/api/users/me/saved-trails` | 🔒 | List saved trails | 11 |
| 13 | POST | `/api/users/me/saved-trails` | 🔒 | Save a trail | 11 |
| 14 | DELETE | `/api/users/me/saved-trails/{trail_id}` | 🔒 | Remove a saved trail | 11 |
| 15 | GET | `/api/users/me/completed-trails` | 🔒 | List completed trails | 16 |
| 16 | POST | `/api/users/me/completed-trails` | 🔒 | Mark a trail as completed | 16 |
| 17 | DELETE | `/api/users/me/completed-trails/{trail_id}` | 🔒 | Unmark a completed trail | 16 |
| 18 | POST | `/api/admin/trails` | 🛠️ | Add a new trail | 14 |
| 19 | PUT | `/api/admin/trails/{id}` | 🛠️ | Edit a trail | 14 |
| 20 | DELETE | `/api/admin/trails/{id}` | 🛠️ | Delete a trail | 14 |
| 21 | GET | `/api/admin/reviews` | 🛠️ | List all reviews for moderation | 15 |
| 22 | GET | `/api/admin/users` | 🛠️ | List users | 21 |
| 23 | PATCH | `/api/admin/users/{id}/status` | 🛠️ | Suspend or reactivate a user | 21 |

> **Handled in the app, not the API:**
> - **Story 10 (guest sign-up prompt):** the app shows the prompt when a guest tries an action that needs a 🔒 endpoint.
> - **Story 19 (share a trail):** the app shares the trail's link through the device's share menu (`share_plus`).
> - **Story 20 (switch language):** the app switches its interface with Flutter localization.
> - **Story 22 (current location):** the app reads the location on the device and draws it on the map over the route from endpoint 7.

---

### Auth

#### 1. POST `/api/auth/register` 🌐

**Input (JSON):**
```json
{
  "name": "Sara",
  "email": "sara@example.com",
  "password": "StrongPass123"
}
```

**Output — 201 Created:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": 12, "name": "Sara", "email": "sara@example.com", "role": "user" }
}
```

**Errors:** `400` missing or invalid fields · `409` email already registered

#### 2. POST `/api/auth/login` 🌐

**Input (JSON):**
```json
{
  "email": "sara@example.com",
  "password": "StrongPass123"
}
```

**Output — 200 OK:** Same structure as register.

**Errors:** `401` invalid email or password · `403` account suspended

#### 3. POST `/api/auth/logout` 🔒

**Input:** JWT in header only.

**Output — 200 OK:**
```json
{ "message": "Logged out successfully" }
```

> The token is added to a blocklist on the server so it can no longer be used, and the app deletes it from secure storage.

---

### User Account

#### 4. GET `/api/users/me` 🔒

**Output — 200 OK:**
```json
{
  "id": 12,
  "name": "Sara",
  "email": "sara@example.com",
  "role": "user",
  "saved_trails_count": 4,
  "completed_trails_count": 2,
  "reviews_count": 3
}
```

#### 5. PUT `/api/users/me` 🔒

**Input (JSON):** only the fields being changed. Changing the password requires the current password.
```json
{
  "name": "Sara A.",
  "current_password": "StrongPass123",
  "new_password": "NewStrongPass456"
}
```

**Output — 200 OK:** The updated user object.

**Errors:** `400` invalid fields · `401` wrong current password · `409` email already used

---

### Trails

#### 6. GET `/api/trails` 🌐

**Input (query parameters, all optional):**

| Parameter | Type | Example | Description |
|---|---|---|---|
| `region` | string | `asir` | Filter by region |
| `difficulty` | string | `easy` · `moderate` · `hard` | Filter by difficulty |
| `search` | string | `السودة` | Search by trail name |
| `page` | integer | `1` | Page number |
| `limit` | integer | `20` | Results per page |

**Example:** `GET /api/trails?region=asir&difficulty=moderate`

**Output — 200 OK** (short summary only, no full route):
```json
{
  "trails": [
    {
      "id": 7,
      "name": "مسار جبل السودة",
      "region": "asir",
      "difficulty": "moderate",
      "distance_km": 8,
      "duration": "3–4 ساعات",
      "avg_rating": 4.8,
      "reviews_count": 125,
      "cover_image": "https://res.cloudinary.com/.../soudah.jpg",
      "start_point": { "lat": 18.2721, "lng": 42.3684 }
    }
  ],
  "page": 1,
  "total": 1
}
```

#### 7. GET `/api/trails/{id}` 🌐

**Input:** Trail ID in the URL path.

**Output — 200 OK:**
```json
{
  "id": 7,
  "name": "مسار جبل السودة",
  "region": "asir",
  "difficulty": "moderate",
  "distance_km": 8,
  "duration": "3–4 ساعات",
  "description": "مسار جبلي يتميز بإطلالاته الطبيعية...",
  "safety_tips": ["احمل كمية كافية من الماء", "ارتدِ حذاءً مناسبًا للمشي"],
  "images": [
    "https://res.cloudinary.com/.../soudah-1.jpg",
    "https://res.cloudinary.com/.../soudah-2.jpg"
  ],
  "start_point": { "lat": 18.2721, "lng": 42.3684 },
  "route_geojson": {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "properties": { "name": "route" },
        "geometry": {
          "type": "LineString",
          "coordinates": [[42.3684, 18.2721], [42.3701, 18.2745], [42.3730, 18.2790]]
        }
      },
      {
        "type": "Feature",
        "properties": { "name": "start" },
        "geometry": { "type": "Point", "coordinates": [42.3684, 18.2721] }
      }
    ]
  },
  "avg_rating": 4.8,
  "reviews_count": 125,
  "share_url": "https://qimmah.app/trails/7",
  "is_saved": false,
  "is_completed": false
}
```

> GeoJSON coordinates are in `[longitude, latitude]` order. `is_saved` and `is_completed` are returned only when a valid JWT is sent. The app draws `route_geojson` on the map and shows the user's current location over it (story 22).

**Errors:** `404` trail not found

---

### Reviews

#### 8. GET `/api/trails/{id}/reviews` 🌐

**Input:** Trail ID in the path · optional query `page`, `limit`.

**Output — 200 OK:**
```json
{
  "reviews": [
    {
      "id": 31,
      "user": { "id": 12, "name": "Sara" },
      "rating": 5,
      "comment": "مسار رائع وإطلالات جميلة",
      "created_at": "2026-10-01T09:30:00Z",
      "updated_at": null
    }
  ],
  "page": 1,
  "total": 125
}
```

#### 9. POST `/api/trails/{id}/reviews` 🔒

**Input (JSON):**
```json
{
  "rating": 5,
  "comment": "مسار رائع وإطلالات جميلة"
}
```

**Output — 201 Created:**
```json
{
  "id": 31,
  "trail_id": 7,
  "rating": 5,
  "comment": "مسار رائع وإطلالات جميلة",
  "created_at": "2026-10-01T09:30:00Z",
  "trail_avg_rating": 4.8
}
```

**Errors:** `400` rating not 1–5 or empty comment · `401` not logged in · `404` trail not found · `409` already reviewed this trail

#### 10. PUT `/api/reviews/{id}` 🔒

**Input (JSON):** the fields being changed.
```json
{
  "rating": 4,
  "comment": "مسار جميل لكنه مزدحم في الإجازات"
}
```

**Output — 200 OK:** The updated review, with the trail's new average rating.

**Errors:** `400` invalid input · `403` not the review's owner · `404` review not found

#### 11. DELETE `/api/reviews/{id}` 🔒 / 🛠️

**Output — 204 No Content**

> A user can delete only their own reviews; an admin can delete any review. The trail's average rating is recalculated after deletion.

**Errors:** `403` not the owner and not an admin · `404` review not found

---

### Saved Trails

#### 12. GET `/api/users/me/saved-trails` 🔒

**Output — 200 OK:** a list of trails in the same summary structure as endpoint 6.

#### 13. POST `/api/users/me/saved-trails` 🔒

**Input (JSON):**
```json
{ "trail_id": 7 }
```

**Output — 201 Created:**
```json
{ "message": "Trail saved", "trail_id": 7 }
```

**Errors:** `404` trail not found · `409` already saved

#### 14. DELETE `/api/users/me/saved-trails/{trail_id}` 🔒

**Output — 204 No Content**

**Errors:** `404` trail not in saved list

---

### Completed Trails

#### 15. GET `/api/users/me/completed-trails` 🔒

**Output — 200 OK:** a list of trails in the same summary structure as endpoint 6, each with a `completed_at` date.

#### 16. POST `/api/users/me/completed-trails` 🔒

**Input (JSON):**
```json
{ "trail_id": 7 }
```

**Output — 201 Created:**
```json
{ "message": "Trail marked as completed", "trail_id": 7, "completed_at": "2026-10-01T15:00:00Z" }
```

**Errors:** `404` trail not found · `409` already marked as completed

#### 17. DELETE `/api/users/me/completed-trails/{trail_id}` 🔒

**Output — 204 No Content**

---

### Admin

#### 18. POST `/api/admin/trails` 🛠️

**Input (`multipart/form-data`):**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | text | ✅ | Trail name |
| `region` | text | ✅ | Region code |
| `difficulty` | text | ✅ | `easy` · `moderate` · `hard` |
| `distance_km` | number | ✅ | Distance in km |
| `duration` | text | ✅ | Estimated duration |
| `description` | text | ✅ | Trail description |
| `safety_tips` | text (JSON array) | ❌ | Safety tips |
| `route_file` | file (.geojson) | ✅ | Route as a LineString, with the starting point as a Point |
| `images` | file(s) | ✅ | Trail photos (uploaded to Cloudinary) |

**Output — 201 Created:**
```json
{
  "id": 8,
  "name": "مسار وادي لجب",
  "images": ["https://res.cloudinary.com/.../lajab-1.jpg"],
  "message": "Trail created"
}
```

**Errors:** `400` invalid data or GeoJSON · `401` not logged in · `403` not an admin

#### 19. PUT `/api/admin/trails/{id}` 🛠️

**Input (`multipart/form-data`):** same fields as endpoint 18; only the changed fields are sent.

**Output — 200 OK:** The updated trail object.

**Errors:** `400` · `401` · `403` · `404` trail not found

#### 20. DELETE `/api/admin/trails/{id}` 🛠️

**Output — 204 No Content**

> Deleting a trail also deletes its reviews, saved entries, and completed entries, and removes its photos from Cloudinary.

#### 21. GET `/api/admin/reviews` 🛠️

**Input (query, optional):** `trail_id`, `page`, `limit`

**Output — 200 OK:**
```json
{
  "reviews": [
    {
      "id": 31,
      "trail": { "id": 7, "name": "مسار جبل السودة" },
      "user": { "id": 12, "name": "Sara" },
      "rating": 5,
      "comment": "مسار رائع وإطلالات جميلة",
      "created_at": "2026-10-01T09:30:00Z"
    }
  ],
  "page": 1,
  "total": 340
}
```

> To delete a review, the admin uses endpoint 11.

#### 22. GET `/api/admin/users` 🛠️

**Input (query, optional):** `search`, `page`, `limit`

**Output — 200 OK:**
```json
{
  "users": [
    { "id": 12, "name": "Sara", "email": "sara@example.com", "reviews_count": 3, "is_suspended": false }
  ],
  "page": 1,
  "total": 1
}
```

#### 23. PATCH `/api/admin/users/{id}/status` 🛠️

**Input (JSON):**
```json
{ "is_suspended": true }
```

**Output — 200 OK:**
```json
{ "id": 12, "is_suspended": true }
```

> A suspended user cannot log in, and their existing tokens are rejected.

**Errors:** `403` not an admin · `404` user not found

---

### HTTP Status Codes Used

| Code | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 201 Created | New resource created |
| 204 No Content | Deleted successfully, no body |
| 400 Bad Request | Invalid input |
| 401 Unauthorized | Missing, invalid, or expired token |
| 403 Forbidden | Not allowed (not the owner, not an admin, or account suspended) |
| 404 Not Found | Resource not found |
| 409 Conflict | Duplicate (email exists, or trail already saved, completed, or reviewed) |



# 5. SCM and QA Strategies

## 5.1 SCM Strategy

### Version Control and Repository Structure

We will use **Git** for version control and **GitHub** to host the code. The project uses a single repository (monorepo) that contains both the frontend and the backend, along with the technical documentation:

```
qimmah/
├── frontend/     (Flutter app)
├── backend/      (Flask API)
├── docs/         (Technical documentation and diagrams)
└── README.md
```

A single repository keeps the project simple for a team of four and makes it easier to review all changes, documentation, and issues in one place. Features that affect both the frontend and the backend, such as OTP verification or reviews, can be handled in a single pull request. Responsibilities between frontend and backend remain clearly separated through folders and feature branches. Tasks and user stories are tracked using **GitHub Issues** and **GitHub Projects**.

### Branching Strategy

We will follow a simple branching model:

- **main**: contains stable, tested, production-ready code only. It is protected, and no one pushes to it directly. Code reaches main only by merging from develop after testing.
- **develop**: the integration branch where all completed features are merged and tested together. The staging environment is deployed from this branch.
- **feature/\***: a separate branch for each task or user story, created from develop and merged back into it through a pull request. Branch names describe the feature, for example `feature/trail-filter`, `feature/otp-verification`, and `feature/reviews`.
- **fix/\***: a short branch for fixing a bug found during testing, created from develop and merged back into it.

Workflow: a team member creates a feature branch from develop, works on the task, opens a pull request into develop, and merges it after review. At the end of each milestone, develop is tested on staging and then merged into main.

### Commits

- Team members commit small, focused changes regularly and push their work at least once a day while working on a task.
- Commit messages follow a consistent format with a prefix that describes the type of change: `feat:` for new features, `fix:` for bug fixes, `docs:` for documentation, `test:` for tests, and `refactor:` for code improvements. Example: `feat: add region filter to trails endpoint`.
- Sensitive data such as database passwords and email credentials are never committed. They are stored in a `.env` file that is listed in `.gitignore`.

### Pull Requests

- Every feature or fix branch is merged into develop through a pull request.
- Each pull request includes a short description of the change and a link to the related user story or issue.
- All automated checks (code style and tests) must pass before the pull request can be merged.

### Code Reviews

- Our team has four members, divided between frontend and backend.
- Each pull request requires approval from at least one other team member before merging.
- Backend changes are reviewed by a backend team member, and frontend changes are reviewed by a frontend team member, since they are most familiar with that part of the code.
- Pull requests that change both frontend and backend require one approval from each side.
- The reviewer checks that the code works as expected, follows the agreed style, includes tests where needed, and does not break existing features.

## 5.2 QA Strategy

### Testing Strategy

Our testing strategy focuses automated tests on the backend, where the core logic, security, and data handling are located, and uses structured manual testing for the frontend, since the user interface changes frequently during MVP development.

### Backend Testing

- **Unit tests** using **pytest** to test individual functions and endpoints, such as password hashing, OTP generation and verification, input validation for reviews, and average rating calculation.
- **Integration tests** using **pytest** with a separate test database, to check that the API and the database work together correctly (for example, creating a review and confirming it is saved and the trail's average rating is updated).
- **Permission tests** to confirm each role can only do what it is allowed to: a guest cannot post reviews or save trails, a registered user can only edit or delete their own reviews, and only an admin can manage trails and suspend users.
- **API testing** using **Postman**, with a shared collection of requests for all endpoints, including valid and invalid inputs.

### Frontend Testing

Manual testing on real Android and iOS devices using a test checklist of critical user flows:

- Sign up with email OTP verification, log in, and log out
- Browse, search, and filter trails as a guest
- View trail details, the map, and safety tips
- Guest is prompted to sign up when trying to review or save a trail
- Add, edit, and delete a review as a registered user
- Admin adds, edits, and deletes trails and removes reviews
- Current location appears on the trail map, and the app works correctly when location permission is denied
- Switching between Arabic and English, and between dark and light mode

### Testing and Quality Tools

- **pytest**: unit and integration tests for the Flask backend
- **Postman**: API testing
- **flake8**: code style checks for Python
- **flutter analyze**: code quality checks for Dart
- **GitHub Actions**: runs code style checks and tests automatically on every pull request
- **GitHub Issues**: reporting and tracking bugs with a "bug" label and a priority level; bugs are fixed in a `fix/*` branch

### Deployment Pipeline

We will use **Docker** to package the backend, so that it runs the same way on every team member's machine and in every environment, and **Railway** to host the backend and the MySQL database. The Flutter mobile app is not containerized or hosted, as it is built and installed directly on devices.

Our pipeline has three environments:

- **Development**: each team member runs the Flask API and MySQL locally using **Docker Compose** (`docker compose up`), which gives everyone the same versions and settings.
- **Staging**: a Railway environment connected to the **develop** branch, with its own MySQL database containing test data. When a pull request is merged into develop, GitHub Actions runs code style checks and all backend tests. If they pass, Railway automatically builds the backend from its Dockerfile and deploys it to staging. A test version of the app (APK) is then built and connected to the staging API, so the team can test it on real devices using the manual test checklist.
- **Production**: a separate Railway environment connected to the **main** branch, with its own production MySQL database. After the staging version passes manual testing, develop is merged into main, and Railway automatically deploys the same tested code to production.

Keeping staging and production as separate Railway environments with separate databases means that testing never affects real user data.

### Quality Goals

- All Must Have user stories have automated tests for their backend endpoints.
- No pull request is merged with failing tests.
- Critical bugs found during testing are fixed before merging develop into main.
