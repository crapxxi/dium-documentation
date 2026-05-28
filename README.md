# DIUM — Fast Food Pre-ordering at Universities

A platform for pre-ordering food at university cafes and canteens. Students place orders in advance, skipping the queues. Payment via Kaspi. The venue owner receives a WhatsApp notification and manages order statuses in real time.

**Website:** [dium.kz](https://dium.kz)

---

> **Source code repositories are hidden for security reasons.**

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Backend | Java 17, Spring Boot, PostgreSQL, Spring Security + JWT |
| Frontend | Angular 21 (SSR), TailwindCSS v4 |
| Notifications | WhatsApp (Green API) |
| Payments | Kaspi |
| Image Storage | Cloudinary |
| Cache / Rate Limiting | Caffeine, Bucket4j |
| Containerization | Docker |

## User Roles

| Role | Capabilities |
|---|---|
| `CLIENT` | Browse menus and venues, place orders, make payments |
| `VENUE_OWNER` | Manage menus, process and track kitchen orders |
| `ADMIN` | Register venue owners, manage venues |

## Order Lifecycle

```
CLIENT places order → PENDING → PREPARING → READY → COMPLETED
                                        └──────────────► CANCELLED
```

The client receives a WhatsApp notification at each stage.

## Documentation

- [Backend](./backend/README.md) — REST API, architecture, endpoints, environment variables
- [Frontend](./frontend/README.md) — Angular app, routes, architecture, key flows
