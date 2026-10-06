# 4. API Specifications

## External APIs

| API | Used By | Purpose | Why It Was Chosen |
|---|---|---|---|
| Google Maps SDK for Android / iOS (via `google_maps_flutter`) | Flutter app (hiker screens and admin panel) | Displays trails on the map, draws each trail's GeoJSON route with its starting point, and shows the user's current location (My Location layer). In the admin panel, it previews an uploaded GeoJSON route before saving. | Reliable map coverage of Saudi Arabia, custom markers, route lines, and a built-in current-location layer. Has an official Flutter package, and its monthly free usage covers the MVP. |
| Cloudinary Upload API (via `cloudinary` Python SDK) | Flask backend | Uploads trail photos and returns their URLs. Photos are served to the app via Cloudinary's CDN. | Free plan with no credit card required. Images are not lost when the server redeploys. Automatic resizing and CDN delivery make photos load fast on mobile. |
| Email Service (SMTP via `Flask-Mail`) | Flask backend | Sends a one-time verification code (OTP) to the user's email during sign-up. | Confirms that each account uses a real email address the user owns, which reduces fake accounts. Flask-Mail integrates directly with Flask and works with standard SMTP email providers. |

> **Device services (not external APIs):** the user's current location is read on the device with the `geolocator` Flutter package, only with the user's permission and only while the app is open. It is never sent to the server.
>
> **Data format (not an external API):** trail routes are stored and exchanged as **GeoJSON**. Each route is a `FeatureCollection` containing the trail path as a `LineString` and the starting point as a `Point`, stored in a MySQL JSON column. Coordinates follow the GeoJSON order `[longitude, latitude]`.
>
> **API keys** are never shared publicly: the Google Maps key is restricted to Qimmah's app, and Cloudinary and email credentials are stored only on the server as environment variables.

## Internal API (Flask REST API)

### General Rules

- **Base URL:** `https://<qimmah-backend>.up.railway.app/api` (hosted on Railway)
- **Format:** All requests and responses use JSON, except admin trail uploads, which use `multipart/form-data` (to send files).
- **Authentication:** Protected endpoints require a JWT in the header: `Authorization: Bearer <token>`
- **Access levels:** 🌐 Guest (no account) · 🔒 Registered user · 🛠️ Admin only
- **Admin access:** the admin uses the same Flutter app; admin screens appear only for accounts with the `admin` role, and every 🛠️ endpoint checks this role on the server.
- **Common errors:** every 🔒 and 🛠️ endpoint returns `401` if the token is missing, expired, or revoked, and every 🛠️ endpoint returns `403` if the user is not an admin.
- **Error format:** `{ "error": "Error message" }`

### Endpoints Overview

| # | Module | Method | URL Path | Access | Description | User Story |
|---|---|---|---|---|---|---|
| 1 | Auth | POST | `/api/auth/register` | 🌐 | Create an account and send a verification code | 1 |
| 2 | Auth | POST | `/api/auth/verify-email` | 🌐 | Verify the email with the OTP code and receive a JWT | 1 |
| 3 | Auth | POST | `/api/auth/resend-code` | 🌐 | Send a new verification code | 1 |
| 4 | Auth | POST | `/api/auth/login` | 🌐 | Log in and receive a JWT | 2 |
| 5 | Auth | POST | `/api/auth/logout` | 🔒 | Log out and revoke the token | 2 |
| 6 | Users | GET | `/api/users/me` | 🔒 | Get the user's account information | 17 |
| 7 | Users | PUT | `/api/users/me` | 🔒 | Edit account information | 17 |
| 8 | Users | GET | `/api/users/me/favorites` | 🔒 | List favorite trails | 11 |
| 9 | Users | POST | `/api/users/me/favorites` | 🔒 | Add a trail to favorites | 11 |
| 10 | Users | DELETE | `/api/users/me/favorites/{trail_id}` | 🔒 | Remove a trail from favorites | 11 |
| 11 | Users | GET | `/api/users/me/completed-trails` | 🔒 | List completed trails | 16 |
| 12 | Users | POST | `/api/users/me/completed-trails` | 🔒 | Mark a trail as completed | 16 |
| 13 | Users | DELETE | `/api/users/me/completed-trails/{trail_id}` | 🔒 | Unmark a completed trail | 16 |
| 14 | Trails | GET | `/api/trails` | 🌐 | List trails with filters and search | 3, 4, 5, 8 |
| 15 | Trails | GET | `/api/trails/{id}` | 🌐 | Get trail details, GeoJSON route, starting point, and safety tips | 6, 7, 18, 22 |
| 16 | Reviews | GET | `/api/trails/{id}/reviews` | 🌐 | List a trail's reviews | 9 |
| 17 | Reviews | POST | `/api/trails/{id}/reviews` | 🔒 | Rate and review a trail | 12 |
| 18 | Reviews | PUT | `/api/reviews/{id}` | 🔒 | Edit own review | 13 |
| 19 | Reviews | DELETE | `/api/reviews/{id}` | 🔒 / 🛠️ | Delete own review, or any review (admin) | 13, 15 |
| 20 | Admin | POST | `/api/admin/trails` | 🛠️ | Add a new trail | 14 |
| 21 | Admin | PUT | `/api/admin/trails/{id}` | 🛠️ | Edit a trail | 14 |
| 22 | Admin | DELETE | `/api/admin/trails/{id}` | 🛠️ | Delete a trail | 14 |
| 23 | Admin | GET | `/api/admin/reviews` | 🛠️ | List all reviews for moderation | 15 |
| 24 | Admin | GET | `/api/admin/users` | 🛠️ | List users | 21 |
| 25 | Admin | PATCH | `/api/admin/users/{id}/status` | 🛠️ | Suspend or reactivate a user | 21 |

> **Handled in the app, not the API:**
> - **Story 10 (guest sign-up prompt):** the app shows the prompt when a guest tries an action that needs a 🔒 endpoint.
> - **Story 19 (share a trail):** the app shares the trail's link through the device's share menu (`share_plus`).
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

> The account is created as unverified, and a 6-digit code is emailed to the user. The code expires after 10 minutes. No token is returned until the email is verified.

**Errors:** `400` missing or invalid fields · `409` email already registered

#### 2. POST `/api/auth/verify-email` 🌐

**Input (JSON):**
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

> A new code replaces the old one. Requests are limited (e.g., once per minute) to prevent abuse.

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

**Errors:** `401` invalid email or password · `403` email not verified, or account suspended

#### 5. POST `/api/auth/logout` 🔒

**Input:** JWT in header only.

**Output — 200 OK:**
```json
{ "message": "Logged out successfully" }
```

> The token is added to a blocklist on the server so it can no longer be used, and the app deletes it from secure storage.

---

### Users Module

#### 6. GET `/api/users/me` 🔒

**Output — 200 OK:**
```json
{
  "id": 12,
  "name": "Sara",
  "email": "sara@example.com",
  "role": "user",
  "favorites_count": 4,
  "completed_trails_count": 2,
  "reviews_count": 3
}
```

#### 7. PUT `/api/users/me` 🔒

**Input (JSON):** only the fields being changed. Changing the password requires the current password.
```json
{
  "name": "Sara A.",
  "current_password": "StrongPass123",
  "new_password": "NewStrongPass456"
}
```

**Output — 200 OK:** The updated user object.

**Errors:** `400` invalid fields · `401` wrong current password

#### 8. GET `/api/users/me/favorites` 🔒

**Output — 200 OK:** a list of favorite trails in the same summary structure as endpoint 14.

#### 9. POST `/api/users/me/favorites` 🔒

**Input (JSON):**
```json
{ "trail_id": 7 }
```

**Output — 201 Created:**
```json
{ "message": "Trail added to favorites", "trail_id": 7 }
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

#### 15. GET `/api/trails/{id}` 🌐

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
  "share_url": "https://<qimmah-domain>/trails/7",
  "is_favorite": false,
  "is_completed": false
}
```

> `is_favorite` and `is_completed` are returned only when a valid JWT is sent. The app draws `route_geojson` on the map and shows the user's current location over it (story 22).

**Errors:** `404` trail not found

---

### Reviews Module

#### 16. GET `/api/trails/{id}/reviews` 🌐

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
  "created_at": "2026-10-01T09:30:00Z",
  "trail_avg_rating": 4.8
}
```

**Errors:** `400` rating not 1–5 or empty comment · `404` trail not found · `409` already reviewed this trail

#### 18. PUT `/api/reviews/{id}` 🔒

**Input (JSON):** the fields being changed.
```json
{
  "rating": 4,
  "comment": "مسار جميل لكنه مزدحم في الإجازات"
}
```

**Output — 200 OK:** The updated review, with the trail's new average rating.

**Errors:** `400` invalid input · `403` not the review's owner · `404` review not found

#### 19. DELETE `/api/reviews/{id}` 🔒 / 🛠️

**Output — 204 No Content**

> A user can delete only their own reviews; an admin can delete any review. The trail's average rating is recalculated after deletion.

**Errors:** `403` not the owner and not an admin · `404` review not found

---

### Admin Module

#### 20. POST `/api/admin/trails` 🛠️

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

**Errors:** `400` invalid data or GeoJSON

#### 21. PUT `/api/admin/trails/{id}` 🛠️

**Input (`multipart/form-data`):** same fields as endpoint 20; only the changed fields are sent.

**Output — 200 OK:** The updated trail object.

**Errors:** `400` invalid data or GeoJSON · `404` trail not found

#### 22. DELETE `/api/admin/trails/{id}` 🛠️

**Output — 204 No Content**

> Deleting a trail also deletes its reviews, favorites, and completed entries, and removes its photos from Cloudinary.

**Errors:** `404` trail not found

#### 23. GET `/api/admin/reviews` 🛠️

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

> To delete a review, the admin uses endpoint 19.

#### 24. GET `/api/admin/users` 🛠️

**Input (query, optional):** `search`, `page`, `limit`

**Output — 200 OK:**
```json
{
  "users": [
    { "id": 12, "name": "Sara", "email": "sara@example.com", "reviews_count": 3, "is_verified": true, "is_suspended": false }
  ],
  "page": 1,
  "total": 1
}
```

#### 25. PATCH `/api/admin/users/{id}/status` 🛠️

**Input (JSON):**
```json
{ "is_suspended": true }
```

**Output — 200 OK:**
```json
{ "id": 12, "is_suspended": true }
```

> A suspended user cannot log in, and their existing tokens are rejected.

**Errors:** `404` user not found

---

### HTTP Status Codes Used

| Code | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 201 Created | New resource created |
| 204 No Content | Deleted successfully, no body |
| 400 Bad Request | Invalid input, or wrong or expired verification code |
| 401 Unauthorized | Missing, invalid, expired, or revoked token |
| 403 Forbidden | Not allowed (not the owner, not an admin, email not verified, or account suspended) |
| 404 Not Found | Resource not found |
| 409 Conflict | Duplicate (email exists, email already verified, or trail already in favorites, completed, or reviewed) |
| 429 Too Many Requests | Too many verification attempts or code requests |
