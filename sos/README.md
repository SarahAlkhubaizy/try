# 4. API Specifications

## External APIs

| API | Used By | Purpose | Why It Was Chosen |
|---|---|---|---|
| Google Maps SDK for Android / iOS (via `google_maps_flutter`) | Flutter app (hiker screens and admin panel) | Displays trails on the map, draws each trail's GeoJSON route with its start and end points, and shows the user's current location (My Location layer). | Reliable map coverage of Saudi Arabia, custom markers, route lines, and a built-in current-location layer. Has an official Flutter package, and its monthly free usage covers the MVP. |
| Cloudinary Upload API (via `cloudinary` Python SDK) | Flask backend | Uploads trail photos and returns their URLs. Photos are served to the app via Cloudinary's CDN. | Free plan with no credit card required. Images are not lost when the server redeploys. Automatic resizing and CDN delivery make photos load fast on mobile. |
| Email Service (SMTP via `Flask-Mail`) | Flask backend | Sends a one-time verification code (OTP) to the user's email during sign-up. | Confirms that each account uses a real email address the user owns, which reduces fake accounts. Flask-Mail integrates directly with Flask and works with standard SMTP email providers. |

> **Device and app packages (not external APIs):**
> - `geolocator` reads the user's current location on the device, only with permission and only while the app is open. It is never sent to the server.
> - `pinput` displays the 6-digit OTP input field on the email verification screen.
>
> **Data format:** trail routes are stored as **GeoJSON** in the `routeCoordinates` JSON column. Each route is a `FeatureCollection` with the trail path as a `LineString`. Coordinates follow the GeoJSON order `[longitude, latitude]`.
>
> **API keys** are never shared publicly: the Google Maps key is restricted to Qimmah's app, and Cloudinary and email credentials are stored only on the server as environment variables.

## Internal API (Flask REST API)

### General Rules

- **Base URL:** `https://<qimmah-backend>.up.railway.app/api` (hosted on Railway)
- **Format:** All requests and responses use JSON, except trail creation and image uploads, which use `multipart/form-data` (to send files).
- **Authentication:** Protected endpoints require a JWT in the header: `Authorization: Bearer <token>`. Tokens expire after 24 hours.
- **Access levels:** 🌐 Guest (no account) · 🔒 Registered user · 🛠️ Admin only
- **Admin access:** the admin uses the same Flutter app; admin screens appear only for accounts with the `admin` role, and every 🛠️ endpoint checks this role on the server.
- **Common errors:** every 🔒 and 🛠️ endpoint returns `401` if the token is missing or expired, `403` if the account is suspended, and every 🛠️ endpoint returns `403` if the user is not an admin.
- **Error format:** `{ "error": "Error message" }`

### Endpoints Overview

| # | Module | Method | URL Path | Access | Class Method (Task 2) | User Story |
|---|---|---|---|---|---|---|
| 1 | Auth | POST | `/api/auth/register` | 🌐 | `User.register()` | 1 |
| 2 | Auth | POST | `/api/auth/verify-email` | 🌐 | `User.verifyEmail()` | 1 |
| 3 | Auth | POST | `/api/auth/resend-code` | 🌐 | `User.resendCode()` | 1 |
| 4 | Auth | POST | `/api/auth/login` | 🌐 | `User.login()` | 2 |
| 5 | Users | GET | `/api/users/me` | 🔒 | User attributes | 17 |
| 6 | Users | PUT | `/api/users/me` | 🔒 | `User.updateProfile()` | 17 |
| 7 | Users | GET | `/api/users/me/reviews` | 🔒 | `User.viewMyReviews()` | 13 |
| 8 | Users | GET | `/api/users/me/favorites` | 🔒 | `User.viewFavorites()` | 11 |
| 9 | Users | POST | `/api/users/me/favorites` | 🔒 | `User.addFavorite()` | 11 |
| 10 | Users | DELETE | `/api/users/me/favorites/{trail_id}` | 🔒 | `User.removeFavorite()` | 11 |
| 11 | Users | GET | `/api/users/me/completed-trails` | 🔒 | `User.viewCompletedTrails()` | 16 |
| 12 | Users | POST | `/api/users/me/completed-trails` | 🔒 | `User.markTrailAsCompleted()` | 16 |
| 13 | Users | DELETE | `/api/users/me/completed-trails/{trail_id}` | 🔒 | `CompletedTrail.removeCompletedTrail()` | 16 |
| 14 | Trails | GET | `/api/trails` | 🌐 | `TrailService.searchByName()` · `filterByRegion()` · `filterByDifficulty()` | 3, 4, 5, 8 |
| 15 | Trails | GET | `/api/trails/{id}` | 🌐 | `Trail.getDetails()` · `getRoute()` · `getSafetyTips()` | 6, 7, 18, 22 |
| 16 | Reviews | GET | `/api/trails/{id}/reviews` | 🌐 | `ReviewService.getReviews()` · `calculateAverageRating()` | 9 |
| 17 | Reviews | POST | `/api/trails/{id}/reviews` | 🔒 | `Review.addReview()` · `validateRating()` | 12 |
| 18 | Reviews | PUT | `/api/reviews/{id}` | 🔒 | `Review.updateReview()` | 13 |
| 19 | Reviews | DELETE | `/api/reviews/{id}` | 🔒 | `Review.deleteReview()` | 13 |
| 20 | Admin | POST | `/api/admin/trails` | 🛠️ | `Admin.addTrail()` | 14 |
| 21 | Admin | PUT | `/api/admin/trails/{id}` | 🛠️ | `Admin.updateTrail()` | 14 |
| 22 | Admin | DELETE | `/api/admin/trails/{id}` | 🛠️ | `Admin.deleteTrail()` | 14 |
| 23 | Admin | POST | `/api/admin/trails/{id}/images` | 🛠️ | `Admin.uploadTrailImages()` | 14 |
| 24 | Admin | PUT | `/api/admin/trails/{id}/route` | 🛠️ | `Admin.updateTrailRoute()` | 14 |
| 25 | Admin | DELETE | `/api/admin/reviews/{id}` | 🛠️ | `ReviewService.deleteInappropriateReview()` | 15 |
| 26 | Admin | PATCH | `/api/admin/users/{id}/suspend` | 🛠️ | `Admin.suspendUser()` | 21 |

> **Handled in the app, not the API:**
> - **Story 2 (log out) — `User.logout()`:** the app deletes the JWT from secure storage, and the token also expires on its own after 24 hours.
> - **Story 10 (guest sign-up prompt):** the app shows the prompt when a guest tries an action that needs a 🔒 endpoint.
> - **Story 19 (share a trail):** the app shares the trail's details through the device's share menu (`share_plus`).
> - **Story 22 (current location):** the app reads the location on the device and draws it on the map over the route from endpoint 15.

---

### Auth Module

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
  "message": "Verification code sent to your email",
  "email": "sara@example.com"
}
```

> The account is created with `isVerified = false`. A 6-digit code is emailed to the user, stored as a hash in `otpCodeHash`, and expires after 10 minutes (`otpExpiresAt`). No token is returned until the email is verified.

**Errors:** `400` missing or invalid fields · `409` email already registered

#### 2. POST `/api/auth/verify-email` 🌐

**Input (JSON):** the code the user enters in the `pinput` field.
```json
{
  "email": "sara@example.com",
  "code": "482915"
}
```

**Output — 200 OK:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": 12, "name": "Sara", "email": "sara@example.com", "role": "user" }
}
```

> On success, `isVerified` becomes `true`. Each wrong code increases `otpAttempts`; after 5 wrong attempts, the user must request a new code.

**Errors:** `400` wrong or expired code · `404` account not found · `429` too many wrong attempts

#### 3. POST `/api/auth/resend-code` 🌐

**Input (JSON):**
```json
{ "email": "sara@example.com" }
```

**Output — 200 OK:**
```json
{ "message": "A new verification code has been sent" }
```

> A new code replaces the old one and resets `otpAttempts`. Requests are limited to once per minute.

**Errors:** `404` account not found · `409` email already verified · `429` requested too soon

#### 4. POST `/api/auth/login` 🌐

**Input (JSON):**
```json
{
  "email": "sara@example.com",
  "password": "StrongPass123"
}
```

**Output — 200 OK:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": 12, "name": "Sara", "email": "sara@example.com", "role": "user" }
}
```

> The `role` field tells the app whether to show the admin panel.

**Errors:** `401` invalid email or password · `403` email not verified, or account suspended (`isSuspended`)

---

### Users Module

#### 5. GET `/api/users/me` 🔒

**Output — 200 OK:**
```json
{
  "id": 12,
  "name": "Sara",
  "email": "sara@example.com",
  "role": "user"
}
```

#### 6. PUT `/api/users/me` 🔒

**Input (JSON):** only the fields being changed.
```json
{
  "name": "Sara A.",
  "email": "sara.a@example.com"
}
```

**Output — 200 OK:** The updated user object.

> If the email is changed, `isVerified` becomes `false` and a new verification code is sent to the new email. The user verifies it with endpoint 2.

**Errors:** `400` invalid fields · `409` email already used

#### 7. GET `/api/users/me/reviews` 🔒

**Output — 200 OK:**
```json
{
  "reviews": [
    {
      "id": 31,
      "trail": { "id": 7, "name": "مسار جبل السودة" },
      "rating": 5,
      "comment": "مسار رائع وإطلالات جميلة",
      "created_at": "2026-10-01T09:30:00Z"
    }
  ]
}
```

#### 8. GET `/api/users/me/favorites` 🔒

**Output — 200 OK:** a list of favorite trails in the same summary structure as endpoint 14.

#### 9. POST `/api/users/me/favorites` 🔒

**Input (JSON):**
```json
{ "trail_id": 7 }
```

**Output — 201 Created:**
```json
{ "message": "Trail added to favorites", "trail_id": 7, "created_at": "2026-10-01T12:00:00Z" }
```

**Errors:** `404` trail not found · `409` already in favorites

#### 10. DELETE `/api/users/me/favorites/{trail_id}` 🔒

**Output — 204 No Content**

**Errors:** `404` trail not in favorites

#### 11. GET `/api/users/me/completed-trails` 🔒

**Output — 200 OK:** a list of trails in the same summary structure as endpoint 14, each with a `completed_at` date.

#### 12. POST `/api/users/me/completed-trails` 🔒

**Input (JSON):**
```json
{ "trail_id": 7 }
```

**Output — 201 Created:**
```json
{ "message": "Trail marked as completed", "trail_id": 7, "completed_at": "2026-10-01T15:00:00Z" }
```

**Errors:** `404` trail not found · `409` already marked as completed

#### 13. DELETE `/api/users/me/completed-trails/{trail_id}` 🔒

**Output — 204 No Content**

**Errors:** `404` trail not in completed list

---

### Trails Module

#### 14. GET `/api/trails` 🌐

**Input (query parameters, all optional):**

| Parameter | Type | Example | Class Method |
|---|---|---|---|
| `search` | string | `السودة` | `TrailService.searchByName()` |
| `region` | string | `asir` | `TrailService.filterByRegion()` |
| `difficulty` | string | `easy` · `moderate` · `hard` | `TrailService.filterByDifficulty()` |
| `page` | integer | `1` | — |
| `limit` | integer | `20` | — |

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
      "distance": 8,
      "estimated_duration": "3–4 ساعات",
      "average_rating": 4.8,
      "cover_image": "https://res.cloudinary.com/.../soudah.jpg",
      "start_point": { "lat": 18.2721, "lng": 42.3684 }
    }
  ],
  "page": 1,
  "total": 1
}
```

> `average_rating` is calculated with `ReviewService.calculateAverageRating()`.

#### 15. GET `/api/trails/{id}` 🌐

**Input:** Trail ID in the URL path.

**Output — 200 OK:**
```json
{
  "id": 7,
  "name": "مسار جبل السودة",
  "description": "مسار جبلي يتميز بإطلالاته الطبيعية...",
  "region": "asir",
  "difficulty": "moderate",
  "distance": 8,
  "estimated_duration": "3–4 ساعات",
  "images": [
    "https://res.cloudinary.com/.../soudah-1.jpg",
    "https://res.cloudinary.com/.../soudah-2.jpg"
  ],
  "start_point": { "lat": 18.2721, "lng": 42.3684 },
  "end_point": { "lat": 18.2790, "lng": 42.3730 },
  "route_coordinates": {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "properties": { "name": "route" },
        "geometry": {
          "type": "LineString",
          "coordinates": [[42.3684, 18.2721], [42.3701, 18.2745], [42.3730, 18.2790]]
        }
      }
    ]
  },
  "safety_tips": "احمل كمية كافية من الماء، وارتدِ حذاءً مناسبًا للمشي",
  "average_rating": 4.8,
  "created_at": "2026-09-20T10:00:00Z",
  "is_favorite": false,
  "is_completed": false
}
```

> `is_favorite` and `is_completed` are returned only when a valid JWT is sent. The app draws `route_coordinates` with the start and end points on the map, and shows the user's current location over it (story 22).

**Errors:** `404` trail not found

---

### Reviews Module

#### 16. GET `/api/trails/{id}/reviews` 🌐

**Input:** Trail ID in the path · optional query `page`, `limit`.

**Output — 200 OK:**
```json
{
  "average_rating": 4.8,
  "reviews": [
    {
      "id": 31,
      "user": { "id": 12, "name": "Sara" },
      "rating": 5,
      "comment": "مسار رائع وإطلالات جميلة",
      "created_at": "2026-10-01T09:30:00Z"
    }
  ],
  "page": 1,
  "total": 125
}
```

#### 17. POST `/api/trails/{id}/reviews` 🔒

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
  "created_at": "2026-10-01T09:30:00Z"
}
```

> `Review.validateRating()` checks that the rating is between 1 and 5.

**Errors:** `400` rating not 1–5 or empty comment · `404` trail not found

#### 18. PUT `/api/reviews/{id}` 🔒

**Input (JSON):** the fields being changed.
```json
{
  "rating": 4,
  "comment": "مسار جميل لكنه مزدحم في الإجازات"
}
```

**Output — 200 OK:** The updated review.

**Errors:** `400` invalid input · `403` not the review's owner · `404` review not found

#### 19. DELETE `/api/reviews/{id}` 🔒

**Output — 204 No Content**

> A user can delete only their own reviews. Admins delete inappropriate reviews with endpoint 25.

**Errors:** `403` not the review's owner · `404` review not found

---

### Admin Module

#### 20. POST `/api/admin/trails` 🛠️

**Input (`multipart/form-data`):**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | text | ✅ | Trail name |
| `description` | text | ✅ | Trail description |
| `region` | text | ✅ | Region code |
| `difficulty` | text | ✅ | `easy` · `moderate` · `hard` |
| `distance` | number | ✅ | Distance in km |
| `estimated_duration` | text | ✅ | Estimated duration |
| `start_latitude` / `start_longitude` | number | ✅ | Starting point |
| `end_latitude` / `end_longitude` | number | ✅ | End point |
| `route_file` | file (.geojson) | ✅ | Trail route as a LineString |
| `safety_tips` | text | ❌ | Safety tips |
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

**Errors:** `400` invalid data or GeoJSON

#### 21. PUT `/api/admin/trails/{id}` 🛠️

**Input (JSON):** the trail information fields being changed (same fields as endpoint 20, without files).

**Output — 200 OK:** The updated trail object.

**Errors:** `400` invalid data · `404` trail not found

#### 22. DELETE `/api/admin/trails/{id}` 🛠️

**Output — 204 No Content**

> Deleting a trail also deletes its reviews, favorites, and completed entries, and removes its photos from Cloudinary.

**Errors:** `404` trail not found

#### 23. POST `/api/admin/trails/{id}/images` 🛠️

**Input (`multipart/form-data`):** `images` — one or more photo files.

**Output — 201 Created:**
```json
{
  "trail_id": 7,
  "images": [
    "https://res.cloudinary.com/.../soudah-1.jpg",
    "https://res.cloudinary.com/.../soudah-3.jpg"
  ]
}
```

**Errors:** `400` invalid file type · `404` trail not found

#### 24. PUT `/api/admin/trails/{id}/route` 🛠️

**Input (`multipart/form-data`):** `route_file` — a `.geojson` file with the trail route as a LineString.

**Output — 200 OK:**
```json
{ "trail_id": 7, "message": "Route updated" }
```

> The app previews the new route on Google Maps before the admin saves it.

**Errors:** `400` invalid GeoJSON · `404` trail not found

#### 25. DELETE `/api/admin/reviews/{id}` 🛠️

**Output — 204 No Content**

> Admins can delete any inappropriate review directly from the trail's reviews list.

**Errors:** `404` review not found

#### 26. PATCH `/api/admin/users/{id}/suspend` 🛠️

**Input:** User ID in the URL path. No body.

**Output — 200 OK:**
```json
{ "id": 12, "is_suspended": true }
```

> The admin suspends a user from the author of an inappropriate review. A suspended user cannot log in, and the server checks `isSuspended` on every request, so their existing token is rejected immediately.

**Errors:** `404` user not found

---

### HTTP Status Codes Used

| Code | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 201 Created | New resource created |
| 204 No Content | Deleted successfully, no body |
| 400 Bad Request | Invalid input, invalid GeoJSON, or wrong or expired verification code |
| 401 Unauthorized | Missing or expired token |
| 403 Forbidden | Not allowed (not the owner, not an admin, email not verified, or account suspended) |
| 404 Not Found | Resource not found |
| 409 Conflict | Duplicate (email exists, email already verified, or trail already in favorites or completed) |
| 429 Too Many Requests | Too many verification attempts or code requests |
