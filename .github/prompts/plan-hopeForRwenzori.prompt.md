# Backend Tech Stack Plan — Hope for Rwenzori Foundation

## Overview
Backend tech stack for handling volunteer applications, donations, payments (Stripe, MTN, Airtel, PayPal, Mastercard), email notifications, real-time chat with earnings tracking, and admin content management. Designed for scalability, security, and maintainability.

## User Context
- **Deployment**: Self-hosted VPS
- **Scale**: Small (< 1,000 monthly users) with growth potential
- **Real-time**: Yes (chat + historical messaging)
- **Team Language**: Node.js / JavaScript
- **Compliance**: To be determined (PCI-DSS for payments recommended)

---

## Recommended Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Runtime** | Node.js v20 LTS | Team familiar, large ecosystem, mature tooling |
| **Web Server** | Express.js | Mature, middleware-rich, ideal for REST APIs |
| **Primary Database** | PostgreSQL v15+ | ACID transactions (critical for payments), strong JSON support, PostGIS ready |
| **Cache/Session** | Redis v7+ | Real-time chat, session management, rate limiting, Pub/Sub |
| **Real-time Comms** | Socket.io | WebSocket-based chat, notifications, presence tracking |
| **Primary Payments** | Stripe API | Handles cards, PayPal, Apple Pay, Google Pay; PCI-compliant |
| **Mobile Money** | MTN MoMo + Airtel Money APIs | Direct integration for regional payment methods |
| **Email Service** | SendGrid or Brevo | Reliable delivery, templates, bounce/complaint webhooks |
| **File Storage** | Cloudinary or AWS S3 | CDN included, image optimization, reduces server load |
| **Authentication** | JWT + bcrypt | Stateless, scalable, industry standard |
| **Process Manager** | PM2 or systemd | Production reliability on VPS |
| **Reverse Proxy** | Nginx | SSL termination, load balancing, static serving |
| **Containerization** | Docker + Docker Compose | Consistent dev/prod environments |
| **Testing** | Jest + Supertest | Fast unit + API integration testing |
| **API Docs** | Swagger/OpenAPI | Auto-generated, interactive documentation |

---

## Architecture Overview

### REST API Structure
```
/api/v1/
  /auth
    POST   /register           - Create new account
    POST   /login             - Authenticate
    POST   /refresh-token     - Refresh JWT
    POST   /logout            - Invalidate session
  
  /applications
    POST   /                  - Submit volunteer application
    GET    /                  - List all (admin filtered)
    GET    /:id               - Read single application
    PATCH  /:id               - Update status (approve/reject)
    DELETE /:id               - Delete (admin only)
  
  /payments
    POST   /stripe/intent     - Create Stripe payment intent
    POST   /stripe/webhook    - Stripe webhook handler
    POST   /mtn/initiate      - Initiate MTN payment
    POST   /airtel/initiate   - Initiate Airtel payment
    GET    /history           - Donation/transaction history
  
  /donors
    GET    /me                - Donor dashboard
    GET    /:id               - Public donor profile
    PATCH  /me                - Update preferences
    GET    /dashboard/stats   - Donation stats
  
  /chat
    POST   /messages          - Send message (REST fallback)
    GET    /history           - Paginated chat history
    GET    /:conversationId   - Get conversation
  
  /content (admin only)
    GET    /pages             - List all content
    PATCH  /pages/:id         - Update page section
    POST   /images            - Upload image (returns CDN URL)
    DELETE /images/:id        - Delete image
  
  /admin
    GET    /users             - List all users
    PATCH  /users/:id/role    - Update user role
    GET    /applications      - Application queue
    GET    /reports           - Donation reports, analytics
    GET    /earnings          - Volunteer earnings ledger
```

### WebSocket Events (Socket.io)
```
Client → Server:
  - user-connect         - Register online presence
  - send-message         - New chat message
  - typing-indicator     - Show typing status
  - mark-read           - Mark message as read

Server → Broadcast:
  - new-message         - Broadcast message to recipients
  - user-online         - User came online
  - user-offline        - User went offline
  - donation-received   - New donation alert
  - application-decision - Approval/rejection notification
```

### Database Schema (7 Core Tables)

#### Users
```sql
id (PK), role (admin/donor/volunteer), email, password_hash, 
first_name, last_name, phone, country, profile_photo_url, 
created_at, updated_at, is_active
```

#### Applications
```sql
id (PK), user_id (FK), status (pending/approved/rejected), 
title, description, skills, availability, motivation, 
documents_url, admin_notes, decision_date, created_at, updated_at
```

#### Payments
```sql
id (PK), donor_id (FK), amount, currency (UGX/USD), 
method (stripe/mtn/airtel/paypal/mastercard), 
external_transaction_id (Stripe ID or MTN reference), 
status (pending/completed/failed/refunded), 
purpose (donation/membership/other), 
created_at, updated_at, metadata (JSON for receipts)
```

#### Donations
```sql
id (PK), payment_id (FK), donor_id (FK), amount, 
frequency (one_time/monthly/yearly), purpose, 
is_anonymous, created_at
```

#### Chat
```sql
id (PK), sender_id (FK), receiver_id (FK), message, 
attachments_url[], is_read, read_at, created_at, updated_at
```

#### Content
```sql
id (PK), page (homepage/about/programs/news), 
section_key (hero/stats/cta), title, body (HTML/JSON), 
image_url, display_order, is_published, updated_by (FK), 
updated_at
```

#### Earnings
```sql
id (PK), volunteer_id (FK), chat_message_id (FK), 
amount, status (pending/paid), paid_at, created_at
```

---

## Folder Structure

```
backend/
├── src/
│   ├── config/
│   │   ├── database.js          # PostgreSQL connection pool
│   │   ├── redis.js             # Redis client + session store
│   │   ├── environment.js       # Environment validation (dotenv)
│   │   └── constants.js         # App-wide constants
│   │
│   ├── models/                  # Database models (Sequelize/TypeORM)
│   │   ├── User.js
│   │   ├── Application.js
│   │   ├── Payment.js
│   │   ├── Donation.js
│   │   ├── Chat.js
│   │   ├── Content.js
│   │   └── Earnings.js
│   │
│   ├── controllers/             # Route handlers
│   │   ├── authController.js
│   │   ├── applicationController.js
│   │   ├── paymentController.js
│   │   ├── donorController.js
│   │   ├── chatController.js
│   │   ├── contentController.js
│   │   └── adminController.js
│   │
│   ├── services/                # Business logic + external integrations
│   │   ├── PaymentService.js    # Strategy pattern for all payment methods
│   │   ├── StripeService.js
│   │   ├── MobileMoneyService.js # MTN + Airtel wrappers
│   │   ├── EmailService.js      # SendGrid integration
│   │   ├── FileUploadService.js # Cloudinary/S3
│   │   ├── ChatService.js       # Message persistence
│   │   └── AuthService.js       # JWT, password hashing
│   │
│   ├── middleware/
│   │   ├── auth.js              # JWT verification
│   │   ├── authorization.js     # Role-based access control
│   │   ├── validation.js        # Request schema validation
│   │   ├── errorHandler.js      # Global error handling
│   │   ├── rateLimiter.js       # Express-rate-limit
│   │   └── requestLogger.js     # Morgan request logging
│   │
│   ├── routes/
│   │   ├── auth.js
│   │   ├── applications.js
│   │   ├── payments.js
│   │   ├── donors.js
│   │   ├── chat.js
│   │   ├── content.js
│   │   ├── admin.js
│   │   └── index.js             # Route aggregator
│   │
│   ├── websocket/
│   │   ├── io.js                # Socket.io setup
│   │   ├── chatEvents.js        # Message handlers
│   │   ├── notificationEvents.js # Alert handlers
│   │   └── middleware.js         # Socket auth
│   │
│   ├── jobs/
│   │   ├── emailQueue.js        # Bull job: send email
│   │   ├── paymentReconciliation.js # Stripe/MTN status sync
│   │   └── earnignsCalculation.js   # Daily earnings update
│   │
│   ├── utils/
│   │   ├── validators.js        # Input validation helpers
│   │   ├── formatters.js        # Response formatting
│   │   ├── logger.js            # Winston logger
│   │   ├── errorCodes.js        # Standardized error responses
│   │   └── helpers.js           # General utilities
│   │
│   └── app.js                   # Express app setup
│
├── tests/
│   ├── unit/
│   │   ├── services/
│   │   ├── utils/
│   │   └── models/
│   │
│   ├── integration/
│   │   ├── auth.test.js
│   │   ├── payments.test.js
│   │   ├── applications.test.js
│   │   └── chat.test.js
│   │
│   └── fixtures/
│       └── testData.js
│
├── docker/
│   ├── Dockerfile
│   ├── .dockerignore
│   └── docker-compose.yml
│
├── migrations/
│   ├── 001_initial_schema.sql
│   ├── 002_add_payments.sql
│   └── ...
│
├── .env.example              # Template for environment variables
├── .eslintrc.json
├── .prettierrc.json
├── jest.config.js
├── package.json
├── package-lock.json
└── README.md
```

---

## Security Implementation

### Authentication & Authorization
- **JWT Strategy**: Access token (15min) + Refresh token (7 days, stored in Redis)
- **Password**: bcrypt with 10+ salt rounds
- **Role-based**: Middleware checks role on admin endpoints
- **Session**: Redis-backed sessions with expiry
- **HTTPS**: Mandatory in production (Let's Encrypt + Certbot)

### Payment Security
- **PCI-DSS Level 1**: Never store raw card data (use Stripe tokens only)
- **Encryption**: Sensitive fields encrypted at rest (crypto module)
- **Webhook Verification**: Validate Stripe/MTN signatures on incoming events
- **Transaction Logging**: All payment attempts logged (immutable audit trail)
- **Retry Logic**: Exponential backoff for failed payment reconciliation

### API Security
- **CORS**: Whitelist frontend domain only (process.env.FRONTEND_URL)
- **Rate Limiting**: 100 req/10 min per IP (stricter for auth endpoints: 5 req/min)
- **Input Validation**: Joi/Zod schemas for all request bodies
- **SQL Injection**: Parameterized queries via ORM (Sequelize/TypeORM)
- **XSS Prevention**: Escape user input, sanitize HTML
- **CSRF**: Token-based protection for state-changing requests
- **Security Headers**: Helmet.js (HSTS, CSP, X-Frame-Options, etc.)

### Data Protection
- **Sensitive Fields**: Encrypt volunteer social security numbers, bank details
- **Logging**: Never log passwords, card numbers, or personal data
- **GDPR**: Right-to-deletion flow (anonymize user data)
- **Backups**: Encrypted, stored off-site, rotated daily

---

## Scalability Roadmap

### Phase 1: MVP (< 500 users)
- Single VPS instance (4GB RAM, 100GB SSD)
- PostgreSQL single node
- Redis single node
- No caching layer except Redis
- Synchronous email via SendGrid
- CDN for static assets (Cloudinary)

### Phase 2: Growth (500–2,000 users)
- Database read replicas (for analytics/reporting queries)
- Redis cluster for high availability
- Bull job queue for async tasks (email, payment reconciliation)
- Application horizontal scaling with PM2 cluster mode
- Metrics collection (New Relic or Prometheus)

### Phase 3: Enterprise (2,000–10,000+ users)
- Load balancer (nginx or HAProxy) across 3+ app instances
- PostgreSQL with connection pooling (PgBouncer)
- Separate payment service (microservice)
- Message queue (RabbitMQ or AWS SQS) for order of operations
- Full observability (centralized logging, APM, alerting)

---

## Implementation Phases

### Phase 1: Project Setup (Week 1)
- [ ] Initialize Node.js project (Express, TypeScript optional)
- [ ] Set up PostgreSQL + Sequelize/TypeORM
- [ ] Configure Redis + express-session
- [ ] Docker + docker-compose setup
- [ ] Environment config with validation
- [ ] Basic logging (Winston)

### Phase 2: Authentication & Database (Week 2)
- [ ] Create database schema (migrations)
- [ ] User model with roles
- [ ] JWT auth (register, login, refresh)
- [ ] Password reset flow via email
- [ ] Rate limiting middleware
- [ ] Swagger docs scaffold

### Phase 3: Volunteer Applications (Week 3)
- [ ] Application model + CRUD endpoints
- [ ] File upload (document validation)
- [ ] Admin review queue
- [ ] Approval/rejection with email notification
- [ ] Application status tracking

### Phase 4: Payment Integration (Week 4–5)
- [ ] Stripe integration (payment intent → webhook)
- [ ] MTN MoMo API integration with retry logic
- [ ] Airtel Money API integration
- [ ] Payment model + transaction logging
- [ ] Webhook signature verification
- [ ] Reconciliation job (sync Stripe/MTN status)

### Phase 5: Real-time Chat (Week 6)
- [ ] Socket.io setup with Redis adapter
- [ ] Chat model + persistence
- [ ] Message send/receive events
- [ ] Typing indicators + online status
- [ ] Chat history pagination
- [ ] Earnings tracking (volunteer chat earnings)

### Phase 6: Email & Notifications (Week 6)
- [ ] SendGrid integration
- [ ] Email templates (approval, donation, chat)
- [ ] Bull job queue for async sending
- [ ] Webhook handlers (bounce, complaint)
- [ ] Test email delivery

### Phase 7: Admin Dashboard API (Week 7)
- [ ] Content CRUD (homepage sections, nav)
- [ ] User management (list, deactivate, role changes)
- [ ] Donation reports + analytics
- [ ] Application queue + bulk actions
- [ ] Chat moderation (pin, flag, delete)

### Phase 8: Testing & Docs (Week 8)
- [ ] Unit tests for services (payment, email, auth)
- [ ] Integration tests for API endpoints
- [ ] Swagger docs finalization
- [ ] Environment variable documentation
- [ ] Deployment runbook

### Phase 9: Deployment Setup (Week 9)
- [ ] Production Dockerfile
- [ ] Nginx config (SSL, reverse proxy)
- [ ] Let's Encrypt SSL certificate
- [ ] PM2 config or systemd service
- [ ] Monitoring setup (PM2 Plus or New Relic)
- [ ] Database backups (automated)

### Phase 10: Security Hardening & Testing (Week 10)
- [ ] Security audit (OWASP Top 10)
- [ ] Load testing (50+ concurrent users)
- [ ] Payment sandbox testing (Stripe, MTN, Airtel)
- [ ] Chat stress testing (100+ messages/sec)
- [ ] Admin access control verification
- [ ] Incident response runbook

---

## Key Dependencies (npm packages)

### Core Framework
- `express` - Web server
- `dotenv` - Environment variable management
- `cors` - Cross-origin requests
- `helmet` - Security headers
- `body-parser` - JSON/form parsing

### Database & ORM
- `sequelize` - ORM for PostgreSQL
- `pg` - PostgreSQL driver
- `sequelize-cli` - Migration management

### Caching & Sessions
- `redis` - Redis client
- `express-session` - Session middleware
- `connect-redis` - Redis session store

### Authentication
- `jsonwebtoken` - JWT creation/verification
- `bcryptjs` - Password hashing
- `passport` - Auth framework
- `passport-jwt` - JWT strategy

### Validation & Middleware
- `joi` - Schema validation
- `express-validator` - Request validation
- `express-rate-limit` - Rate limiting
- `morgan` - HTTP logging

### Payments
- `stripe` - Stripe SDK
- `axios` - HTTP client (MTN/Airtel APIs)

### Email
- `sendgrid` or `brevo` - Email service SDKs
- `nodemailer` - Fallback SMTP

### Real-time
- `socket.io` - WebSocket library
- `socket.io-redis` - Redis adapter for scaling

### File Upload
- `multer` - File upload middleware
- `cloudinary` - Image hosting SDK

### Background Jobs
- `bull` - Job queue (Redis-backed)
- `bull-board` - Queue UI

### Testing
- `jest` - Testing framework
- `supertest` - HTTP assertion library
- `@faker-js/faker` - Test data generation

### Development
- `nodemon` - Auto-restart on file changes
- `eslint` - Linting
- `prettier` - Code formatting

### Utilities
- `uuid` - Generate unique IDs
- `moment` or `date-fns` - Date manipulation
- `lodash` - Utility functions
- `winston` - Logging

---

## Environment Variables (.env.example)

```env
# Server
NODE_ENV=development
PORT=3000
HOST=0.0.0.0

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=your_secure_password
DB_NAME=hope_rwenzori

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=your_super_secret_key_min_32_chars
JWT_REFRESH_SECRET=your_refresh_secret_key
JWT_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...

# MTN MoMo
MTN_API_KEY=your_mtn_api_key
MTN_COLLECTION_PRIMARY_KEY=your_primary_key
MTN_COLLECTION_API_USER=your_api_user

# Airtel
AIRTEL_CLIENT_ID=your_client_id
AIRTEL_CLIENT_SECRET=your_client_secret
AIRTEL_API_KEY=your_api_key

# Email
SENDGRID_API_KEY=SG.your_sendgrid_key
EMAIL_FROM=noreply@hopeforrwenzori.org

# File Upload
CLOUDINARY_CLOUD_NAME=your_cloud
CLOUDINARY_API_KEY=your_key
CLOUDINARY_API_SECRET=your_secret

# Frontend
FRONTEND_URL=http://localhost:3001
CORS_ORIGIN=http://localhost:3001

# Logging
LOG_LEVEL=debug
LOG_FILE=logs/app.log
```

---

## Questions for Refinement

1. **Admin Dashboard UI**: Do you want a React-based admin panel or just backend APIs?
2. **Chat Moderation**: Should admins be able to read all donor-volunteer chats, or only flagged messages?
3. **Automation**: Should volunteers auto-match to donors based on skills? Or manual assignment?
4. **Donor Tiers**: Do you want recognition levels (Bronze/Silver/Gold) based on cumulative donations?
5. **Tax Receipts**: Should the system generate downloadable tax receipt PDFs for donations?
6. **API Rate Limits**: Different for public endpoints vs. authenticated vs. admin?
7. **Data Retention**: How long to retain chat history? Payment records?
8. **Internationalization**: Multi-language support for emails/content?
9. **Analytics**: Real-time dashboard or weekly/monthly reports via email?
10. **Refunds**: Full refund process for failed/disputed payments?

---

## Next Steps

1. **Review** this plan and confirm all technology choices
2. **Clarify** optional features and questions above
3. **Set Timeline**: Assign each phase a start/end date
4. **Assign Team**: Who leads backend? Who reviews PRs?
5. **Create Repository**: Set up GitHub repo structure
6. **Begin Phase 1**: Start scaffolding the project
