# Technology Stack Reference

> **Scope:** Reference architecture for production Node.js/React applications with compliance considerations for SOC 2, HIPAA, PCI DSS, and GDPR.

---

## Quick Stack Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                │
│  React 18 + TypeScript + Vite + TailwindCSS + Zustand          │
│  React Query + Socket.io-client + React Router                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      REVERSE PROXY                              │
│  nginx (TLS termination, rate limiting, static files)          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         BACKEND                                 │
│  Fastify + TypeScript + Prisma + Zod + Pino                    │
│  Socket.io (real-time) + BullMQ (jobs) + ioredis               │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌───────────────────┐ ┌─────────────┐ ┌─────────────────┐
│   PostgreSQL 15   │ │  Redis 7    │ │  LiveKit        │
│   (Primary DB)    │ │  (Cache/PubSub) │  (Video/Audio)  │
└───────────────────┘ └─────────────┘ └─────────────────┘
```

---

## Core Stack

### Runtime & Language

| Technology | Version | Purpose | Compliance Notes |
|------------|---------|---------|------------------|
| **Node.js** | 20+ LTS | Runtime | LTS for security patches |
| **TypeScript** | 5.3+ | Type safety | Compile-time error prevention |

**Configuration:**
```json
// tsconfig.json - Production settings
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Backend Framework

| Technology | Version | Purpose | Why Chosen |
|------------|---------|---------|------------|
| **Fastify** | 4.25+ | Web framework | 2-3x faster than Express, schema validation built-in |
| @fastify/helmet | 11.x | Security headers | OWASP protections (XSS, clickjacking) |
| @fastify/rate-limit | 9.x | Rate limiting | DDoS protection, compliance requirement |
| @fastify/cors | 8.x | CORS | Controlled cross-origin access |
| @fastify/cookie | 9.x | Cookies | Secure session management |
| @fastify/swagger | 8.x | API docs | Auto-generated OpenAPI |

**Security Configuration:**
```typescript
// Fastify security setup - Satisfies SOC 2, PCI DSS
import Fastify from 'fastify';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';

const app = Fastify({
  logger: true,  // Audit logging
  trustProxy: true,  // Behind nginx
});

await app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],  // TailwindCSS
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "wss:"],  // WebSocket
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
});

await app.register(rateLimit, {
  max: 100,           // Requests per window
  timeWindow: '1 minute',
  keyGenerator: (req) => req.ip,  // Or user ID for authenticated
});
```

### Database & ORM

| Technology | Version | Purpose | Compliance Notes |
|------------|---------|---------|------------------|
| **PostgreSQL** | 15-alpine | Primary database | ACID, encryption at rest, audit logging |
| **Prisma** | 5.22+ | ORM | Type-safe queries, schema migrations |

**Prisma Compliance Configuration:**
```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")  // Must use ?sslmode=require
}

// Audit trail model (HIPAA, SOC 2)
model AuditLog {
  id            String   @id @default(uuid())
  timestamp     DateTime @default(now())
  userId        String
  action        String   // CREATE, READ, UPDATE, DELETE
  resource      String
  resourceId    String
  ipAddress     String
  userAgent     String
  oldValue      Json?    // For UPDATE/DELETE
  newValue      Json?    // For CREATE/UPDATE

  @@index([userId, timestamp])
  @@index([resource, resourceId])
}
```

**When to use `db push` vs `migrate`:**

| Command | Use Case | Production Safe? |
|---------|----------|------------------|
| `prisma db push` | Prototyping, no migration history needed | ⚠️ Use with caution |
| `prisma migrate deploy` | Production with audit trail | ✅ Yes |

See [Prisma & TypeScript Workflows](./prisma-typescript-workflows.md) for details.

### Caching & Message Queue

| Technology | Version | Purpose | Compliance Notes |
|------------|---------|---------|------------------|
| **Redis** | 7-alpine | Cache, sessions, pub/sub | Enable TLS, encrypted persistence |
| **ioredis** | 5.3+ | Redis client | Cluster support, reconnection |
| **BullMQ** | 5.x | Job queue | Reliable background jobs |

**Redis Security Configuration:**
```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --appendfsync everysec
    volumes:
      - redis_data:/data
```

**For compliance environments, add TLS:**
```yaml
command: >
  redis-server
  --requirepass ${REDIS_PASSWORD}
  --tls-port 6379
  --port 0
  --tls-cert-file /tls/redis.crt
  --tls-key-file /tls/redis.key
  --tls-ca-cert-file /tls/ca.crt
```

---

## Real-Time & Communication

### WebSocket (Socket.io)

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| **Socket.io** | 4.6+ | Real-time events | Notifications, live updates |
| socket.io-client | 4.6+ | Browser client | React integration |

**Secure Socket.io Setup:**
```typescript
import { Server } from 'socket.io';
import { verifyToken } from './auth';

const io = new Server(server, {
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true
  },
  // Security
  pingTimeout: 60000,
  pingInterval: 25000,
});

// Authentication middleware
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const user = await verifyToken(token);
    socket.data.user = user;
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});
```

### Video/Audio (LiveKit)

| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| **LiveKit** | Video conferencing | Self-hostable, WebRTC, HIPAA-capable |
| livekit-server-sdk | Server integration | Room management, tokens |
| @livekit/components-react | React components | Pre-built UI |

**Why LiveKit over alternatives:**

| Feature | LiveKit | Twilio | Daily.co |
|---------|---------|--------|----------|
| Self-hosted | ✅ Yes | ❌ No | ❌ No |
| HIPAA capable | ✅ Yes | ✅ Yes | ✅ Yes |
| Pricing | Open source | Per minute | Per minute |
| WebRTC native | ✅ Yes | ✅ Yes | ✅ Yes |

**LiveKit Docker Setup:**
```yaml
# docker-compose.yml
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /etc/livekit.yaml
    ports:
      - "7880:7880"   # HTTP
      - "7881:7881"   # WebRTC (TCP)
      - "7882:7882/udp"  # WebRTC (UDP)
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml:ro
```

---

## Email Integration

### Recommended: Resend

| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| **Resend** | Transactional email | Developer-first, React Email support |

**Why Resend over alternatives:**

| Feature | Resend | SendGrid | AWS SES | Nodemailer |
|---------|--------|----------|---------|------------|
| React Email | ✅ Native | ❌ No | ❌ No | ❌ No |
| API simplicity | ✅ Excellent | ⚠️ Complex | ⚠️ Complex | ✅ Simple |
| Deliverability | ✅ High | ✅ High | ✅ High | Depends |
| Free tier | 3k/month | 100/day | 62k/month | N/A |
| HIPAA BAA | ✅ Available | ✅ Available | ✅ Available | N/A |

**Resend Integration:**
```typescript
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

// With React Email templates
import { BookingConfirmation } from '@/emails/booking-confirmation';

await resend.emails.send({
  from: 'bookings@yourdomain.com',
  to: user.email,
  subject: 'Booking Confirmed',
  react: BookingConfirmation({ booking, user }),
});
```

**React Email Template Example:**
```tsx
// emails/booking-confirmation.tsx
import { Html, Body, Container, Text, Button } from '@react-email/components';

export function BookingConfirmation({ booking, user }) {
  return (
    <Html>
      <Body style={{ fontFamily: 'sans-serif' }}>
        <Container>
          <Text>Hi {user.name},</Text>
          <Text>Your booking for {booking.eventType.name} is confirmed.</Text>
          <Text>Date: {format(booking.startTime, 'PPP p')}</Text>
          <Button href={`${process.env.APP_URL}/bookings/${booking.id}`}>
            View Booking
          </Button>
        </Container>
      </Body>
    </Html>
  );
}
```

---

## Frontend Stack

### Core Framework

| Technology | Version | Purpose | Why Chosen |
|------------|---------|---------|------------|
| **React** | 18.2+ | UI framework | Concurrent rendering, hooks |
| **Vite** | 5.x | Build tool | 10-100x faster than Webpack |
| **React Router** | 6.x | Routing | Nested routes, data loading |

### State Management

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| **Zustand** | 4.4+ | Client state | UI state, user preferences |
| **React Query** | 5.x | Server state | API data, caching, sync |

**When to use which:**

| State Type | Tool | Example |
|------------|------|---------|
| UI state | Zustand | Modal open/close, sidebar toggle |
| Form state | React Hook Form | Form inputs, validation |
| Server data | React Query | User data, bookings list |
| URL state | React Router | Filters, pagination |
| Real-time | Socket.io | Notifications, live updates |

### Styling

| Technology | Version | Purpose | Why Chosen |
|------------|---------|---------|------------|
| **TailwindCSS** | 3.4+ | Utility CSS | Rapid development, consistent design |
| PostCSS | 8.x | CSS processing | Required by Tailwind |
| Autoprefixer | 10.x | Vendor prefixes | Browser compatibility |

---

## Security Stack

### Authentication & Authorization

| Technology | Version | Purpose | Compliance |
|------------|---------|---------|------------|
| **@node-rs/argon2** | 1.7+ | Password hashing | OWASP recommended over bcrypt |
| **nanoid** | 5.x | Token generation | Cryptographically secure |
| **jose** | 5.x | JWT handling | Standard compliance |

**Password Hashing Configuration:**
```typescript
import { hash, verify } from '@node-rs/argon2';

// OWASP recommended settings
const hashPassword = async (password: string) => {
  return hash(password, {
    memoryCost: 65536,  // 64 MB
    timeCost: 3,        // 3 iterations
    parallelism: 4,     // 4 threads
  });
};
```

### ID Generation Strategy

| Use Case | Tool | Format | Example |
|----------|------|--------|---------|
| Public IDs (URLs) | nanoid | `V1StGXR8_Z5jdHi` | Booking links |
| Audit trails | UUID v7 | `01893a4f-...` | Traceable, time-ordered |
| Database PKs | UUID v4 | `550e8400-...` | Standard format |

```typescript
import { nanoid } from 'nanoid';
import { v7 as uuidv7 } from 'uuid';

// Public-facing short ID
const bookingSlug = nanoid(12);  // V1StGXR8_Z5j

// Audit trail (time-ordered, traceable)
const auditId = uuidv7();  // 01893a4f-7a6b-7...
```

---

## Logging & Monitoring

### Logging (Pino)

| Technology | Version | Purpose | Compliance |
|------------|---------|---------|------------|
| **Pino** | 8.x | Structured logging | Fastest, JSON for aggregation |
| pino-pretty | 10.x | Dev formatting | Human-readable in development |

**Compliant Logging Setup:**
```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // HIPAA: Never log PHI in plain text
  redact: {
    paths: ['req.headers.authorization', 'password', 'ssn', 'dob'],
    censor: '[REDACTED]'
  },
  // SOC 2: Include request context
  mixin: () => ({
    service: 'api',
    environment: process.env.NODE_ENV,
  }),
});
```

### Recommended Additions

| Technology | Purpose | When to Add |
|------------|---------|-------------|
| **Sentry** | Error tracking | Production monitoring |
| **OpenTelemetry** | Distributed tracing | Microservices, compliance audits |
| **Prometheus** | Metrics | Performance monitoring |

---

## Infrastructure

### Docker Images

| Image | Purpose | Size |
|-------|---------|------|
| node:20-alpine | Application | ~180MB |
| postgres:15-alpine | Database | ~240MB |
| redis:7-alpine | Cache | ~30MB |
| nginx:alpine | Reverse proxy | ~25MB |
| livekit/livekit-server | Video | ~50MB |

### Reverse Proxy Options

| Technology | Use Case | Complexity |
|------------|----------|------------|
| **nginx** | Standard deployments | Low |
| Traefik | Docker-native, auto-discovery | Medium |
| Caddy | Automatic HTTPS | Low |

---

## Validation

### Zod (Recommended)

| Technology | Version | Purpose | Why Chosen |
|------------|---------|---------|------------|
| **Zod** | 3.22+ | Schema validation | TypeScript inference, runtime checks |

**API Validation Pattern:**
```typescript
import { z } from 'zod';

// Schema definition
const CreateBookingSchema = z.object({
  eventTypeId: z.string().uuid(),
  startTime: z.string().datetime(),
  attendees: z.array(z.object({
    email: z.string().email(),
    name: z.string().min(1),
  })).min(1),
  notes: z.string().max(1000).optional(),
});

// Type inference
type CreateBookingInput = z.infer<typeof CreateBookingSchema>;

// Fastify integration
app.post('/bookings', {
  schema: {
    body: CreateBookingSchema,
  },
}, async (req, res) => {
  const data = req.body as CreateBookingInput;  // Type-safe
  // ...
});
```

---

## Date & Time

| Technology | Version | Purpose | Why Chosen |
|------------|---------|---------|------------|
| **date-fns** | 3.x | Date utilities | Tree-shakeable, immutable |
| **date-fns-tz** | 3.x | Timezone handling | Critical for scheduling |

**Timezone Best Practices:**
```typescript
import { zonedTimeToUtc, utcToZonedTime, format } from 'date-fns-tz';

// Always store as UTC
const storeBooking = (localTime: Date, timezone: string) => {
  const utcTime = zonedTimeToUtc(localTime, timezone);
  return db.booking.create({
    data: { startTime: utcTime, timezone }
  });
};

// Display in user's timezone
const displayBooking = (booking: Booking) => {
  const localTime = utcToZonedTime(booking.startTime, booking.timezone);
  return format(localTime, 'PPP p', { timeZone: booking.timezone });
};
```

---

## Testing

| Technology | Version | Purpose | Why Chosen |
|------------|---------|---------|------------|
| **Vitest** | 1.x | Test runner | Vite-native, Jest-compatible |
| @vitest/coverage-v8 | 1.x | Coverage | V8 coverage, no instrumentation |
| Testing Library | 14.x | React testing | User-centric testing |
| MSW | 2.x | API mocking | Request interception |

---

## Development Tools

| Technology | Version | Purpose |
|------------|---------|---------|
| **tsx** | 4.x | Run TypeScript directly |
| ESLint | 8.x | Code linting |
| Prettier | 3.x | Code formatting |
| Husky | 8.x | Git hooks |
| lint-staged | 15.x | Staged file linting |
| concurrently | 8.x | Parallel commands |

---

## Integration Providers

### OAuth Providers

| Provider | Purpose | Scopes Needed |
|----------|---------|---------------|
| Google | Authentication | email, profile |
| Microsoft | Authentication | email, profile |
| Apple | Authentication | email, name |

### Calendar Sync

| Provider | Protocol | Notes |
|----------|----------|-------|
| Google Calendar | REST API | OAuth required |
| Microsoft/Outlook | Microsoft Graph | OAuth required |
| CalDAV | CalDAV protocol | Generic, self-hosted calendars |

### Notifications

| Channel | Provider | Purpose |
|---------|----------|---------|
| Email | **Resend** | Transactional emails |
| SMS | Twilio | Booking reminders |
| Push | Web Push API | Browser notifications |

### Video Conferencing

| Provider | Purpose | Self-Hosted? |
|----------|---------|--------------|
| **LiveKit** | Primary video | ✅ Yes |
| Google Meet | Integration | ❌ No |
| Zoom | Integration | ❌ No |

---

## Package.json Structure

```json
{
  "name": "app",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "concurrently \"npm run dev:api\" \"npm run dev:web\"",
    "dev:api": "npm run dev -w apps/api",
    "dev:web": "npm run dev -w apps/web",
    "build": "npm run build -w apps/api && npm run build -w apps/web",
    "test": "vitest",
    "lint": "eslint . --ext .ts,.tsx",
    "format": "prettier --write .",
    "prepare": "husky install"
  },
  "dependencies": {
    "prisma": "^5.22.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "typescript": "^5.3.3",
    "vitest": "^1.1.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.1",
    "husky": "^8.0.3",
    "lint-staged": "^15.2.0",
    "concurrently": "^8.2.2"
  }
}
```

---

## Compliance Considerations

See [Security & Compliance Guide](./security-compliance.md) for detailed requirements.

### Quick Reference

| Technology | SOC 2 | HIPAA | PCI DSS | Action Required |
|------------|-------|-------|---------|-----------------|
| PostgreSQL | ✅ | ✅ | ✅ | Enable SSL, encryption at rest |
| Redis | ⚠️ | ⚠️ | ⚠️ | Add TLS, authentication |
| Socket.io | ⚠️ | ⚠️ | ⚠️ | Token auth, signed payloads |
| Pino (logging) | ✅ | ⚠️ | ✅ | Redact PHI/PII |
| LiveKit | ✅ | ✅ | N/A | End-to-end encryption option |
| Resend | ✅ | ✅ | N/A | BAA available |

---

## References

### Official Documentation
- [Fastify Documentation](https://www.fastify.io/docs/latest/)
- [Prisma Documentation](https://www.prisma.io/docs)
- [React Documentation](https://react.dev/)
- [Vite Documentation](https://vitejs.dev/)
- [TailwindCSS Documentation](https://tailwindcss.com/docs)
- [Socket.io Documentation](https://socket.io/docs/v4/)
- [LiveKit Documentation](https://docs.livekit.io/)
- [Resend Documentation](https://resend.com/docs)

### Security Resources
- [OWASP Cheat Sheets](https://cheatsheetseries.owasp.org/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [Fastify Security](https://www.fastify.io/docs/latest/Reference/Security/)
