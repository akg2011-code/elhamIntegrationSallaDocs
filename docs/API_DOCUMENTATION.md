# API Documentation

## Overview

This document provides comprehensive documentation for all API endpoints, webhook handlers, and external API integrations in the Salla-Elham Integration platform.

## Table of Contents

1. [Internal API Endpoints](#internal-api-endpoints)
2. [Webhook Handlers](#webhook-handlers)
3. [External API Integrations](#external-api-integrations)
4. [Authentication](#authentication)
5. [Error Handling](#error-handling)

---

## Internal API Endpoints

### Base URL

- **Development**: `http://localhost:3000`
- **Production**: Configured via `NEXT_PUBLIC_APP_URL`

All endpoints are Next.js API routes located in `app/api/`.

---

### 1. GET /api/elham/courses

**Purpose**: Fetch Elham courses/journeys for the authenticated user.

**Authentication**: Required (Supabase Auth)

**Request**:
```http
GET /api/elham/courses
```

**Headers**:
- Cookie: Supabase session cookie (automatic)

**Query Parameters**: None

**Response**:

**Success (200)**:
```json
{
  "courses": [
    {
      "id": 123,
      "name": "Course Name (localized)",
      "description": "Course description (localized)",
      "title_ar": "اسم الدورة",
      "title_en": "Course Name",
      "cover_photo": "https://example.com/image.jpg",
      ...
    }
  ]
}
```

**Error (400)**:
```json
{
  "error": "Elham API key not found. Please add it in settings."
}
```

**Error (401)**:
```json
{
  "error": "Unauthorized"
}
```

**Error (500)**:
```json
{
  "error": "Failed to fetch courses"
}
```

**Implementation Details**:
- Decrypts user's Elham API key
- Calls Elham API: `GET https://api-ksa.elhamsol.com/api/v1/integration/lx/`
- Localizes course names based on user's locale (from cookies)
- Returns courses with localized `name` and `description` fields

**Code Reference**: `app/api/elham/courses/route.ts`

---

### 2. GET /api/elham/teams

**Purpose**: Fetch Elham teams for the authenticated user.

**Authentication**: Required (Supabase Auth)

**Request**:
```http
GET /api/elham/teams
```

**Headers**:
- Cookie: Supabase session cookie (automatic)

**Query Parameters**: None

**Response**:

**Success (200)**:
```json
{
  "teams": [
    {
      "id": 456,
      "title": "Team Name",
      "lxs": [123, 124, 125],
      "trainees": [789, 790]
    }
  ]
}
```

**Error (400)**:
```json
{
  "error": "Elham API key not found. Please add it in settings."
}
```

**Error (401)**:
```json
{
  "error": "Unauthorized"
}
```

**Error (500)**:
```json
{
  "error": "Failed to fetch teams"
}
```

**Implementation Details**:
- Decrypts user's Elham API key
- Calls Elham API: `GET https://api-ksa.elhamsol.com/api/v1/integration/teams/`
- Returns teams with associated course IDs (lxs) and trainee IDs

**Code Reference**: `app/api/elham/teams/route.ts`

---

### 3. POST /api/auth/logout

**Purpose**: Logout the current user.

**Authentication**: Required (Supabase Auth)

**Request**:
```http
POST /api/auth/logout
```

**Headers**:
- Cookie: Supabase session cookie (automatic)

**Response**:

**Success (200)**:
```json
{
  "success": true
}
```

**Implementation Details**:
- Signs out user from Supabase Auth
- Clears session cookies
- Redirects to login page

**Code Reference**: `app/api/auth/logout/route.ts`

---

## Webhook Handlers

### 4. POST /api/webhooks/salla

**Purpose**: Handle webhook events from Salla platform.

**Authentication**: None (webhook endpoint)

**Request**:
```http
POST /api/webhooks/salla
Content-Type: application/json
```

**Headers**:
- `Content-Type: application/json`

**Request Body**: Varies by event type (see below)

**Response**:

**Success (200)**:
```json
{
  "success": true
}
```

**Error (400)**:
```json
{
  "error": "Missing merchant ID or data"
}
```

**Error (404)**:
```json
{
  "error": "User not found"
}
```

**Error (500)**:
```json
{
  "error": "Webhook processing failed"
}
```

**Code Reference**: `app/api/webhooks/salla/route.ts`

---

#### Event: app.store.authorize

**Purpose**: Handle Salla store authorization (OAuth).

**Request Body**:
```json
{
  "event": "app.store.authorize",
  "merchant": "12345",
  "data": {
    "access_token": "salla_access_token_here",
    "refresh_token": "salla_refresh_token_here"
  }
}
```

**Processing**:
1. Extract merchant ID and tokens
2. Fetch merchant info from Salla API
3. Extract merchant email
4. Check if user exists by email
5. If user exists: Update tokens and merchant ID
6. If user doesn't exist:
   - Generate random password
   - Create user in Supabase Auth
   - Create users table record
   - Send welcome email
7. Encrypt and store tokens

**Response**: `{ success: true }`

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 23-214)

---

#### Event: order.created, order.updated, order.customer.updated, order.products.updated

**Purpose**: Handle order events and process enrollments.

**Request Body**:
```json
{
  "event": "order.created",
  "merchant": "12345",
  "data": {
    "id": "order_123",
    "status": {
      "name": "تم التنفيذ"
    },
    "customer": {
      "email": "customer@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "mobile": "1234567890",
      "mobile_code": "+966"
    },
    "items": [
      {
        "product": {
          "id": "product_456",
          "name": "Product Name"
        }
      }
    ]
  }
}
```

**Processing**:
1. Extract merchant ID and order data
2. Find user by merchant ID
3. For each product in order:
   - Check if product has mapping
   - Create/update order_syncs record
   - If order status is "تم التنفيذ":
     - Get or create trainee in Elham
     - Add trainee to team
     - Update order_syncs status

**Response**: `{ success: true }`

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 217-426)

---

#### Event: order.refunded, order.cancelled

**Purpose**: Handle refund/cancellation and remove trainees from teams.

**Request Body**:
```json
{
  "event": "order.refunded",
  "merchant": "12345",
  "data": {
    "id": "order_123"
  }
}
```

**Processing**:
1. Extract merchant ID and order ID
2. Find user by merchant ID
3. Find all order_syncs for the order
4. For each sync record:
   - Remove trainee from team in Elham
   - Update order_syncs status

**Response**: `{ success: true }`

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 428-541)

---

#### GET /api/webhooks/salla

**Purpose**: Webhook verification endpoint.

**Request**:
```http
GET /api/webhooks/salla
```

**Response**:
```json
{
  "message": "Salla webhook endpoint"
}
```

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 554-557)

---

## External API Integrations

### Salla API

**Base URL**: `https://api.salla.dev`

**Authentication**: OAuth 2.0 Bearer Token

**Code Reference**: `lib/salla/api.ts`

---

#### GET /admin/v2/products

**Purpose**: Fetch products from Salla store.

**Authentication**: Bearer token (Salla access token)

**Request**:
```http
GET https://api.salla.dev/admin/v2/products
Authorization: Bearer {access_token}
```

**Response**:
```json
[
  {
    "id": "product_123",
    "name": "Product Name",
    "description": "Product description",
    "main_image": "https://example.com/image.jpg",
    "price": 100.00,
    "status": "active"
  }
]
```

**Implementation**: `fetchSallaProducts(accessToken: string)`

**Code Reference**: `lib/salla/api.ts` (lines 90-119)

---

#### GET /admin/v2/products/{id}

**Purpose**: Get specific product by ID.

**Authentication**: Bearer token (Salla access token)

**Request**:
```http
GET https://api.salla.dev/admin/v2/products/{productId}
Authorization: Bearer {access_token}
```

**Response**:
```json
{
  "id": "product_123",
  "name": "Product Name",
  ...
}
```

**Implementation**: `getSallaProductById(accessToken: string, productId: string)`

**Code Reference**: `lib/salla/api.ts` (lines 124-143)

---

#### GET /oauth2/user/info

**Purpose**: Get merchant information.

**Authentication**: Bearer token (Salla access token)

**Request**:
```http
GET https://accounts.salla.sa/oauth2/user/info
Authorization: Bearer {access_token}
```

**Response**:
```json
{
  "status": 200,
  "success": true,
  "data": {
    "id": 123,
    "name": "Merchant Name",
    "email": "merchant@example.com",
    "mobile": "1234567890",
    "merchant": {
      "id": 456,
      "name": "Store Name",
      "domain": "store.salla.sa"
    }
  }
}
```

**Implementation**: `fetchSallaMerchantInfo(accessToken: string)`

**Code Reference**: `lib/salla/api.ts` (lines 148-166)

---

#### POST /oauth2/token (Token Refresh)

**Purpose**: Refresh Salla OAuth token.

**Authentication**: Client credentials

**Request**:
```http
POST https://accounts.salla.sa/oauth2/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "{refresh_token}",
  "client_id": "{client_id}",
  "client_secret": "{client_secret}"
}
```

**Response**:
```json
{
  "access_token": "new_access_token",
  "refresh_token": "new_refresh_token",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

**Implementation**: `refreshSallaToken(refreshToken: string, clientId: string, clientSecret: string)`

**Code Reference**: `lib/salla/api.ts` (lines 61-85)

---

### Elham API

**Base URLs**:
- Integration API: `https://api-ksa.elhamsol.com/api/v1/integration/`
- Legacy API: `https://api.elham.academy/api/v1/`

**Authentication**: API Key in `API-KEY` header

**Code Reference**: `lib/elham/api.ts`

---

#### GET /integration/lx/

**Purpose**: Fetch journeys (courses) from Elham.

**Authentication**: API Key

**Request**:
```http
GET https://api-ksa.elhamsol.com/api/v1/integration/lx/
API-KEY: {api_key}
Content-Type: application/json
```

**Response**:
```json
{
  "count": 10,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 123,
      "title_ar": "اسم الدورة",
      "title_en": "Course Name",
      "description_ar": "وصف الدورة",
      "description_en": "Course description",
      "cover_photo": "https://example.com/image.jpg",
      ...
    }
  ]
}
```

**Implementation**: `fetchJourneys(apiKey: string, locale: string)`

**Code Reference**: `lib/elham/api.ts` (lines 150-195)

---

#### GET /integration/teams/

**Purpose**: Fetch teams from Elham.

**Authentication**: API Key

**Request**:
```http
GET https://api-ksa.elhamsol.com/api/v1/integration/teams/
API-KEY: {api_key}
Content-Type: application/json
```

**Response**:
```json
{
  "count": 5,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 456,
      "title": "Team Name",
      "lxs": [123, 124, 125],
      "trainees": [789, 790]
    }
  ]
}
```

**Implementation**: `fetchTeams(apiKey: string)`

**Code Reference**: `lib/elham/api.ts` (lines 259-299)

---

#### GET /integration/trainees/

**Purpose**: Get trainee by email.

**Authentication**: API Key

**Request**:
```http
GET https://api-ksa.elhamsol.com/api/v1/integration/trainees/?email={email}
API-KEY: {api_key}
Content-Type: application/json
```

**Response**:
```json
{
  "count": 1,
  "results": [
    {
      "id": 789,
      "email": "trainee@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "mobile": "+9661234567890"
    }
  ]
}
```

**Implementation**: `getTraineeByEmail(apiKey: string, email: string)`

**Returns**: Trainee ID (number) or null if not found

**Code Reference**: `lib/elham/api.ts` (lines 304-334)

---

#### POST /integration/trainees/

**Purpose**: Create new trainee in Elham.

**Authentication**: API Key

**Request**:
```http
POST https://api-ksa.elhamsol.com/api/v1/integration/trainees/
API-KEY: {api_key}
Content-Type: application/json

{
  "email": "trainee@example.com",
  "mobile": "+9661234567890",
  "first_name": "John",
  "middle_name": null,
  "last_name": "Doe"
}
```

**Response**:
```json
{
  "id": 789,
  "email": "trainee@example.com",
  ...
}
```

**Implementation**: `addTrainee(apiKey: string, traineeData: AddTraineeRequest)`

**Code Reference**: `lib/elham/api.ts` (lines 339-362)

---

#### POST /integration/teams/add-trainees/

**Purpose**: Add trainees to a team.

**Authentication**: API Key

**Request**:
```http
POST https://api-ksa.elhamsol.com/api/v1/integration/teams/add-trainees/
API-KEY: {api_key}
Content-Type: application/json

{
  "team": 456,
  "trainees": [789, 790],
  "clear": false
}
```

**Response**: Success response (format varies)

**Implementation**: `addTraineesToTeam(apiKey: string, teamData: AddTraineesToTeamRequest)`

**Code Reference**: `lib/elham/api.ts` (lines 367-389)

---

#### DELETE /integration/teams/remove-trainee/{teamId}/{traineeId}/

**Purpose**: Remove trainee from team.

**Authentication**: API Key

**Request**:
```http
DELETE https://api-ksa.elhamsol.com/api/v1/integration/teams/remove-trainee/456/789/
API-KEY: {api_key}
Content-Type: application/json
```

**Response**: Success response (format varies)

**Implementation**: `removeTraineeFromTeam(apiKey: string, teamId: number, traineeId: number)`

**Code Reference**: `lib/elham/api.ts` (lines 394-420)

---

## Authentication

### Internal API Authentication

All internal API endpoints (except webhooks) require Supabase Auth authentication:

1. **Session-Based**: Uses Supabase session cookies
2. **Server-Side**: Validated via `createClient()` from `lib/supabase/server.ts`
3. **Client-Side**: Validated via `createClient()` from `lib/supabase/client.ts`

**Example**:
```typescript
const supabase = await createClient();
const { data: { user }, error } = await supabase.auth.getUser();

if (error || !user) {
  return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
}
```

### External API Authentication

#### Salla API
- **Method**: OAuth 2.0 Bearer Token
- **Token Storage**: Encrypted in database
- **Token Refresh**: Supported via `refreshSallaToken()`

#### Elham API
- **Method**: API Key in header
- **Header**: `API-KEY: {api_key}`
- **Storage**: Encrypted in database
- **Usage**: Decrypted only when making API calls

---

## Error Handling

### Error Response Format

All errors follow this format:
```json
{
  "error": "Error message here"
}
```

### HTTP Status Codes

- **200**: Success
- **400**: Bad Request (missing/invalid parameters)
- **401**: Unauthorized (authentication required)
- **404**: Not Found (resource doesn't exist)
- **500**: Internal Server Error (server-side error)

### Common Error Scenarios

1. **Missing Authentication**:
   - Status: 401
   - Message: "Unauthorized"

2. **Missing API Key**:
   - Status: 400
   - Message: "Elham API key not found. Please add it in settings."

3. **User Not Found**:
   - Status: 404
   - Message: "User not found"

4. **API Call Failure**:
   - Status: 500
   - Message: Error message from API or generic error

5. **Invalid Request**:
   - Status: 400
   - Message: Specific validation error

### Error Logging

- All errors logged to console with context
- Webhook errors stored in `order_syncs.error_message`
- API errors include original error details

---

## Rate Limiting

Currently, no rate limiting is implemented. Consider implementing:
- Per-user rate limits
- Per-IP rate limits
- API-specific rate limits

---

## Webhook Security

### Current Implementation

- Webhook endpoint is publicly accessible
- No signature verification implemented
- Relies on Salla's webhook security

### Recommended Enhancements

1. **Webhook Signature Verification**:
   - Verify webhook signatures using `SALLA_CLIENT_SECRET`
   - Validate request authenticity

2. **IP Whitelisting**:
   - Restrict webhook endpoint to Salla IPs

3. **Request Validation**:
   - Validate required fields
   - Check merchant ID exists

---

## API Usage Examples

### Fetching Courses (Client-Side)

```typescript
const response = await fetch('/api/elham/courses');
const data = await response.json();
if (data.courses) {
  // Use courses
}
```

### Fetching Teams (Client-Side)

```typescript
const response = await fetch('/api/elham/teams');
const data = await response.json();
if (data.teams) {
  // Use teams
}
```

### Creating Mapping (Client-Side)

```typescript
const { data, error } = await supabase
  .from('product_course_map')
  .insert({
    user_id: user.id,
    salla_product_id: productId,
    elham_course_id: courseId,
    elham_team_id: teamId,
    // ... other fields
  });
```

---

## Testing

### Webhook Testing

Use tools like:
- **ngrok**: Expose local server for webhook testing
- **Postman**: Send test webhook payloads
- **Salla Developer Console**: Test webhook events

### API Testing

- Use browser DevTools Network tab
- Use Postman/Insomnia for API testing
- Test authentication flows
- Test error scenarios

---

*For business logic, see [Business Logic Documentation](./BUSINESS_LOGIC.md)*
*For technical details, see [Technical Documentation](./TECHNICAL_DOCUMENTATION.md)*
*For workflows, see [Workflows Documentation](./WORKFLOWS.md)*

