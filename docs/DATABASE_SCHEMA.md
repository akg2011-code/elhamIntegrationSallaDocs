# Database Schema Documentation

## Overview

The Salla-Elham Integration uses Supabase (PostgreSQL) as its database. The schema is designed with Row Level Security (RLS) to ensure users can only access their own data. All tables use UUID primary keys and include automatic timestamp management.

## Entity Relationship Diagram

```
┌─────────────────┐
│  auth.users     │ (Supabase Auth)
└────────┬────────┘
         │
         │ 1:1
         │
┌────────▼────────┐
│     users       │
│  (Main Table)   │
└────────┬────────┘
         │
         │ 1:N
         ├──────────────────┬──────────────────┬──────────────────┐
         │                  │                  │                  │
┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
│ elham_courses   │ │product_course_  │ │  order_syncs    │
│                 │ │     map         │ │                  │
└─────────────────┘ └─────────────────┘ └──────────────────┘
```

## Tables

### 1. users

**Purpose**: Stores user account information, encrypted API keys, and Salla OAuth tokens.

**Schema**:

| Column                    | Type                     | Constraints                                              | Description                             |
| ------------------------- | ------------------------ | -------------------------------------------------------- | --------------------------------------- |
| `id`                      | UUID                     | PRIMARY KEY, REFERENCES auth.users(id) ON DELETE CASCADE | User ID from Supabase Auth              |
| `email`                   | TEXT                     | NOT NULL                                                 | User email address                      |
| `elham_api_key_encrypted` | TEXT                     | NULLABLE                                                 | Encrypted Elham API key (AES encrypted) |
| `salla_merchant_id`       | TEXT                     | NULLABLE                                                 | Salla merchant/store ID                 |
| `salla_access_token`      | TEXT                     | NULLABLE                                                 | Encrypted Salla OAuth access token      |
| `salla_refresh_token`     | TEXT                     | NULLABLE                                                 | Encrypted Salla OAuth refresh token     |
| `created_at`              | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                                  | Record creation timestamp               |
| `updated_at`              | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                                  | Record last update timestamp            |

**Indexes**:

- `idx_users_email` on `email` - For email lookups

**Relationships**:

- One-to-one with `auth.users` (Supabase Auth)
- One-to-many with `elham_courses`
- One-to-many with `product_course_map`
- One-to-many with `order_syncs`

**RLS Policies**:

- **SELECT**: Users can view their own data (`auth.uid() = id`)
- **UPDATE**: Users can update their own data (`auth.uid() = id`)

**Triggers**:

- `update_users_updated_at` - Automatically updates `updated_at` on UPDATE

**Usage Notes**:

- `id` must match `auth.users.id` (created by Supabase Auth)
- All encrypted fields use AES encryption
- `salla_merchant_id` is used to identify user from Salla webhooks
- Email is used for user lookup during Salla authorization

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 2-11)

---

### 2. elham_courses

**Purpose**: Stores cached Elham course/journey information. Note: This table may not be actively used as courses are fetched directly from Elham API.

**Schema**:

| Column            | Type                     | Constraints                                      | Description                         |
| ----------------- | ------------------------ | ------------------------------------------------ | ----------------------------------- |
| `id`              | UUID                     | PRIMARY KEY, DEFAULT gen_random_uuid()           | Internal course record ID           |
| `user_id`         | UUID                     | NOT NULL, REFERENCES users(id) ON DELETE CASCADE | Owner user ID                       |
| `elham_course_id` | TEXT                     | NOT NULL                                         | Elham course/journey ID (LX ID)     |
| `name`            | TEXT                     | NOT NULL                                         | Course name                         |
| `description`     | TEXT                     | NULLABLE                                         | Course description                  |
| `raw_data`        | JSONB                    | NOT NULL                                         | Complete course data from Elham API |
| `created_at`      | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                          | Record creation timestamp           |
| `updated_at`      | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                          | Record last update timestamp        |

**Unique Constraints**:

- `UNIQUE(user_id, elham_course_id)` - One course per user

**Indexes**:

- `idx_elham_courses_user_id` on `user_id` - For user-specific queries

**Relationships**:

- Many-to-one with `users` (via `user_id`)

**RLS Policies**:

- **SELECT**: Users can view their own courses (`auth.uid() = user_id`)
- **INSERT**: Users can insert their own courses (`auth.uid() = user_id`)
- **UPDATE**: Users can update their own courses (`auth.uid() = user_id`)
- **DELETE**: Users can delete their own courses (`auth.uid() = user_id`)

**Triggers**:

- `update_elham_courses_updated_at` - Automatically updates `updated_at` on UPDATE

**Usage Notes**:

- This table may be used for caching, but courses are primarily fetched from Elham API
- `raw_data` stores complete JSON response from Elham API
- `elham_course_id` corresponds to Elham's LX (Learning Experience) ID

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 13-24)

---

### 3. product_course_map

**Purpose**: Maps Salla products to Elham courses/teams. This is the core mapping table that enables automatic enrollment.

**Schema**:

| Column                | Type                     | Constraints                                      | Description                      |
| --------------------- | ------------------------ | ------------------------------------------------ | -------------------------------- |
| `id`                  | UUID                     | PRIMARY KEY, DEFAULT gen_random_uuid()           | Mapping record ID                |
| `user_id`             | UUID                     | NOT NULL, REFERENCES users(id) ON DELETE CASCADE | Owner user ID                    |
| `salla_product_id`    | TEXT                     | NOT NULL                                         | Salla product ID                 |
| `elham_course_id`     | TEXT                     | NOT NULL                                         | Elham course/journey ID (LX ID)  |
| `elham_team_id`       | INTEGER                  | NULLABLE                                         | Elham team ID (for enrollment)   |
| `elham_team_name`     | TEXT                     | NULLABLE                                         | Elham team name (for display)    |
| `elham_course_name`   | TEXT                     | NULLABLE                                         | Elham course name (for display)  |
| `elham_course_image`  | TEXT                     | NULLABLE                                         | Elham course image URL           |
| `salla_product_name`  | TEXT                     | NULLABLE                                         | Salla product name (for display) |
| `salla_product_image` | TEXT                     | NULLABLE                                         | Salla product image URL          |
| `created_at`          | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                          | Record creation timestamp        |
| `updated_at`          | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                          | Record last update timestamp     |

**Unique Constraints**:

- `UNIQUE(user_id, salla_product_id)` - One mapping per product per user

**Indexes**:

- `idx_product_course_map_user_id` on `user_id` - For user-specific queries
- `idx_product_course_map_salla_product_id` on `salla_product_id` - For product lookups

**Relationships**:

- Many-to-one with `users` (via `user_id`)

**RLS Policies**:

- **SELECT**: Users can view their own mappings (`auth.uid() = user_id`)
- **INSERT**: Users can insert their own mappings (`auth.uid() = user_id`)
- **UPDATE**: Users can update their own mappings (`auth.uid() = user_id`)
- **DELETE**: Users can delete their own mappings (`auth.uid() = user_id`)

**Triggers**:

- `update_product_course_map_updated_at` - Automatically updates `updated_at` on UPDATE

**Business Rules**:

- One Salla product can map to one Elham course/team per user
- `elham_team_id` is required for enrollment (enrollment happens at team level)
- `elham_course_id` identifies the specific course (LX)
- Display names and images stored for UI purposes

**Migration History**:

- Initial: `20240101000001_initial_schema.sql` - Created table with basic fields
- Migration 2: `20240102000001_add_mapping_names_and_images.sql` - Added name and image columns
- Migration 3: `20240103000001_add_team_columns.sql` - Added team ID and name columns

**Code Reference**:

- `supabase/migrations/20240101000001_initial_schema.sql` (lines 26-35)
- `supabase/migrations/20240102000001_add_mapping_names_and_images.sql`
- `supabase/migrations/20240103000001_add_team_columns.sql`

---

### 4. order_syncs

**Purpose**: Tracks order processing and enrollment status. This table logs all order events and their processing results.

**Schema**:

| Column               | Type                     | Constraints                                      | Description                         |
| -------------------- | ------------------------ | ------------------------------------------------ | ----------------------------------- |
| `id`                 | UUID                     | PRIMARY KEY, DEFAULT gen_random_uuid()           | Sync record ID                      |
| `user_id`            | UUID                     | NOT NULL, REFERENCES users(id) ON DELETE CASCADE | Owner user ID                       |
| `salla_order_id`     | TEXT                     | NOT NULL                                         | Salla order ID                      |
| `salla_product_id`   | TEXT                     | NOT NULL                                         | Salla product ID from order         |
| `salla_product_name` | TEXT                     | NULLABLE                                         | Salla product name (for display)    |
| `elham_course_id`    | TEXT                     | NOT NULL                                         | Elham course ID (from mapping)      |
| `elham_course_name`  | TEXT                     | NULLABLE                                         | Elham course name (for display)     |
| `elham_team_id`      | INTEGER                  | NULLABLE                                         | Elham team ID (from mapping)        |
| `elham_trainee_id`   | INTEGER                  | NULLABLE                                         | Elham trainee ID (after enrollment) |
| `customer_email`     | TEXT                     | NOT NULL                                         | Customer email from order           |
| `customer_name`      | TEXT                     | NULLABLE                                         | Customer full name                  |
| `status`             | TEXT                     | NOT NULL                                         | Order/enrollment status             |
| `error_message`      | TEXT                     | NULLABLE                                         | Error message if processing failed  |
| `created_at`         | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                          | Record creation timestamp           |
| `updated_at`         | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW()                          | Record last update timestamp        |

**Indexes**:

- `idx_order_syncs_user_id` on `user_id` - For user-specific queries
- `idx_order_syncs_salla_order_id` on `salla_order_id` - For order lookups

**Relationships**:

- Many-to-one with `users` (via `user_id`)

**RLS Policies**:

- **SELECT**: Users can view their own syncs (`auth.uid() = user_id`)
- **INSERT**:
  - Users can insert their own syncs (`auth.uid() = user_id`)
  - Service role can insert syncs (for webhook processing) (`true`)
- **UPDATE**: Service role can update syncs (for webhook processing) (`true`)

**Triggers**:

- `update_order_syncs_updated_at` - Automatically updates `updated_at` on UPDATE

**Status Values**:

- Order statuses from Salla (e.g., "تم التنفيذ" - Completed)
- Enrollment statuses:
  - "تمت الاضافة الي المجموعة داخل الهام" - Successfully added to team
  - "ازالة المتدرب من المجموعه داخل الهام" - Successfully removed from team
- Error messages stored in `error_message` field

**Business Rules**:

- One record per order/product combination
- Status tracks both order status and enrollment status
- `elham_trainee_id` populated after trainee is created/found
- `elham_team_id` comes from product mapping
- Used to prevent duplicate processing

**Migration History**:

- Initial: `20240101000001_initial_schema.sql` - Created table with basic fields and status enum
- Migration 4: `20240104000001_update_order_syncs_schema.sql` - Added display fields and removed status enum constraint
- Migration 5: `20240105000001_add_elham_trainee_id_to_order_syncs.sql` - Added trainee ID column

**Code Reference**:

- `supabase/migrations/20240101000001_initial_schema.sql` (lines 37-49)
- `supabase/migrations/20240104000001_update_order_syncs_schema.sql`
- `supabase/migrations/20240105000001_add_elham_trainee_id_to_order_syncs.sql`

---

## Database Functions

### update_updated_at_column()

**Purpose**: Automatically updates the `updated_at` timestamp when a record is updated.

**Implementation**:

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = TIMEZONE('utc'::text, NOW());
  RETURN NEW;
END;
$$ language 'plpgsql';
```

**Usage**: Applied as a trigger on all tables with `updated_at` columns.

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 112-118)

---

## Triggers

All tables with `updated_at` columns have triggers that automatically update the timestamp on UPDATE:

1. `update_users_updated_at` - On `users` table
2. `update_elham_courses_updated_at` - On `elham_courses` table
3. `update_product_course_map_updated_at` - On `product_course_map` table
4. `update_order_syncs_updated_at` - On `order_syncs` table

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 120-131)

---

## Row Level Security (RLS)

### Overview

All tables have RLS enabled to ensure users can only access their own data. Policies use `auth.uid()` to identify the current authenticated user.

### Policy Patterns

1. **User-Specific Access**: All policies check `auth.uid() = user_id` or `auth.uid() = id`
2. **Service Role Access**: `order_syncs` allows service role to insert/update for webhook processing
3. **Cascade Deletes**: All foreign keys use `ON DELETE CASCADE` to maintain referential integrity

### Policy Details

**users**:

- SELECT: Users can view own data
- UPDATE: Users can update own data

**elham_courses**:

- SELECT, INSERT, UPDATE, DELETE: Users can manage own courses

**product_course_map**:

- SELECT, INSERT, UPDATE, DELETE: Users can manage own mappings

**order_syncs**:

- SELECT: Users can view own syncs
- INSERT: Users and service role can insert
- UPDATE: Service role can update (for webhook processing)

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 59-109)

---

## Indexes

### Performance Optimization

Indexes are created on frequently queried columns:

1. **users**:

   - `idx_users_email` on `email` - For email-based lookups

2. **elham_courses**:

   - `idx_elham_courses_user_id` on `user_id` - For user-specific queries

3. **product_course_map**:

   - `idx_product_course_map_user_id` on `user_id` - For user-specific queries
   - `idx_product_course_map_salla_product_id` on `salla_product_id` - For product lookups during order processing

4. **order_syncs**:
   - `idx_order_syncs_user_id` on `user_id` - For user-specific queries
   - `idx_order_syncs_salla_order_id` on `salla_order_id` - For order lookups

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 51-57)

---

## Migration History

### Migration 1: Initial Schema (20240101000001_initial_schema.sql)

**Created**:

- `users` table
- `elham_courses` table
- `product_course_map` table
- `order_syncs` table (with status enum constraint)
- All indexes
- All RLS policies
- All triggers
- `update_updated_at_column()` function

### Migration 2: Add Mapping Names and Images (20240102000001_add_mapping_names_and_images.sql)

**Added to `product_course_map`**:

- `elham_course_name`
- `elham_course_image`
- `salla_product_name`
- `salla_product_image`

**Purpose**: Store display names and images for UI purposes.

### Migration 3: Add Team Columns (20240103000001_add_team_columns.sql)

**Added to `product_course_map`**:

- `elham_team_id` (INTEGER)
- `elham_team_name` (TEXT)

**Purpose**: Support team-based enrollment in Elham.

### Migration 4: Update Order Syncs Schema (20240104000001_update_order_syncs_schema.sql)

**Added to `order_syncs`**:

- `salla_product_name`
- `elham_course_name`
- `customer_name`
- `elham_team_id`

**Removed**:

- Status enum constraint (to allow Salla status names)

**Purpose**: Store display information and support flexible status values.

### Migration 5: Add Trainee ID (20240105000001_add_elham_trainee_id_to_order_syncs.sql)

**Added to `order_syncs`**:

- `elham_trainee_id` (INTEGER)

**Purpose**: Link order syncs to Elham trainees for refund/cancellation processing.

---

## Data Flow

### Enrollment Data Flow

```
Salla Order
    ↓
Webhook → order_syncs (created)
    ↓
Check product_course_map (by salla_product_id)
    ↓
Get elham_team_id from mapping
    ↓
Enroll trainee → order_syncs (updated with trainee_id)
```

### User Data Flow

```
Supabase Auth → auth.users
    ↓
Create users record (linked via id)
    ↓
Store encrypted tokens/keys
```

### Mapping Data Flow

```
User creates mapping
    ↓
product_course_map (inserted)
    ↓
Used during order processing
```

---

## Constraints and Validations

### Foreign Key Constraints

- All `user_id` columns reference `users(id)` with `ON DELETE CASCADE`
- `users.id` references `auth.users(id)` with `ON DELETE CASCADE`

### Unique Constraints

- `users`: Email uniqueness (enforced by Supabase Auth)
- `elham_courses`: `UNIQUE(user_id, elham_course_id)`
- `product_course_map`: `UNIQUE(user_id, salla_product_id)`

### Data Integrity

- Cascade deletes ensure referential integrity
- Timestamps automatically maintained via triggers
- RLS ensures data isolation between users

---

## Query Patterns

### Common Queries

1. **Get user by email**:

   ```sql
   SELECT * FROM users WHERE email = $1;
   ```

2. **Get mappings for user**:

   ```sql
   SELECT * FROM product_course_map WHERE user_id = $1;
   ```

3. **Get order syncs for user**:

   ```sql
   SELECT * FROM order_syncs WHERE user_id = $1 ORDER BY created_at DESC;
   ```

4. **Check if product has mapping**:

   ```sql
   SELECT * FROM product_course_map
   WHERE user_id = $1 AND salla_product_id = $2;
   ```

5. **Get order sync by order and product**:
   ```sql
   SELECT * FROM order_syncs
   WHERE user_id = $1
     AND salla_order_id = $2
     AND salla_product_id = $3;
   ```

---

## Security Considerations

1. **Encrypted Fields**: API keys and tokens stored encrypted
2. **RLS**: Database-level access control
3. **Service Role**: Only used server-side, never exposed to client
4. **Cascade Deletes**: Maintains data integrity
5. **No Direct Access**: All access through Supabase client with RLS

---

_For business logic, see [Business Logic Documentation](./BUSINESS_LOGIC.md)_
_For API details, see [API Documentation](./API_DOCUMENTATION.md)_
_For workflows, see [Workflows Documentation](./WORKFLOWS.md)_
