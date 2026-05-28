# Dium

Angular 21 food ordering platform with SSR. Customers browse venues, add items to cart, and pay via Kaspi. Venue owners manage menus and track incoming orders through a kitchen panel.

## Tech stack

| Layer | Technology |
|---|---|
| Framework | Angular 21 (standalone components) |
| Rendering | Angular SSR (Express) |
| Styling | TailwindCSS v4 |
| i18n | ngx-translate |
| HTTP | Angular HttpClient + functional interceptor |
| State | BehaviorSubject / RxJS signals |
| Auth | JWT (localStorage) with auto-expiry timer |
| Backend | REST API — `https://dium-backend.onrender.com/api/v1` |

## Getting started

```bash
npm install
npm start          # dev server at http://localhost:4200
npm run build      # production build → dist/
npm run serve:ssr:dium   # run SSR build
```

Environment files live in `src/environments/`:

| File | `apiUrl` |
|---|---|
| `environment.development.ts` | `http://localhost:8080/api/v1` |
| `environment.ts` | `https://dium-backend.onrender.com/api/v1` |

## Routes

| Path | Component | Guard |
|---|---|---|
| `/` | LandingComponent | guestGuard |
| `/venues` | VenueListComponent | — |
| `/venue/:id` | VenueDetailComponent | — |
| `/cart` | CartComponent | — |
| `/orders` | OrdersComponent | — |
| `/order-history` | OrderHistoryComponent | — |
| `/login` | LoginComponent | guestGuard |
| `/register` | RegisterComponent | — |
| `/activate` | ActivateComponent | — |
| `/forgot-password` | ForgotPasswordComponent | guestGuard |
| `/profile` | ProfileComponent | — |
| `/kitchen` | KitchenPanelComponent | — |
| `/manage-venue/:id` | ManageVenueComponent | — |
| `/create-venue` | CreateVenueComponent | — |
| `/admin` | AdminRegisterOwnerComponent | — |
| `/support` | SupportComponent | — |
| `/traction` | TractionComponent | — |
| `/about` | AboutComponent | — |
| `/**` | → `/venues` | — |

## User roles

| Role | Access |
|---|---|
| `CLIENT` | Browse venues, cart, orders, profile |
| `VENUE_OWNER` | All CLIENT routes + kitchen panel + venue management |
| `ADMIN` | All routes + register new venue owners |

## Architecture

```
src/
├── app/
│   ├── components/       # feature components (standalone)
│   ├── services/
│   │   ├── auth.service.ts   # JWT lifecycle, user state stream
│   │   └── cart.service.ts   # cart state, checkout, delivery fee
│   ├── interceptors/
│   │   └── auth.interceptor.ts  # attaches Bearer token; skips i18n assets
│   ├── guards/
│   │   └── guest.guard.ts    # redirects authenticated users away from auth pages
│   └── models/
│       └── api.models.ts     # all shared TypeScript interfaces
├── environments/             # environment configs
├── server.ts                 # Express SSR entry point
└── main.ts / main.server.ts  # bootstrap
```

## Key flows

### Authentication
1. `POST /auth/register` → SMS sent to phone
2. `POST /auth/activate?phone&code` → JWT returned, stored in localStorage
3. `authInterceptor` attaches `Authorization: Bearer <token>` to every API request
4. JWT expiry is decoded client-side; `setTimeout` calls `logout()` at expiry

### Cart & checkout
- Cart persists in localStorage across sessions
- Cart is scoped to a single venue — adding a product from a different venue prompts a clear
- Delivery mode toggles a fee from the venue's `deliveryPrice` field
- `CartService.checkout()` posts an `OrderRequest` to `/orders`
- After placing the order, the user pays via a Kaspi redirect URL fetched from `/orders/:id/kaspi-url`

### Venue list ordering
Venues are sorted by: **open + in stock** → **closed** → **open but sold out**, then alphabetically within each group.

## API models

| Interface | Description |
|---|---|
| `User` | Authenticated user (phone, name, role, paymentFrom) |
| `Venue` | Restaurant/cafe metadata |
| `Product` | Menu item with modifiers flag |
| `Modifier` / `ModifierGroup` | Product add-ons (e.g. sizes, toppings) |
| `CartItem` | `Product` extended with count and selected modifiers |
| `OrderRequest` / `OrderResponse` | Order create payload and server response |

## Order statuses

| Status | Meaning |
|---|---|
| `PENDING` | Awaiting payment |
| `PREPARING` | Kitchen accepted, cooking |
| `READY` | Ready for pickup |
| `COMPLETED` | Picked up by customer |
| `CANCELLED` | Cancelled |

## Running tests

```bash
npm test   # Vitest
```
