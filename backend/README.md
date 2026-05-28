# DIUM — Fast Food Pre-ordering at Universities

REST API backend for the DIUM platform: students place orders at university cafes in advance, skipping queues. The venue owner receives a WhatsApp notification, manages the order status, and accepts payment via Kaspi.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language / Platform | Java 17, Spring Boot 4.0.x |
| Database | PostgreSQL + Spring Data JPA |
| Security | Spring Security, JWT (jjwt 0.11) |
| Notifications | WhatsApp (Green API) |
| Image Storage | Cloudinary |
| Cache | Caffeine (in-memory JCache) |
| Rate Limiting | Bucket4j |
| DTO Mapping | MapStruct |
| API Documentation | SpringDoc OpenAPI (Swagger UI) |
| Containerization | Docker (multi-stage build) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         HTTP Clients                            │
│                  (Mobile App / Web Frontend)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │  HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Spring Boot App                            │
│                                                                 │
│  ┌──────────────┐   ┌─────────────────────────────────────────┐ │
│  │  Bucket4j    │   │         Security Layer                  │ │
│  │ Rate Limiter │   │  JwtAuthenticationFilter + SecurityConfig│ │
│  └──────┬───────┘   └───────────────────┬─────────────────────┘ │
│         │                               │                       │
│         ▼                               ▼                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     Controllers                          │   │
│  │  AuthController  OrderController  VenueController        │   │
│  │  ProductController               ModifierController      │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │ DTO                               │
│                             ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      Services                            │   │
│  │  AuthService    OrderService    VenueService             │   │
│  │  ProductService ModifierService JwtService               │   │
│  │  FileService    OrderExpirationService                   │   │
│  └──────────┬───────────────────────────┬───────────────────┘   │
│             │                           │                       │
│             │ AOP (AfterReturning)      │                       │
│             ▼                           ▼                       │
│  ┌──────────────────┐   ┌──────────────────────────────────┐    │
│  │MessageSenderAspect│  │        Repositories              │    │
│  │ (WhatsApp notify) │  │  UserRepo  OrderRepo  VenueRepo  │    │
│  └────────┬─────────┘  │  ProductRepo  OtpRepo  ...       │    │
│           │             └──────────────┬───────────────────┘    │
│           │                            │                        │
└───────────┼────────────────────────────┼────────────────────────┘
            │                            │
            ▼                            ▼
   ┌─────────────────┐          ┌─────────────────┐
   │  WhatsApp API   │          │   PostgreSQL DB  │
   │  (Green API)    │          │                 │
   └─────────────────┘          └─────────────────┘
            │
            ▼
   ┌─────────────────┐
   │   Cloudinary    │
   │ (Image Storage) │
   └─────────────────┘
```

### Application Layers

**Controllers** — receive HTTP requests, validate input (`@Valid`), delegate business logic to services. Access control via `@PreAuthorize`.

**Services** — all business logic. Each service is responsible for one domain area.

**Repositories** — database access via Spring Data JPA. `BaseRepository` extends `JpaRepository` and adds a `findByIdOrThrow` method.

**DTOs + Mappers** — requests and responses are isolated from models. MapStruct generates mappings at compile time.

**AOP Aspect** — `MessageSenderAspect` intercepts a successful return from `OrderService.createOrder` and asynchronously sends a WhatsApp notification to the venue owner.

---

## Domain Model

```
User (CLIENT / VENUE_OWNER / ADMIN)
 │
 ├── owns ──► Venue
 │             ├── has ──► Product
 │             │             └── has ──► ModifierGroup
 │             │                          └── has ──► Modifier
 │             └── receives ──► Order
 │                               └── contains ──► OrderItem
 │                                                  └── uses ──► Modifier[]
 └── places ──► Order
```

### Entities

| Entity | Description |
|---|---|
| `User` | System user; implements `UserDetails`; identified by phone number |
| `Venue` | Venue (cafe/canteen) at a university; each venue has one `VENUE_OWNER` |
| `Product` | Menu item with a category (`ProductCategory`), price, and availability flag |
| `ModifierGroup` | A group of options for a product (e.g. "Sauces", "Size") |
| `Modifier` | A specific option with a price delta (`priceDelta`) |
| `Order` | Customer order; contains a list of `OrderItem`, delivery address, status, `pickupCode` |
| `OrderItem` | Order line: product × quantity × price at time of purchase + selected modifiers |
| `OtpCode` | OTP code for phone confirmation/password reset; stored in Caffeine cache (TTL 5 min) |

---

## Order Lifecycle

```
CLIENT places order
        │
        ▼
    PENDING  ──── VENUE_OWNER rejects ──► CANCELLED
        │                                 (WhatsApp → CLIENT)
        │ VENUE_OWNER accepts
        ▼
   PREPARING  ◄─ WhatsApp notification → CLIENT ("Order is being prepared")
        │
        │ VENUE_OWNER ready to hand out
        ▼
     READY    ◄─ WhatsApp notification → CLIENT ("Order ready, code: XXXX")
        │
        │ VENUE_OWNER completes
        ▼
   COMPLETED
```

`pickupCode` generation: deterministic formula `(id × 48271) % 9000 + 1000` — always a 4-digit number.

---

## Authentication

All protected endpoints require the `Authorization: Bearer <JWT>` header.

```
Register → OTP via WhatsApp → Activate → JWT token
```

1. `POST /api/v1/auth/register` — creates a user (`isConfirmed = false`), sends OTP via WhatsApp
2. `POST /api/v1/auth/activate?phone=&code=` — confirms phone number, returns JWT
3. `POST /api/v1/auth/login` — login by phone/password, returns JWT
4. `POST /api/v1/auth/forgot-password` — OTP for password reset
5. `POST /api/v1/auth/reset-password` — set a new password via OTP

Account is locked (`isAccountNonLocked = false`) while `isConfirmed = false`.

---

## Roles and Permissions

| Role | Capabilities |
|---|---|
| `CLIENT` | Register, browse menus/venues, create and view own orders |
| `VENUE_OWNER` | Manage menu and products of their venue, process orders (status / cancel), view kitchen queue and history |
| `ADMIN` | Register venue owners (`/api/v1/auth/venue-register`), full venue management |

---

## Rate Limiting (Bucket4j + Caffeine)

| Endpoint | Limit |
|---|---|
| `/api/auth/**` | 5 requests / minute per IP |
| `/api/whatsapp/**` | 10 requests / hour per IP |
| All other `/api/**` | 100 requests / minute per IP |

On exceeded limit: `HTTP 429 Too Many Requests`.

---

## API Endpoints (brief overview)

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/api/v1/auth/register` | — | Register a client |
| POST | `/api/v1/auth/login` | — | Login |
| POST | `/api/v1/auth/activate` | — | Confirm phone number |
| GET | `/api/v1/auth/me` | AUTH | Current user profile |
| GET | `/api/v1/venues` | — | List of venues |
| POST | `/api/v1/venues` | ADMIN | Create a venue |
| GET | `/api/v1/venues/{id}/products` | — | Venue menu |
| POST | `/api/v1/products` | VENUE_OWNER | Add a product |
| POST | `/api/v1/orders` | AUTH | Create an order |
| GET | `/api/v1/orders/user` | AUTH | My orders |
| GET | `/api/v1/orders/kitchen` | VENUE_OWNER | Active kitchen orders |
| PATCH | `/api/v1/orders/{id}/status` | VENUE_OWNER | Update order status |
| PATCH | `/api/v1/orders/{id}/cancel` | VENUE_OWNER | Cancel an order |
| GET | `/api/v1/orders/{id}/kaspi-url` | AUTH | Kaspi payment URL |

Full interactive documentation: `GET /swagger-ui.html`

---

## Environment Variables

| Variable | Description | Dev Default |
|---|---|---|
| `JWT_KEY` | Secret for signing JWT | `default_secret_for_dev_only_12345` |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name | `temp` |
| `CLOUDINARY_API_KEY` | Cloudinary API key | `12345` |
| `CLOUDINARY_API_SECRET` | Cloudinary API secret | `secret` |
| `WHATSAPP_API_URL` | Green API base URL | — |
| `WHATSAPP_INSTANCE_ID` | Green API instance ID | — |
| `WHATSAPP_API_TOKEN` | Green API token | — |
| `PORT` | Server port | `8080` |

---

## Running

### Locally (dev profile)

```bash
# Configure environment variables in .env or environment
./mvnw spring-boot:run
```

### Docker

```bash
docker build -t dium-backend .
docker run -p 8080:8080 \
  -e JWT_KEY=... \
  -e WHATSAPP_API_URL=... \
  -e WHATSAPP_INSTANCE_ID=... \
  -e WHATSAPP_API_TOKEN=... \
  -e CLOUDINARY_CLOUD_NAME=... \
  -e CLOUDINARY_API_KEY=... \
  -e CLOUDINARY_API_SECRET=... \
  dium-backend
```

The prod profile is activated automatically via `ENTRYPOINT` in the Dockerfile.

### Tests

```bash
./mvnw test
```

---

## Project Structure

```
src/main/java/com/dium/demo/
├── aspects/          # AOP: WhatsApp notifications after order creation
├── config/           # Spring configuration (Security, Cache, WhatsApp)
├── controllers/      # REST controllers
├── dto/
│   ├── requests/     # Incoming DTOs
│   └── responses/    # Outgoing DTOs
├── enums/            # OrderStatus, UserRole, ProductCategory
├── exceptions/       # BusinessLogicException, AccessDeniedException, GlobalExceptionHandler
├── mappers/          # MapStruct entity ↔ DTO mapping
├── models/           # JPA entities
├── repositories/     # Spring Data repositories
├── security/         # JWT filter, SecurityConfig
├── services/         # Business logic
└── whatsapp/         # WhatsApp API client
```
