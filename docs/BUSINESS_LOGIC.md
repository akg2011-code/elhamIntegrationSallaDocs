# Business Logic Documentation

## Business Model and Value Proposition

### Overview

The Salla-Elham Integration platform serves as a middleware solution that automates the connection between e-commerce transactions (Salla) and learning management (Elham Academy). The primary value proposition is eliminating manual work in course enrollment by automatically enrolling customers when they purchase specific products.

### Core Business Value

1. **Automation**: Reduces manual enrollment work from hours to zero
2. **Accuracy**: Eliminates human error in course assignments
3. **Scalability**: Handles unlimited enrollments without additional effort
4. **Real-time Processing**: Instant enrollment upon order completion
5. **Lifecycle Management**: Automatically handles refunds and cancellations

## User Personas

### Primary Persona: E-commerce Merchant

**Profile**:

- Operates a Salla store selling digital or physical products
- Offers courses or training through Elham Academy
- Wants to automate course enrollment based on product purchases
- May have multiple products mapped to different courses/teams

**Goals**:

- Automate course enrollment process
- Reduce manual administrative work
- Ensure customers get access to courses immediately after purchase
- Track enrollment status and order history

**Pain Points Solved**:

- Manual enrollment is time-consuming
- Risk of missing enrollments
- Difficulty tracking which customers purchased which courses
- Managing refunds and cancellations manually

### Secondary Persona: System Administrator

**Profile**:

- Manages the integration platform
- Configures API keys and connections
- Monitors system health and order processing
- Troubleshoots integration issues

**Goals**:

- Ensure reliable integration between platforms
- Monitor order processing and enrollment status
- Maintain security of API keys and tokens
- Track system performance

## Core Business Rules

### 1. User Account Creation

**Rule**: Users can be created in two ways:

- **Manual Registration**: User signs up via registration page with email and password
- **Automatic Creation**: User account is automatically created when Salla store is authorized via webhook

**Business Logic**:

- When Salla store authorizes the app, the system:
  1. Fetches merchant information from Salla API
  2. Extracts merchant email address
  3. Checks if user exists by email
  4. If user doesn't exist, creates new account with:
     - Email from Salla merchant info
     - Randomly generated 16-character password
     - Encrypted Salla access and refresh tokens
     - Salla merchant ID
  5. Sends welcome email with login credentials (for new users only)

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 23-214)

### 2. Product-to-Course Mapping

**Rule**: Each Salla product can be mapped to exactly one Elham course/team combination.

**Business Logic**:

- Mapping is unique per user and per product
- One product can only map to one course/team
- Multiple products can map to the same course/team
- Mapping includes:
  - Salla product ID
  - Elham course ID (LX ID)
  - Elham team ID
  - Product and course names/images (for display)

**Constraints**:

- User must have connected Salla store (have access token)
- User must have configured Elham API key
- Product must exist in Salla store
- Course/team must exist in Elham Academy

**Code Reference**: `app/(dashboard)/mapping/page.tsx` (lines 190-281)

### 3. Order Processing and Enrollment

**Rule**: Customers are automatically enrolled in courses when orders containing mapped products reach "تم التنفيذ" (Completed) status.

**Business Logic Flow**:

1. **Order Created/Updated Event**:

   - Webhook receives order event from Salla
   - System identifies merchant from webhook payload
   - For each product in the order:
     - Check if product has a mapping
     - If mapped, create/update order_syncs record
     - Store order status, product info, customer info

2. **Order Status Check**:

   - If order status is "تم التنفيذ" (Completed):
     - Decrypt Elham API key
     - Check if customer (trainee) exists in Elham by email
     - If trainee doesn't exist, create new trainee with:
       - Email
       - Mobile number (if available)
       - First name and last name
     - Add trainee to the mapped team
     - Update order_syncs status to "تمت الاضافة الي المجموعة داخل الهام"

3. **Error Handling**:
   - If enrollment fails, status remains as order status
   - Error message stored in order_syncs.error_message
   - System continues processing other products in the order

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 217-426)

### 4. Refund and Cancellation Handling

**Rule**: When an order is refunded or cancelled, trainees are automatically removed from the associated Elham team.

**Business Logic**:

- Webhook receives `order.refunded` or `order.cancelled` event
- System finds all order_syncs records for the order
- For each record with team_id and trainee_id:
  - Remove trainee from team via Elham API
  - Update status to "ازالة المتدرب من المجموعه داخل الهام"
- If removal fails, error is logged but process continues

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 428-541)

### 5. Duplicate Processing Prevention

**Rule**: System prevents duplicate enrollments for the same order/product combination.

**Business Logic**:

- Before processing enrollment, check if order_syncs record exists
- If record exists with final status ("تمت الاضافة الي الهام"), skip processing
- This prevents re-processing the same order multiple times

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 299-311)

### 6. Team-Based Enrollment

**Rule**: Enrollment is done at the team level, not individual course level.

**Business Logic**:

- Elham uses a team structure where teams contain multiple courses (LXs)
- When a product is mapped, it maps to:
  - A specific course (LX ID) for identification
  - A team ID for enrollment
- Trainees are added to teams, giving them access to all courses in that team
- This allows for bundled course access

**Code Reference**: `lib/elham/api.ts` (lines 367-389)

## Decision Trees

### Order Processing Decision Tree

```
Order Event Received
    │
    ├─> Is merchant ID valid?
    │   ├─> No → Return 404 Error
    │   └─> Yes → Continue
    │
    ├─> For each product in order:
    │   │
    │   ├─> Does product have mapping?
    │   │   ├─> No → Skip product
    │   │   └─> Yes → Continue
    │   │
    │   ├─> Has this order/product been processed?
    │   │   ├─> Yes (final status) → Skip
    │   │   └─> No → Continue
    │   │
    │   ├─> Create/Update order_syncs record
    │   │
    │   └─> Is order status "تم التنفيذ"?
    │       ├─> No → Store status, wait
    │       └─> Yes → Process enrollment
    │           │
    │           ├─> Does trainee exist? (by email)
    │           │   ├─> No → Create trainee
    │           │   └─> Yes → Use existing
    │           │
    │           └─> Add trainee to team
    │               ├─> Success → Update status
    │               └─> Error → Log error, keep status
```

### User Creation Decision Tree

```
Salla Authorization Webhook
    │
    ├─> Extract merchant email
    │
    ├─> Does user exist? (by email)
    │   ├─> Yes → Update tokens and merchant ID
    │   └─> No → Create new user
    │       │
    │       ├─> Generate random password
    │       ├─> Create Supabase Auth user
    │       ├─> Create users table record
    │       ├─> Encrypt and store tokens
    │       └─> Send welcome email
```

## Business Workflows Overview

### Workflow 1: Initial Setup

1. Merchant installs Salla app → Webhook creates/updates user account
2. Merchant logs in with credentials (from email or manual registration)
3. Merchant adds Elham API key in Settings
4. Merchant maps products to courses in Mapping page
5. System is ready for automatic enrollments

### Workflow 2: Order Processing

1. Customer purchases mapped product in Salla
2. Salla sends webhook to integration platform
3. System creates order_syncs record
4. When order status becomes "تم التنفيذ":
   - System creates/finds trainee in Elham
   - System adds trainee to mapped team
   - Status updated to success

### Workflow 3: Refund Processing

1. Merchant refunds/cancels order in Salla
2. Salla sends webhook to integration platform
3. System finds all order_syncs for the order
4. System removes trainee from team in Elham
5. Status updated to reflect removal

## Data Flow

### Enrollment Data Flow

```
Salla Order
    ↓
Webhook Event
    ↓
Integration Platform
    ↓
Check Product Mapping
    ↓
Create/Update order_syncs
    ↓
Order Status = "تم التنفيذ"?
    ↓ Yes
Get/Create Trainee in Elham
    ↓
Add Trainee to Team
    ↓
Update order_syncs Status
```

### User Data Flow

```
Salla Authorization
    ↓
Fetch Merchant Info (API)
    ↓
Extract Email
    ↓
Check User Exists
    ↓
Create/Update User
    ↓
Encrypt Tokens
    ↓
Store in Database
    ↓
Send Welcome Email (if new)
```

## Business Metrics and Tracking

### Key Metrics Tracked

1. **Order Syncs**: All order processing attempts are logged in `order_syncs` table
2. **Status Tracking**: Each sync has a status indicating:
   - Order status (from Salla)
   - Enrollment status (success/failure)
   - Error messages (if any)
3. **Trainee IDs**: Links Salla customers to Elham trainees
4. **Timestamps**: Created and updated timestamps for audit trail

### Status Values

- **Order Statuses**: Direct from Salla (e.g., "تم التنفيذ")
- **Enrollment Statuses**:
  - "تمت الاضافة الي المجموعة داخل الهام" - Successfully added to team
  - "ازالة المتدرب من المجموعه داخل الهام" - Successfully removed from team
  - Error messages stored in `error_message` field

## Business Constraints

1. **One-to-One Mapping**: One product maps to one course/team
2. **User Isolation**: Each user can only see and manage their own data (RLS)
3. **API Key Required**: Elham API key must be configured before mapping
4. **Salla Connection Required**: Salla store must be connected before mapping
5. **Team-Based Only**: Enrollment is always at team level, not individual course

## Error Handling Business Rules

1. **Non-Critical Errors**: System continues processing other products if one fails
2. **Error Logging**: All errors stored in order_syncs.error_message
3. **Retry Logic**: Manual retry possible by re-processing order (if status changes)
4. **Email Errors**: Email sending failures don't block user creation
5. **API Failures**: Elham API failures are logged but don't crash the system

---

_For technical implementation details, see [Technical Documentation](./TECHNICAL_DOCUMENTATION.md)_
_For detailed workflows, see [Workflows Documentation](./WORKFLOWS.md)_
