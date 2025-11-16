# Workflows Documentation

## Overview

This document provides detailed step-by-step workflows for all user journeys and system processes in the Salla-Elham Integration platform.

## Table of Contents

1. [User Registration and Authentication](#user-registration-and-authentication)
2. [Salla Store Connection (OAuth)](#salla-store-connection-oauth)
3. [Elham API Key Configuration](#elham-api-key-configuration)
4. [Product-to-Course Mapping](#product-to-course-mapping)
5. [Order Processing and Enrollment Automation](#order-processing-and-enrollment-automation)
6. [Refund and Cancellation Handling](#refund-and-cancellation-handling)
7. [Email Notification Workflows](#email-notification-workflows)

---

## User Registration and Authentication

### Workflow 1A: Manual User Registration

**Trigger**: User visits registration page

**Steps**:

1. **User Access Registration Page**

   - URL: `/register`
   - Page: `app/(auth)/register/page.tsx`
   - User sees registration form with email and password fields

2. **User Enters Credentials**

   - Email address (required)
   - Password (required, minimum length enforced by Supabase)
   - Form validation occurs client-side

3. **Submit Registration**

   - User clicks "Sign Up" button
   - Form submission triggers `handleRegister` function
   - Supabase Auth `signUp()` called with email and password

4. **Account Creation**

   - Supabase Auth creates user account
   - Email confirmation handled (if enabled)
   - User record created in `auth.users` table

5. **Create Users Table Record**

   - System creates corresponding record in `users` table
   - Links to `auth.users` via foreign key
   - Initializes with user email

6. **Redirect to Dashboard**
   - User automatically logged in
   - Redirected to `/dashboard`
   - Session established via Supabase Auth

**Code Reference**: `app/(auth)/register/page.tsx`

**Error Handling**:

- Email already exists → Show error message
- Weak password → Supabase validation error
- Network error → Show generic error message

---

### Workflow 1B: User Login

**Trigger**: User visits login page

**Steps**:

1. **User Access Login Page**

   - URL: `/login`
   - Page: `app/(auth)/login/page.tsx`
   - User sees login form

2. **User Enters Credentials**

   - Email address
   - Password
   - Optional: "Remember me" functionality

3. **Submit Login**

   - User clicks "Sign In" button
   - Form submission triggers `handleLogin` function
   - Supabase Auth `signInWithPassword()` called

4. **Authentication**

   - Supabase validates credentials
   - Session created if valid
   - Session stored in secure cookies

5. **Redirect to Dashboard**
   - User redirected to `/dashboard`
   - Protected routes now accessible

**Code Reference**: `app/(auth)/login/page.tsx`

**Error Handling**:

- Invalid credentials → Show error message
- User not found → Show error message
- Network error → Show generic error message

---

### Workflow 1C: Automatic User Creation (Salla Webhook)

**Trigger**: Salla store authorizes the app

**Steps**:

1. **Salla Sends Authorization Webhook**

   - Event: `app.store.authorize`
   - Endpoint: `POST /api/webhooks/salla`
   - Payload includes:
     - `merchant`: Merchant ID
     - `data.access_token`: OAuth access token
     - `data.refresh_token`: OAuth refresh token

2. **Extract Merchant Information**

   - System extracts `merchant` ID from webhook
   - Uses `access_token` to call Salla API
   - Endpoint: `GET https://accounts.salla.sa/oauth2/user/info`
   - Retrieves merchant email and name

3. **Check User Existence**

   - Query `users` table by email
   - If user exists:
     - Update tokens and merchant ID
     - Skip to step 6
   - If user doesn't exist:
     - Continue to step 4

4. **Generate Random Password**

   - Generate 16-character random password
   - Function: `generateRandomPassword(16)`
   - Password includes letters, numbers, and special characters

5. **Create User Account**

   - Create user in Supabase Auth:
     - Email from merchant info
     - Generated password
     - Email confirmed automatically
   - Create record in `users` table:
     - Link to auth user
     - Store encrypted tokens
     - Store merchant ID

6. **Encrypt and Store Tokens**

   - Encrypt `access_token` using AES
   - Encrypt `refresh_token` using AES
   - Store encrypted values in `users` table:
     - `salla_access_token`
     - `salla_refresh_token`
     - `salla_merchant_id`

7. **Send Welcome Email** (New users only)

   - Compose welcome email with:
     - Login credentials (email and password)
     - Login URL
     - Merchant name (if available)
   - Send via SMTP
   - Email includes HTML and plain text versions

8. **Return Success Response**
   - Return `{ success: true }` to Salla
   - Webhook processing complete

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 23-214)

**Error Handling**:

- Missing merchant ID → Return 400 error
- Missing tokens → Return 400 error
- API call failure → Return 500 error
- Email send failure → Log error but continue (non-blocking)

---

## Salla Store Connection (OAuth)

### Workflow 2: Salla OAuth Connection

**Trigger**: Merchant installs Salla app in their store

**Steps**:

1. **Merchant Installs App**

   - Merchant goes to Salla App Store
   - Clicks "Install" on the integration app
   - Salla initiates OAuth flow

2. **OAuth Authorization**

   - Merchant authorizes app permissions
   - Salla generates OAuth tokens
   - Salla sends webhook to integration platform

3. **Webhook Processing** (See Workflow 1C)

   - System receives `app.store.authorize` event
   - Creates/updates user account
   - Stores encrypted tokens

4. **Connection Established**
   - Merchant account now linked to Salla store
   - Tokens stored securely
   - Merchant can now:
     - View products from Salla
     - Create product mappings
     - Receive order webhooks

**Verification**:

- Merchant can see connection status in Dashboard
- `salla_merchant_id` present in `users` table
- Connection status shown as "Connected"

**Code Reference**:

- `app/api/webhooks/salla/route.ts` (lines 23-214)
- `app/(dashboard)/dashboard/page.tsx` (connection status check)

---

## Elham API Key Configuration

### Workflow 3: Adding Elham API Key

**Trigger**: User navigates to Settings page

**Steps**:

1. **User Access Settings Page**

   - URL: `/settings`
   - Page: `app/(dashboard)/settings/page.tsx`
   - User sees API key input field

2. **Check Existing API Key**

   - System checks if user has existing API key
   - If exists: Show masked value ("••••••••••••")
   - If not: Show empty input field

3. **User Enters API Key**

   - User pastes Elham API key
   - Input validation (if any)

4. **Submit API Key**

   - User clicks "Save API Key" button
   - Form submission triggers `handleSaveApiKey`

5. **Encrypt API Key**

   - System encrypts API key using AES
   - Function: `encryptApiKey(apiKey)`
   - Uses `ENCRYPTION_SECRET` from environment

6. **Store Encrypted Key**

   - Update `users` table:
     - Set `elham_api_key_encrypted` to encrypted value
     - Update `updated_at` timestamp

7. **Show Success Message**
   - Display success notification
   - Mask API key in input field
   - API key now ready for use

**Code Reference**: `app/(dashboard)/settings/page.tsx`

**Error Handling**:

- Invalid API key format → Validation error
- Database error → Show error message
- Encryption error → Show error message

**Security Notes**:

- API key never stored in plain text
- Only encrypted value stored in database
- API key decrypted only when making Elham API calls

---

## Product-to-Course Mapping

### Workflow 4: Creating Product-to-Course Mapping

**Trigger**: User navigates to Mapping page

**Steps**:

1. **User Access Mapping Page**

   - URL: `/mapping`
   - Page: `app/(dashboard)/mapping/page.tsx`
   - User sees mapping form and existing mappings table

2. **Load Required Data**

   - **Salla Products**:
     - Decrypt Salla access token
     - Call Salla API: `GET /admin/v2/products`
     - Display products in dropdown
   - **Elham Teams**:
     - Decrypt Elham API key
     - Call Elham API: `GET /api/elham/teams`
     - Display teams in dropdown
   - **Elham Courses**:
     - Decrypt Elham API key
     - Call Elham API: `GET /api/elham/courses`
     - Load into CoursesContext
     - Display courses nested under teams

3. **User Selects Salla Product**

   - User clicks product dropdown
   - Sees list of products with images
   - Selects a product
   - Product ID stored in state

4. **User Selects Elham Team and Course**

   - User clicks team/course dropdown
   - Sees hierarchical structure:
     - Teams (top level)
     - Courses (LXs) nested under teams
   - User selects team and course combination
   - Team ID and Course ID stored in state

5. **Validate Selection**

   - Check both product and course selected
   - Check if mapping already exists
   - Query `product_course_map` table:
     - Same `user_id`
     - Same `salla_product_id`
     - Same `elham_course_id`

6. **Create Mapping**

   - If mapping doesn't exist:
     - Insert into `product_course_map`:
       - `user_id`
       - `salla_product_id`
       - `elham_course_id`
       - `elham_team_id`
       - `elham_team_name`
       - `salla_product_name`
       - `salla_product_image`
       - `elham_course_name`
       - `elham_course_image`
   - If mapping exists:
     - Show error: "Mapping already exists"

7. **Update UI**
   - Show success message
   - Refresh mappings list
   - Clear form selections
   - New mapping appears in table

**Code Reference**: `app/(dashboard)/mapping/page.tsx` (lines 190-281)

**Error Handling**:

- Salla API error → Show error message
- Elham API error → Show error message
- Duplicate mapping → Show error message
- Database error → Show error message

**Deleting a Mapping**:

1. User clicks "Delete" button on mapping
2. Confirmation dialog appears
3. User confirms deletion
4. Delete from `product_course_map` table
5. Refresh mappings list

---

## Order Processing and Enrollment Automation

### Workflow 5: Order Processing and Enrollment

**Trigger**: Customer purchases mapped product in Salla store

**Steps**:

1. **Customer Places Order**

   - Customer adds mapped product to cart
   - Customer completes checkout
   - Order created in Salla

2. **Salla Sends Webhook**

   - Event: `order.created`, `order.updated`, `order.customer.updated`, or `order.products.updated`
   - Endpoint: `POST /api/webhooks/salla`
   - Payload includes:
     - `merchant`: Merchant ID
     - `data`: Complete order object
       - `id`: Order ID
       - `customer`: Customer information
         - `email`: Customer email
         - `first_name`: First name
         - `last_name`: Last name
         - `mobile`: Mobile number
         - `mobile_code`: Mobile code
       - `items`: Order items array
         - `product.id`: Product ID
         - `product.name`: Product name
       - `status.name`: Order status

3. **Webhook Processing**

   - Extract merchant ID from payload
   - Find user by `salla_merchant_id`
   - Extract order data
   - Extract customer information

4. **Process Each Product in Order**

   - Loop through `items` array
   - For each item:
     - Extract product ID
     - Check if product has mapping:
       - Query `product_course_map`:
         - `user_id` = current user
         - `salla_product_id` = product ID
     - If no mapping: Skip product
     - If mapping exists: Continue

5. **Check Existing Sync Record**

   - Query `order_syncs`:
     - `user_id` = current user
     - `salla_order_id` = order ID
     - `salla_product_id` = product ID
   - If exists with final status ("تمت الاضافة الي الهام"): Skip
   - If exists: Update record
   - If not exists: Create new record

6. **Create/Update Order Sync Record**

   - Insert/Update `order_syncs`:
     - `user_id`
     - `salla_order_id`
     - `salla_product_id`
     - `salla_product_name`
     - `elham_course_id`
     - `elham_course_name`
     - `customer_email`
     - `customer_name`
     - `elham_team_id`
     - `status`: Current order status
     - `error_message`: null

7. **Check Order Status**

   - If status is NOT "تم التنفيذ" (Completed):
     - Store status and wait
     - Process will continue when status changes
   - If status IS "تم التنفيذ":
     - Continue to enrollment process

8. **Enrollment Process** (Status = "تم التنفيذ")

   - Decrypt Elham API key
   - **Get or Create Trainee**:
     - Try to get trainee by email:
       - Call: `GET /api/v1/integration/trainees/?email={email}`
     - If trainee exists: Use existing trainee ID
     - If trainee doesn't exist:
       - Create new trainee:
         - Call: `POST /api/v1/integration/trainees/`
         - Body:
           - `email`: Customer email
           - `mobile`: Full mobile number (code + number)
           - `first_name`: Customer first name
           - `last_name`: Customer last name
         - Get trainee ID from response

9. **Add Trainee to Team**

   - Call Elham API:
     - Endpoint: `POST /api/v1/integration/teams/add-trainees/`
     - Body:
       - `team`: Team ID from mapping
       - `trainees`: Array with trainee ID
       - `clear`: false (don't clear existing trainees)

10. **Update Sync Status**

    - Update `order_syncs`:
      - `elham_trainee_id`: Trainee ID
      - `status`: "تمت الاضافة الي المجموعة داخل الهام"
      - `error_message`: null
      - `updated_at`: Current timestamp

11. **Error Handling** (If enrollment fails)

    - Catch any errors
    - Update `order_syncs`:
      - `status`: Keep order status
      - `error_message`: Error message
      - `updated_at`: Current timestamp
    - Continue processing other products

12. **Return Success Response**
    - Return `{ success: true }` to Salla
    - Webhook processing complete

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 217-426)

**Status Flow**:

```
Order Created → order_syncs created with order status
    ↓
Order Status Changes → order_syncs updated
    ↓
Status = "تم التنفيذ" → Enrollment triggered
    ↓
Trainee Created/Found → Trainee added to team
    ↓
Status = "تمت الاضافة الي المجموعة داخل الهام" → Complete
```

**Error Scenarios**:

- Elham API key missing → Error logged, status unchanged
- Trainee creation fails → Error logged, status unchanged
- Team doesn't exist → Error logged, status unchanged
- Network error → Error logged, status unchanged

---

## Refund and Cancellation Handling

### Workflow 6: Order Refund/Cancellation

**Trigger**: Merchant refunds or cancels order in Salla

**Steps**:

1. **Merchant Refunds/Cancels Order**

   - Merchant processes refund in Salla dashboard
   - Or merchant cancels order
   - Order status changes in Salla

2. **Salla Sends Webhook**

   - Event: `order.refunded` or `order.cancelled`
   - Endpoint: `POST /api/webhooks/salla`
   - Payload includes:
     - `merchant`: Merchant ID
     - `data`: Order object with `id`

3. **Webhook Processing**

   - Extract merchant ID
   - Find user by `salla_merchant_id`
   - Extract order ID

4. **Find Order Sync Records**

   - Query `order_syncs`:
     - `user_id` = current user
     - `salla_order_id` = order ID
   - Get all records for this order
   - If no records found: Return success (nothing to process)

5. **Process Each Sync Record**

   - Loop through sync records
   - For each record:
     - Check if has `elham_team_id` and `elham_trainee_id`
     - If missing: Skip record (log and continue)
     - If present: Continue

6. **Remove Trainee from Team**

   - Decrypt Elham API key
   - Call Elham API:
     - Endpoint: `DELETE /api/v1/integration/teams/remove-trainee/{teamId}/{traineeId}/`
     - Removes trainee from team

7. **Update Sync Status**

   - Update `order_syncs`:
     - `status`: "ازالة المتدرب من المجموعه داخل الهام"
     - `error_message`: null
     - `updated_at`: Current timestamp

8. **Error Handling** (If removal fails)

   - Catch any errors
   - Update `order_syncs`:
     - `status`: Keep previous status
     - `error_message`: Error message
     - `updated_at`: Current timestamp
   - Continue processing other records

9. **Return Success Response**
   - Return `{ success: true }` to Salla
   - Webhook processing complete

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 428-541)

**Important Notes**:

- Trainee is removed from team, not deleted from Elham
- Trainee account remains in Elham system
- Only team membership is removed
- Process continues even if one removal fails

---

## Email Notification Workflows

### Workflow 7: Welcome Email Sending

**Trigger**: New user account created via Salla webhook

**Steps**:

1. **User Account Created** (See Workflow 1C)

   - New user created in Supabase Auth
   - User record created in `users` table

2. **Generate Credentials**

   - Email: From merchant info
   - Password: Randomly generated 16-character password

3. **Compose Email**

   - **Subject**: "Welcome to Salla-Elham Integration - Your Account Credentials"
   - **HTML Content**:
     - Welcome message
     - Login credentials (email and password)
     - Login button/link
     - Support information
   - **Plain Text Content**: Same information in plain text

4. **Validate SMTP Configuration**

   - Check environment variables:
     - `SMTP_HOST`: Required
     - `SMTP_USER`: Required
     - `SMTP_PASS`: Required
     - `SMTP_FROM`: Optional (uses SMTP_USER if not set)
   - If missing: Throw error

5. **Create SMTP Transporter**

   - Use nodemailer
   - Configuration:
     - Host: `SMTP_HOST`
     - Port: 587
     - Secure: false (TLS)
     - Auth: `SMTP_USER` and `SMTP_PASS`

6. **Send Email**

   - From: `SMTP_FROM` or `SMTP_USER`
   - To: User email
   - Send via SMTP transporter
   - Include HTML and plain text versions

7. **Error Handling**
   - If email fails: Log error
   - **Important**: Email failure does NOT block user creation
   - User account is created even if email fails
   - Error logged for troubleshooting

**Code Reference**: `lib/email/send.ts`

**Email Template Features**:

- Responsive HTML design
- Branded styling
- Clear credential display
- Direct login link
- Security warning about password

**Error Scenarios**:

- SMTP configuration missing → Error logged, user creation continues
- SMTP connection failure → Error logged, user creation continues
- Invalid email address → Error logged, user creation continues

---

## Workflow Integration Points

### How Workflows Connect

```
User Registration/Login
    ↓
Salla Connection (OAuth)
    ↓
Elham API Key Configuration
    ↓
Product-to-Course Mapping
    ↓
Order Processing (Automatic)
    ↓
Enrollment (Automatic)
```

### State Transitions

**User States**:

- Not Registered → Registered → Logged In
- No Salla Connection → Salla Connected
- No Elham Key → Elham Key Configured
- No Mappings → Mappings Created

**Order States**:

- Order Created → Order Sync Record Created
- Order Status Changed → Sync Record Updated
- Order Completed → Enrollment Triggered
- Enrollment Success → Status Updated
- Order Refunded → Trainee Removed

---

_For business rules, see [Business Logic Documentation](./BUSINESS_LOGIC.md)_
_For technical details, see [Technical Documentation](./TECHNICAL_DOCUMENTATION.md)_
