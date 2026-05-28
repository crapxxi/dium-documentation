# DIUM — Быстрый предзаказ еды в университетах

REST API бэкенд для платформы DIUM: студенты оформляют заказы в университетских кафе заранее, минуя очереди. Владелец заведения получает уведомление в WhatsApp, управляет статусом заказа и принимает оплату через Kaspi.

---

## Стек технологий

| Слой | Технология |
|---|---|
| Язык / платформа | Java 17, Spring Boot 4.0.x |
| БД | PostgreSQL + Spring Data JPA |
| Безопасность | Spring Security, JWT (jjwt 0.11) |
| Уведомления | WhatsApp (Green API) |
| Хранение изображений | Cloudinary |
| Кэш | Caffeine (in-memory JCache) |
| Rate limiting | Bucket4j |
| Маппинг DTO | MapStruct |
| Документация API | SpringDoc OpenAPI (Swagger UI) |
| Контейнеризация | Docker (multi-stage build) |

---

## Архитектура

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

### Слои приложения

**Controllers** — принимают HTTP-запросы, валидируют входные данные (`@Valid`), делегируют бизнес-логику сервисам. Разграничение доступа через `@PreAuthorize`.

**Services** — вся бизнес-логика. Каждый сервис отвечает за одну доменную область.

**Repositories** — доступ к БД через Spring Data JPA. `BaseRepository` расширяет `JpaRepository` и добавляет метод `findByIdOrThrow`.

**DTOs + Mappers** — запросы и ответы изолированы от моделей. MapStruct генерирует маппинг на этапе компиляции.

**AOP Aspect** — `MessageSenderAspect` перехватывает успешный возврат из `OrderService.createOrder` и асинхронно шлёт WhatsApp-уведомление владельцу заведения.

---

## Доменная модель

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

### Сущности

| Сущность | Описание |
|---|---|
| `User` | Пользователь системы; реализует `UserDetails`; идентифицируется по номеру телефона |
| `Venue` | Заведение (кафе/буфет) в университете; у каждого заведения один `VENUE_OWNER` |
| `Product` | Позиция меню с категорией (`ProductCategory`), ценой и флагом наличия |
| `ModifierGroup` | Группа опций к продукту (например «Соусы», «Размер») |
| `Modifier` | Конкретная опция с дельтой цены (`priceDelta`) |
| `Order` | Заказ клиента; содержит список `OrderItem`, адрес доставки, статус, `pickupCode` |
| `OrderItem` | Строка заказа: продукт × количество × цена на момент покупки + выбранные модификаторы |
| `OtpCode` | OTP-код для подтверждения телефона/сброса пароля; хранится в Caffeine-кэше (TTL 5 мин) |

---

## Жизненный цикл заказа

```
CLIENT создаёт заказ
        │
        ▼
    PENDING  ──── VENUE_OWNER отклоняет ──► CANCELLED
        │                                    (WhatsApp → CLIENT)
        │ VENUE_OWNER принимает
        ▼
   PREPARING  ◄─ WhatsApp уведомление → CLIENT ("Заказ готовится")
        │
        │ VENUE_OWNER готов выдать
        ▼
     READY    ◄─ WhatsApp уведомление → CLIENT ("Заказ готов, код: XXXX")
        │
        │ VENUE_OWNER завершает
        ▼
   COMPLETED
```

Генерация `pickupCode`: детерминированная формула `(id × 48271) % 9000 + 1000` — всегда 4-значное число.

---

## Аутентификация

Все защищённые эндпоинты требуют заголовок `Authorization: Bearer <JWT>`.

```
Регистрация → OTP в WhatsApp → Активация → JWT-токен
```

1. `POST /api/v1/auth/register` — создаёт пользователя (`isConfirmed = false`), отправляет OTP в WhatsApp
2. `POST /api/v1/auth/activate?phone=&code=` — подтверждает телефон, возвращает JWT
3. `POST /api/v1/auth/login` — вход по телефону/паролю, возвращает JWT
4. `POST /api/v1/auth/forgot-password` — OTP для сброса пароля
5. `POST /api/v1/auth/reset-password` — установить новый пароль по OTP

Аккаунт заблокирован (`isAccountNonLocked = false`) пока `isConfirmed = false`.

---

## Роли и права

| Роль | Возможности |
|---|---|
| `CLIENT` | Регистрация, просмотр меню/заведений, создание и просмотр своих заказов |
| `VENUE_OWNER` | Управление меню и продуктами своего заведения, обработка заказов (статус / отмена), просмотр кухонной очереди и истории |
| `ADMIN` | Регистрация владельцев заведений (`/api/v1/auth/venue-register`), полное управление заведениями |

---

## Rate Limiting (Bucket4j + Caffeine)

| Эндпоинт | Лимит |
|---|---|
| `/api/auth/**` | 5 запросов / минута с IP |
| `/api/whatsapp/**` | 10 запросов / час с IP |
| Все остальные `/api/**` | 100 запросов / минута с IP |

При превышении: `HTTP 429 Too Many Requests`.

---

## API Endpoints (краткий обзор)

| Метод | Путь | Роль | Описание |
|---|---|---|---|
| POST | `/api/v1/auth/register` | — | Регистрация клиента |
| POST | `/api/v1/auth/login` | — | Вход |
| POST | `/api/v1/auth/activate` | — | Подтверждение телефона |
| GET | `/api/v1/auth/me` | AUTH | Профиль текущего пользователя |
| GET | `/api/v1/venues` | — | Список заведений |
| POST | `/api/v1/venues` | ADMIN | Создать заведение |
| GET | `/api/v1/venues/{id}/products` | — | Меню заведения |
| POST | `/api/v1/products` | VENUE_OWNER | Добавить продукт |
| POST | `/api/v1/orders` | AUTH | Создать заказ |
| GET | `/api/v1/orders/user` | AUTH | Мои заказы |
| GET | `/api/v1/orders/kitchen` | VENUE_OWNER | Активные заказы кухни |
| PATCH | `/api/v1/orders/{id}/status` | VENUE_OWNER | Обновить статус заказа |
| PATCH | `/api/v1/orders/{id}/cancel` | VENUE_OWNER | Отменить заказ |
| GET | `/api/v1/orders/{id}/kaspi-url` | AUTH | Ссылка для оплаты Kaspi |

Полная интерактивная документация: `GET /swagger-ui.html`

---

## Переменные окружения

| Переменная | Описание | Dev-дефолт |
|---|---|---|
| `JWT_KEY` | Секрет для подписи JWT | `default_secret_for_dev_only_12345` |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name | `temp` |
| `CLOUDINARY_API_KEY` | Cloudinary API key | `12345` |
| `CLOUDINARY_API_SECRET` | Cloudinary API secret | `secret` |
| `WHATSAPP_API_URL` | Базовый URL Green API | — |
| `WHATSAPP_INSTANCE_ID` | ID инстанса Green API | — |
| `WHATSAPP_API_TOKEN` | Токен Green API | — |
| `PORT` | Порт сервера | `8080` |

---

## Запуск

### Локально (dev-профиль)

```bash
# Настройте переменные окружения в .env или environment
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

Prod-профиль активируется автоматически через `ENTRYPOINT` в Dockerfile.

### Тесты

```bash
./mvnw test
```

---

## Структура проекта

```
src/main/java/com/dium/demo/
├── aspects/          # AOP: WhatsApp-уведомления после создания заказа
├── config/           # Spring-конфигурация (Security, Cache, WhatsApp)
├── controllers/      # REST-контроллеры
├── dto/
│   ├── requests/     # Входящие DTO
│   └── responses/    # Исходящие DTO
├── enums/            # OrderStatus, UserRole, ProductCategory
├── exceptions/       # BusinessLogicException, AccessDeniedException, GlobalExceptionHandler
├── mappers/          # MapStruct-маппинг сущность ↔ DTO
├── models/           # JPA-сущности
├── repositories/     # Spring Data репозитории
├── security/         # JWT-фильтр, SecurityConfig
├── services/         # Бизнес-логика
└── whatsapp/         # Клиент WhatsApp API
```
