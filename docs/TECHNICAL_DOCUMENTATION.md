# Technical Documentation

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Browser                        │
│  (React Components, Next.js Pages, Tailwind CSS)            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ HTTP/HTTPS
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   Next.js Application                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Frontend (App Router)                                │  │
│  │  - Authentication Pages                               │  │
│  │  - Dashboard Pages                                    │  │
│  │  - Settings & Mapping Pages                           │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  API Routes (Server Actions)                         │  │
│  │  - /api/webhooks/salla                                │  │
│  │  - /api/elham/courses                                 │  │
│  │  - /api/elham/teams                                   │  │
│  │  - /api/auth/logout                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Library Modules                                      │  │
│  │  - Supabase Clients (server/client/admin)            │  │
│  │  - Salla API Client                                   │  │
│  │  - Elham API Client                                   │  │
│  │  - Encryption Utilities                               │  │
│  │  - Email Service                                      │  │
│  └──────────────────────────────────────────────────────┘  │
└────────┬───────────────────────────────┬────────────────────┘
         │                               │
         │                               │
┌────────▼────────┐            ┌───────▼────────┐
│   Supabase      │            │  External APIs   │
│   (PostgreSQL)  │            │  - Salla OAuth   │
│   - Users       │            │  - Elham REST    │
│   - Mappings    │            │  - Webhooks      │
│   - Order Syncs │            └──────────────────┘
└─────────────────┘
```

### Component Architecture

#### Frontend Components

- **Pages**: Next.js App Router pages in `app/` directory
  - Authentication: `app/(auth)/login`, `app/(auth)/register`
  - Dashboard: `app/(dashboard)/dashboard`, `app/(dashboard)/settings`, `app/(dashboard)/mapping`
- **UI Components**: Reusable components in `components/ui/`
  - Button, Dialog, Dropdown, Input, Select (Radix UI based)
- **Context Providers**: React Context for state management
  - `CoursesContext`: Manages Elham courses data
  - `DirectionProvider`: Handles RTL/LTR text direction

#### Backend Services

- **API Routes**: Next.js API routes in `app/api/`
- **Library Modules**: Utility modules in `lib/`
  - Supabase clients (server, client, admin)
  - API clients (Salla, Elham)
  - Encryption utilities
  - Email service
  - Internationalization

## Technology Stack

### Core Framework

- **Next.js 16.0.1**: React framework with App Router
- **React 19.2.0**: UI library
- **TypeScript 5.9.3**: Type-safe JavaScript

### Database & Backend

- **Supabase**: PostgreSQL database with:
  - Authentication (Supabase Auth)
  - Row Level Security (RLS)
  - Real-time capabilities
- **@supabase/ssr 0.7.0**: Server-side rendering support
- **@supabase/supabase-js 2.78.0**: JavaScript client

### UI & Styling

- **Tailwind CSS 4**: Utility-first CSS framework
- **Radix UI**: Accessible component primitives
  - `@radix-ui/react-dialog`
  - `@radix-ui/react-dropdown-menu`
  - `@radix-ui/react-select`
  - `@radix-ui/react-direction`
- **lucide-react**: Icon library
- **tailwindcss-animate**: Animation utilities

### Internationalization

- **next-intl 4.4.0**: Internationalization framework
- **Cookie-based locale**: Locale stored in cookies
- **Supported Languages**: Arabic (ar), English (en)

### Security & Encryption

- **crypto-js 4.2.0**: AES encryption for API keys
- **js-cookie 3.0.5**: Cookie management

### Email Service

- **nodemailer 7.0.10**: SMTP email sending
- **@types/nodemailer 7.0.3**: TypeScript types

### Development Tools

- **ESLint 9**: Code linting
- **eslint-config-next**: Next.js ESLint configuration
- **TypeScript**: Type checking

## Security Implementation

### Authentication & Authorization

#### Supabase Auth Integration

- **Email/Password Authentication**: Standard email and password login
- **Session Management**: Handled by Supabase Auth
- **Server-Side Sessions**: Using `@supabase/ssr` for secure server-side session handling

**Implementation**:
- Client-side: `lib/supabase/client.ts` - Creates Supabase client for browser
- Server-side: `lib/supabase/server.ts` - Creates Supabase client for server components
- Admin: `lib/supabase/admin.ts` - Creates admin client with service role key

**Code Reference**: 
- `lib/supabase/client.ts`
- `lib/supabase/server.ts`
- `lib/supabase/admin.ts`

#### Row Level Security (RLS)

All database tables have RLS enabled with policies:

- **Users Table**: Users can only view/update their own data
- **Product Course Map**: Users can only manage their own mappings
- **Order Syncs**: Users can only view their own sync records
- **Service Role Access**: Admin client can insert/update order_syncs for webhook processing

**Code Reference**: `supabase/migrations/20240101000001_initial_schema.sql` (lines 59-109)

### Encryption

#### API Key Encryption

**Algorithm**: AES (Advanced Encryption Standard)
**Library**: crypto-js
**Secret**: Stored in environment variables (`ENCRYPTION_SECRET` or `NEXT_PUBLIC_ENCRYPTION_SECRET`)

**Implementation**:
- **Encryption**: `encryptApiKey(key: string)` - Encrypts API keys before storage
- **Decryption**: `decryptApiKey(encrypted: string)` - Decrypts API keys when needed
- **Storage**: Encrypted values stored in database
- **Usage**: API keys are only decrypted when making API calls

**Code Reference**: `lib/crypto/encrypt.ts`

**Security Considerations**:
- Encryption secret must be kept secure
- Different secrets for client/server (optional)
- Never log decrypted API keys
- API keys decrypted only when needed

#### Token Encryption

Salla OAuth tokens (access_token, refresh_token) are encrypted before storage:
- Tokens encrypted using same AES encryption
- Stored in `users` table
- Decrypted only when making Salla API calls

### Environment Variables

**Required Variables**:
```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# Encryption
ENCRYPTION_SECRET=your_encryption_secret
NEXT_PUBLIC_ENCRYPTION_SECRET=your_encryption_secret

# Salla (optional, for webhook verification)
SALLA_CLIENT_ID=your_client_id
SALLA_CLIENT_SECRET=your_client_secret

# Email (SMTP)
SMTP_HOST=smtp.example.com
SMTP_USER=your_smtp_user
SMTP_PASS=your_smtp_password
SMTP_FROM=noreply@example.com

# App URL
NEXT_PUBLIC_APP_URL=https://your-app-url.com
```

**Security Best Practices**:
- Never commit `.env.local` to version control
- Use different secrets for development and production
- Rotate secrets periodically
- Use strong, random encryption secrets

### Data Protection

1. **Encrypted Storage**: All sensitive data (API keys, tokens) encrypted at rest
2. **RLS Policies**: Database-level access control
3. **HTTPS**: Required for production (handled by deployment platform)
4. **No Client-Side Secrets**: Service role key never exposed to client
5. **Secure Cookies**: Session cookies managed by Supabase Auth

## Authentication and Authorization Flows

### User Registration Flow

```
User → Registration Page
    ↓
Enter Email & Password
    ↓
Supabase Auth.createUser()
    ↓
Create users table record
    ↓
Redirect to Dashboard
```

**Code Reference**: `app/(auth)/register/page.tsx`

### User Login Flow

```
User → Login Page
    ↓
Enter Email & Password
    ↓
Supabase Auth.signInWithPassword()
    ↓
Session Created
    ↓
Redirect to Dashboard
```

**Code Reference**: `app/(auth)/login/page.tsx`

### Automatic User Creation Flow

```
Salla Webhook → app.store.authorize
    ↓
Fetch Merchant Info (Salla API)
    ↓
Extract Email
    ↓
Check User Exists
    ↓
Create User (if new)
    ↓
Encrypt & Store Tokens
    ↓
Send Welcome Email
```

**Code Reference**: `app/api/webhooks/salla/route.ts` (lines 23-214)

### Protected Route Access

- **Middleware**: Supabase middleware checks authentication
- **Server Components**: Use `createClient()` from `lib/supabase/server.ts`
- **Client Components**: Use `createClient()` from `lib/supabase/client.ts`
- **API Routes**: Check authentication before processing

**Code Reference**: `lib/supabase/middleware.ts`

## Error Handling Strategies

### API Error Handling

1. **Try-Catch Blocks**: All async operations wrapped in try-catch
2. **Error Logging**: Errors logged to console with context
3. **User-Friendly Messages**: Errors translated and shown to users
4. **Graceful Degradation**: System continues operating when non-critical errors occur

### Webhook Error Handling

- **Validation Errors**: Return 400 with error message
- **Processing Errors**: Log error, update order_syncs with error message
- **Continue Processing**: If one product fails, continue with others
- **Status Tracking**: All errors stored in order_syncs.error_message

**Code Reference**: `app/api/webhooks/salla/route.ts`

### Database Error Handling

- **RLS Violations**: Caught and return appropriate error
- **Constraint Violations**: Handled with user-friendly messages
- **Connection Errors**: Logged and retried if possible

### Email Error Handling

- **SMTP Errors**: Logged but don't block user creation
- **Validation**: SMTP config validated before sending
- **Fallback**: Email failures don't prevent account creation

**Code Reference**: `lib/email/send.ts`

## Internationalization (i18n) Implementation

### Architecture

- **Framework**: next-intl
- **Locale Storage**: Cookies
- **Supported Locales**: Arabic (ar), English (en)
- **Default Locale**: Arabic

### Implementation Details

1. **Locale Detection**: 
   - Server-side: `lib/i18n/request.ts` - Gets locale from cookies
   - Client-side: `lib/locale-client.ts` - Client-side locale management

2. **Translation Files**:
   - `locales/ar.json` - Arabic translations
   - `locales/en.json` - English translations

3. **Usage**:
   - Server Components: `await getTranslations()`
   - Client Components: `useTranslations()`
   - API Routes: `await getLocale()`

**Code Reference**:
- `lib/i18n/config.ts`
- `lib/i18n/messages.ts`
- `lib/i18n/request.ts`
- `i18n.ts`

### Direction Support

- **RTL Support**: Arabic uses right-to-left layout
- **Component**: `DirectionProvider` handles text direction
- **Implementation**: Uses `@radix-ui/react-direction`

**Code Reference**: `components/DirectionProvider.tsx`

## External API Integrations

### Salla API Integration

**Base URL**: `https://api.salla.dev`
**Authentication**: OAuth 2.0 Bearer Token

**Endpoints Used**:
- `GET /admin/v2/products` - Fetch products
- `GET /admin/v2/products/{id}` - Get product by ID
- `GET /oauth2/user/info` - Get merchant info

**Webhook Events**:
- `app.store.authorize` - Store authorization
- `order.created` - Order created
- `order.updated` - Order updated
- `order.customer.updated` - Customer updated
- `order.products.updated` - Products updated
- `order.refunded` - Order refunded
- `order.cancelled` - Order cancelled

**Token Management**:
- Access tokens encrypted before storage
- Refresh tokens stored for token renewal
- Token refresh implemented (if needed)

**Code Reference**: `lib/salla/api.ts`

### Elham API Integration

**Base URLs**:
- `https://api-ksa.elhamsol.com/api/v1/integration/` - Main integration API
- `https://api.elham.academy/api/v1/` - Legacy API (for enrollment)

**Authentication**: API Key in `API-KEY` header

**Endpoints Used**:
- `GET /lx/` - Fetch journeys (courses)
- `GET /teams/` - Fetch teams
- `GET /trainees/` - Get trainee by email
- `POST /trainees/` - Create trainee
- `POST /teams/add-trainees/` - Add trainees to team
- `DELETE /teams/remove-trainee/{teamId}/{traineeId}/` - Remove trainee from team

**Response Formats**:
- Paginated responses with `results` array
- Localized content (title_ar, title_en, description_ar, description_en)

**Code Reference**: `lib/elham/api.ts`

## Performance Considerations

### Database Queries

- **Indexes**: Created on frequently queried columns
  - `users.email`
  - `product_course_map.user_id`, `product_course_map.salla_product_id`
  - `order_syncs.user_id`, `order_syncs.salla_order_id`

- **RLS Optimization**: Policies optimized for user-specific queries

### API Calls

- **Caching**: Courses fetched and cached in React Context
- **Batching**: Multiple products processed in single webhook call
- **Error Retry**: Manual retry possible for failed enrollments

### Frontend Optimization

- **Code Splitting**: Next.js automatic code splitting
- **Image Optimization**: Next.js Image component
- **Lazy Loading**: Components loaded on demand

## Deployment Architecture

### Recommended Platform: Vercel

- **Framework**: Next.js optimized for Vercel
- **Environment Variables**: Set in Vercel dashboard
- **Database**: Supabase (external service)
- **HTTPS**: Automatic via Vercel

### Environment Setup

1. **Development**:
   - Local Supabase instance (optional)
   - `.env.local` for environment variables
   - `npm run dev` for development server

2. **Production**:
   - Vercel deployment
   - Supabase cloud instance
   - Environment variables in Vercel dashboard

## Monitoring and Logging

### Logging Strategy

- **Console Logging**: Used for development and debugging
- **Error Logging**: All errors logged with context
- **Webhook Logging**: Webhook events logged for debugging
- **Email Logging**: Email send status logged

### Monitoring Points

1. **Order Processing**: Track order_syncs table for processing status
2. **API Errors**: Monitor Elham and Salla API call failures
3. **User Activity**: Track user logins and actions
4. **Database Performance**: Monitor query performance via Supabase dashboard

## Troubleshooting

### Common Issues

1. **Webhook Not Receiving Events**:
   - Check webhook URL configuration in Salla
   - Verify webhook endpoint is accessible
   - Check webhook logs

2. **Enrollment Failures**:
   - Check Elham API key is valid
   - Verify team ID exists in Elham
   - Check order_syncs.error_message for details

3. **Authentication Issues**:
   - Verify Supabase credentials
   - Check RLS policies
   - Verify user exists in auth.users

4. **Encryption Errors**:
   - Verify ENCRYPTION_SECRET is set
   - Check encryption secret matches for encrypt/decrypt
   - Verify environment variables are loaded

---

*For detailed workflows, see [Workflows Documentation](./WORKFLOWS.md)*
*For API details, see [API Documentation](./API_DOCUMENTATION.md)*
*For database structure, see [Database Schema Documentation](./DATABASE_SCHEMA.md)*

