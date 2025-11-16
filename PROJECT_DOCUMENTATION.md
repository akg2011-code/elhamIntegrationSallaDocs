# Salla-Elham Integration - Complete Project Documentation

## Executive Summary

The Salla-Elham Integration is a Next.js middleware application that seamlessly connects Salla (E-commerce platform) and Elham (Learning Management System/Academy platform). The system enables merchants to automatically enroll customers in Elham courses when they purchase mapped products in their Salla store.

### Key Value Propositions

1. **Automated Course Enrollment**: Eliminates manual work by automatically enrolling customers in courses when they purchase specific products
2. **Seamless Integration**: Connects two major platforms (Salla and Elham) through webhooks and API integrations
3. **Team-Based Learning**: Supports Elham's team structure for organized course delivery
4. **Order Lifecycle Management**: Handles order creation, updates, refunds, and cancellations with appropriate course enrollment/unenrollment
5. **Secure Authentication**: Uses Supabase Auth with encrypted API key storage
6. **Bilingual Support**: Full Arabic and English support with cookie-based locale management

## Table of Contents

1. [Business Logic Documentation](./docs/BUSINESS_LOGIC.md) - Business model, rules, and user personas
2. [Technical Documentation](./docs/TECHNICAL_DOCUMENTATION.md) - Architecture, tech stack, and security
3. [Workflows Documentation](./docs/WORKFLOWS.md) - Detailed step-by-step workflows
4. [Database Schema Documentation](./docs/DATABASE_SCHEMA.md) - Complete database structure and relationships
5. [API Documentation](./docs/API_DOCUMENTATION.md) - All API endpoints and integrations

## Quick Reference Guide

### Core Functionality

- **User Management**: Automatic user creation via Salla webhook, manual registration, and authentication
- **Store Connection**: OAuth-based Salla store connection with encrypted token storage
- **API Configuration**: Elham API key management with AES encryption
- **Product Mapping**: Map Salla products to Elham courses/teams
- **Order Processing**: Automatic enrollment when orders are completed
- **Refund Handling**: Automatic unenrollment when orders are refunded/cancelled

### Key Technologies

- **Framework**: Next.js 16+ with TypeScript
- **Database**: Supabase (PostgreSQL) with Row Level Security
- **Authentication**: Supabase Auth
- **Encryption**: AES encryption via crypto-js
- **Internationalization**: next-intl with cookie-based locale
- **Styling**: Tailwind CSS with Radix UI components

### Important Files

- **Webhook Handler**: `app/api/webhooks/salla/route.ts`
- **Salla API Client**: `lib/salla/api.ts`
- **Elham API Client**: `lib/elham/api.ts`
- **Encryption Utilities**: `lib/crypto/encrypt.ts`
- **Database Migrations**: `supabase/migrations/`

## Business Model Overview

The platform serves as a middleware connecting e-commerce (Salla) and learning management (Elham) platforms. Merchants can:

1. Connect their Salla store via OAuth
2. Configure their Elham Academy API key
3. Map products to courses/teams
4. Automatically enroll customers when they purchase mapped products

### User Flow

```
Salla Store → Webhook → Integration Platform → Elham Academy
     ↓                                           ↓
  Order Created                            Customer Enrolled
  Order Completed                          Added to Team
  Order Refunded                           Removed from Team
```

## High-Level Architecture

```
┌─────────────────┐
│   Salla Store   │
│   (E-commerce)  │
└────────┬────────┘
         │ Webhooks
         │ (OAuth)
         ▼
┌─────────────────┐
│  Integration    │
│    Platform     │
│  (Next.js App)  │
└────────┬────────┘
         │ API Calls
         │ (API Key)
         ▼
┌─────────────────┐
│  Elham Academy  │
│   (LXS/Teams)   │
└─────────────────┘
```

### Components

1. **Frontend**: Next.js pages with React components
2. **Backend**: Next.js API routes and server actions
3. **Database**: Supabase PostgreSQL with RLS
4. **External APIs**: Salla OAuth API, Elham REST API
5. **Webhooks**: Salla webhook endpoint for real-time events

## Security Features

- **Encrypted Storage**: All API keys and tokens encrypted with AES
- **Row Level Security**: Database-level access control
- **OAuth Integration**: Secure token management for Salla
- **Environment Variables**: Sensitive data stored in environment variables
- **HTTPS**: Required for production deployments

## Getting Started

For detailed setup instructions, see the [README.md](./README.md) file.

For comprehensive documentation on specific topics, refer to the detailed sections:

- **Business Logic**: Understanding the business rules and workflows
- **Technical Documentation**: Implementation details and architecture
- **Workflows**: Step-by-step process flows
- **Database Schema**: Data structure and relationships
- **API Documentation**: Endpoint reference and integration details

## Support and Maintenance

- **Repository**: [GitHub Repository](https://github.com/akg2011-code/elhamIntegrationSalla)
- **Issues**: Report issues via GitHub Issues
- **Documentation**: All documentation is maintained in this repository

---

_Last Updated: 2025_
_Version: 1.0.0_
