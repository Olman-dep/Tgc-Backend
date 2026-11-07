# 🃏 TCG Market Tracker — Backend

API REST para **gestionar colecciones de Pokémon TCG, consultar precios, registrar compras y analizar la evolución del valor de una colección**.

Este documento describe el contexto, la arquitectura y las decisiones del backend.

---

## 1. Stack

| Área | Tecnología |
|---|---|
| Lenguaje | Kotlin 2.x (JVM 21) |
| Framework | Spring Boot 3.x |
| Persistencia | Spring Data JPA (Hibernate) + PostgreSQL 16 |
| Migraciones | Flyway |
| Seguridad | Spring Security + JWT (jjwt) + BCrypt |
| Validación | Bean Validation (`jakarta.validation`) |
| Documentación | springdoc-openapi (Swagger UI) |
| Cliente HTTP | `RestClient` |
| Jobs | `@Scheduled` |
| Rate limiting | Bucket4j |
| Tests | JUnit 5, MockK, SpringMockK, Testcontainers |
| Build | Gradle (Kotlin DSL) |
| Contenedores | Docker + Docker Compose |

---

## 2. Qué hace el backend

1. Registro e inicio de sesión de usuarios (JWT).
2. Búsqueda de cartas consumiendo la API externa de Pokémon TCG (el frontend **nunca** llama directo a la API externa).
3. Gestión de la colección de cada usuario (cantidad, precio de compra, fecha, condición, idioma).
4. Cálculo de valor actual, inversión, ganancia y ROI.
5. Historial de precios por carta.
6. Alertas de precio.
7. Job programado que actualiza precios y evalúa alertas.
8. Endpoint de dashboard con estadísticas agregadas.

```text
React ──► Backend (Spring Boot) ──► API externa Pokémon TCG
                │
                ├──► PostgreSQL
                └──► Scheduler (precios + alertas)
```

---

## 3. Arquitectura

Arquitectura por capas, organizada **por feature**:

```text
backend/
├── build.gradle.kts
├── Dockerfile
└── src/
    ├── main/
    │   ├── kotlin/com/tcgtracker/
    │   │   ├── TcgTrackerApplication.kt
    │   │   ├── auth/            controller, service, dto
    │   │   ├── user/            entity, repository
    │   │   ├── cards/           controller, service, repository, dto, mapper
    │   │   ├── collection/      controller, service, repository, dto
    │   │   ├── prices/          controller, service, repository
    │   │   ├── alerts/          controller, service, repository
    │   │   ├── dashboard/       controller, service
    │   │   ├── infrastructure/
    │   │   │   ├── externalapi/ cliente de la API Pokémon TCG
    │   │   │   └── scheduler/   jobs programados
    │   │   ├── config/          security, cors, openapi, rate limit
    │   │   └── common/          excepciones, handler global
    │   └── resources/
    │       ├── application.yml
    │       └── db/migration/V1__init.sql
    └── test/
```

### Reglas

- Los **controllers** solo reciben/devuelven DTOs; la lógica vive en los **services**.
- Las **entidades JPA nunca se exponen** directamente.
- Las entidades JPA son clases normales (no `data class`); los DTOs sí son `data class`.
- Toda consulta de colección o alertas se filtra por el usuario autenticado.
- Manejo global de errores con `@RestControllerAdvice`.

---

## 4. Modelo de datos

```text
User ──< CollectionItem >── Card ──< PriceHistory
 │
 └──< PriceAlert >── Card
```

### users
`id, email (unique), username (unique), password (BCrypt), role (USER|ADMIN), created_at, updated_at`

### cards
`id, external_id (unique), name, set_id, set_name, number, rarity, image_url, market_price, created_at, updated_at`

### collection_items
`id, user_id, card_id, quantity, purchase_price, purchase_date, condition, language, created_at, updated_at`

### price_history
`id, card_id, price, currency, source, recorded_at`

### price_alerts
`id, user_id, card_id, target_price, active, created_at, triggered_at`

> **Moneda:** pokemontcg.io entrega precios en USD. Se guarda el precio original con su `currency` y se convierte a COP al mostrar (o al ingresar al historial), según la decisión que se tome.

---

## 5. Endpoints

### Auth
```http
POST /api/auth/register
POST /api/auth/login
```

### Cards
```http
GET /api/cards
GET /api/cards/{id}
GET /api/cards/search
GET /api/cards/{id}/prices
```

### Collection
```http
GET    /api/collection
POST   /api/collection
GET    /api/collection/{id}
PUT    /api/collection/{id}
DELETE /api/collection/{id}
```

### Alerts
```http
GET    /api/alerts
POST   /api/alerts
PUT    /api/alerts/{id}
DELETE /api/alerts/{id}
```

### Dashboard
```http
GET /api/dashboard
```

Las rutas privadas requieren `Authorization: Bearer <token>`.
Swagger UI: `http://localhost:8080/swagger-ui.html`

---

## 6. Cálculos

```text
currentValue = Σ(currentPrice × quantity)
investment   = Σ(purchasePrice × quantity)
profit       = currentValue − investment
ROI (%)      = (profit / investment) × 100
```

Usar `BigDecimal` para dinero (nunca `Double`) y controlar la división por cero cuando `investment = 0`.

---

## 7. Seguridad

- JWT firmado con `JWT_SECRET` (mínimo 256 bits).
- Contraseñas con BCrypt.
- Sesión stateless.
- CORS limitado a `FRONTEND_URL`.
- Validación de inputs en todos los DTOs.
- Rate limiting básico (Bucket4j).
- Secretos y API keys solo por variables de entorno.
- No loguear contraseñas, JWT ni API keys.

---

## 8. Jobs programados

Cada 6 horas:

```text
1. Obtener cartas relevantes (las que están en colecciones o alertas)
2. Consultar la API externa
3. Actualizar Card.marketPrice
4. Insertar registro en PriceHistory
5. Evaluar PriceAlerts activas (precio <= target)
6. Marcar triggeredAt / notificar
```

Logs esperados:

```text
INFO  Price update started
INFO  120 cards processed
WARN  5 cards failed
INFO  Price update completed
```

---

## 9. Variables de entorno

```env
DATABASE_URL=jdbc:postgresql://localhost:5432/tcg
DATABASE_USERNAME=
DATABASE_PASSWORD=

JWT_SECRET=
JWT_EXPIRATION_MINUTES=60

POKEMON_API_URL=https://api.pokemontcg.io/v2
POKEMON_API_KEY=

FRONTEND_URL=http://localhost:5173
```

Nunca subir `.env` a GitHub.

---

## 10. Cómo ejecutar

```bash
# 1. Base de datos
docker compose up -d postgres

# 2. Backend
cd backend
./gradlew bootRun

# 3. Tests
./gradlew test
```

---

## 11. Testing

Casos clave:

- Login correcto e incorrecto.
- Acceso sin token (401).
- Usuario accediendo a la colección de otro (403/404).
- Agregar, actualizar y eliminar items de colección.
- Cálculo de ROI.
- Crear y ejecutar alertas.

Integración con **Testcontainers (PostgreSQL)**.

---

## 12. Plan por fases

**MVP:** auth, búsqueda de cartas, colección, dashboard, historial de precios.
**V2:** alertas, filtros avanzados, rankings, emails.
**V3+:** importar/exportar CSV, perfil público, marketplace.

> Prioridad: MVP → funcionamiento → calidad → testing → deployment → features extra.

---

## 13. Convenciones

- Commits: `feat:`, `fix:`, `test:`, `docs:`, `refactor:`.
- Ramas: `main`, `develop`, `feature/*`.
- Principios: SOLID, DRY, separación de responsabilidades, DTOs, inyección de dependencias.
