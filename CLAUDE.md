# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Structure

This is a monorepo with Git submodules containing a full-stack order management application:

- **mercury-app/** - Frontend (React 19 + TypeScript + Vite)
- **Backend/** - Backend (Deno + Supabase + Hono)

## Essential Commands

### Frontend (mercury-app/)
```bash
# Development
npm run dev                      # Start dev server with proxy to backend

# Production builds
npm run build:production         # Full pipeline: clean, typecheck, lint, test, build
npm run build:pwa               # Build with PWA enabled
npm run build:no-pwa            # Build without PWA

# Testing
npm run test:all                # Run all test suites
npm run test:auth               # Authentication tests
npm run test:components         # Component tests
npm run test:pwa                # PWA functionality tests

# Code quality
npm run typecheck               # TypeScript checking
npm run lint                    # ESLint
npm run format                  # Prettier

# PWA management
npm run pwa:enable              # Enable PWA functionality
npm run pwa:disable             # Disable PWA functionality

# Shadcn components
pnpx shadcn@latest add button   # Add new Shadcn UI components
```

### Backend (Backend/)
```bash
# Development
deno task dev                   # Watch mode development server

# Production
deno task start                 # Production server

# Code quality and testing
deno task test                  # Run tests
deno task lint                  # Linting
deno task fmt                   # Formatting
deno task check                 # Type checking
```

### Git Submodules
```bash
# Clone with submodules
git clone --recursive <repo-url>

# Update submodules
git submodule update --remote

# Initialize submodules (if cloned without --recursive)
git submodule update --init --recursive
```

## Architecture Overview

### Monorepo with Submodules
- Each component (frontend/backend) is a separate Git repository
- Main repository orchestrates both via submodules
- Independent development and deployment cycles

### Frontend Architecture (mercury-app/)
- **Framework**: React 19 with TanStack Router (file-based routing)
- **UI**: Shadcn/ui + Radix UI primitives with Tailwind CSS
- **State**: TanStack Store + React Query for data fetching
- **Forms**: React Hook Form + Zod validation
- **PWA**: Configurable via VITE_PWA_DISABLED environment variable
- **Offline-first**: Service worker with local storage fallbacks
- **File structure**: Routes in `src/routes/`, components in `src/components/`

### Backend Architecture (Backend/)
- **Runtime**: Deno 2.2.14 with Hono web framework
- **Database**: Supabase (PostgreSQL) with Row Level Security
- **Security**: Zod validation on 13/15 critical endpoints, CSRF protection, XSS prevention
- **Integrations**: WhatsApp alerts, AWS S3, Stripe, Resend email
- **Monitoring**: Security logging with WhatsApp notifications

### Key Integration Points
- **Authentication**: Supabase Auth for both frontend and backend
- **Real-time**: Supabase subscriptions for live updates
- **Development Proxy**: Frontend dev server proxies `/api` to `localhost:3030`
- **File Uploads**: Direct to AWS S3 with signed URLs
- **Payment Processing**: Stripe integration with webhook handling

## Environment Variables

### Frontend (.env)
```env
VITE_SUPABASE_URL=your_supabase_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
VITE_STRIPE_PUBLISHABLE_KEY=your_stripe_key
VITE_PWA_DISABLED=true          # Toggle PWA functionality
```

### Backend (.env)
```env
SUPABASE_URL=your_supabase_url
SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_key
STRIPE_SECRET_KEY=your_stripe_secret
WHATSAPP_TOKEN=your_whatsapp_token
WHATSAPP_PHONE_NUMBER_ID=your_phone_id
```

## Testing Strategy

### Frontend Testing
- **Unit tests**: Utilities and pure functions with Vitest
- **Component tests**: React Testing Library
- **Integration tests**: Authentication flows
- **PWA tests**: Service worker functionality
- **Multiple test configurations**: Auth, components, PWA, hooks

### Backend Testing
- **API endpoint tests**: Deno's built-in test runner
- **Security validation tests**: Input validation and sanitization
- **Load testing**: Performance validation

## Security Features

This codebase follows security-first practices:
- **Input validation**: Zod schemas on all critical endpoints
- **CSRF protection**: Token validation
- **XSS protection**: Content sanitization with DOMPurify
- **Rate limiting**: Authentication endpoints
- **Comprehensive logging**: Security events with WhatsApp alerts
- **Row Level Security**: Database-level access control

## Development Notes

### PWA Configuration
- PWA can be enabled/disabled via `VITE_PWA_DISABLED` environment variable
- Use `npm run build:pwa` or `npm run build:no-pwa` for specific builds
- Service worker handles offline functionality and caching strategies

### Component Development
- Use `pnpx shadcn@latest add <component>` to add new UI components
- Follow existing patterns in `src/components/` for consistency
- Utilize TanStack Router for file-based routing in `src/routes/`

### Database Management
- Supabase handles authentication, real-time subscriptions, and storage
- Row Level Security policies enforce data access controls
- Migration scripts available in Backend/migrations/

### Deployment
- **Frontend**: Vercel deployment with automatic builds
- **Backend**: Deno Deploy with environment configuration
- **Monitoring**: Production alerts via WhatsApp integration

This is a production-ready application with comprehensive testing, security measures, and modern development practices optimized for order management with offline-first PWA capabilities.