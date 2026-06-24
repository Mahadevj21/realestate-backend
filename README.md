# PropManage (RealEstate)

PropManage is a full-stack property listing app with three roles: buyers browse approved listings and request purchases, sellers manage listings and respond to offers, and admins moderate listings and view platform aggregates. Balances on user records act as an internal wallet; completing a sale moves money between buyer, seller, and admin accounts and writes transaction rows. The repo splits into `backend/` (Spring Boot) and `frontend/` (Create React App).

## Roles and what each can do

**Buyer** (`User.Role.BUYER`, routes under `/buyer`)

- Browse approved unsold listings: `GET /buyer/available`, search `GET /buyer/search`, filters under `/buyer/filter/*` (public, no token).
- Request a purchase: `POST /buyer/buy/{propertyId}/buyer/{buyerId}` creates a `PENDING` deal if the property is not sold and the buyer's balance is at least the list price. No money moves yet.
- View own balance, deals, and favourites (paths include `{buyerId}`; controller compares that ID to the JWT `userId` in `SecurityContext`).
- Add/remove/list favourites for properties.

**Seller** (`User.Role.SELLER`, routes under `/seller`)

- Add listings via `POST /properties/add/{sellerId}` (seller ID must match token). New listings start `approved=false`.
- Update or delete own properties (`PropertyController` checks ownership via `getCurrentUserId()`).
- View own properties, pending deals, all deals, balance, and a simple active-vs-sold count for charts.
- Accept or reject pending deals: `POST /seller/deals/{dealId}/accept` or `/reject`. Accept triggers payment finalization.

**Admin** (`User.Role.ADMIN`, routes under `/admin`, requires `ROLE_ADMIN`)

- List all users; block/unblock users.
- List all properties; approve or reject listings (`approveProperty` / `rejectProperty` toggles the `approved` flag).
- View completed deals, own brokerage balance, platform stats, six-month growth chart data, and a summary report.

Unauthenticated users can register and log in (`/auth/register`, `/auth/login`) and read the public property search endpoints listed above.

## Tech stack

**Backend** (`backend/`, Java 17)

- Spring Boot 3.5.10
- Spring Data JPA, Spring Security, Spring Web, Spring Mail
- PostgreSQL 42.7.0 (primary DB in `application.properties`)
- H2 on classpath (runtime) but not configured as the active datasource
- JJWT 0.11.5
- Lombok

**Frontend** (`frontend/`)

- React 18.2, react-scripts 5
- axios 1.4, leaflet / react-leaflet, recharts 2.12
- Testing Library packages present; one default `App.test.js`

## How the wallet/transaction flow works

**1. Listing.** A seller creates a property. It is not visible to buyers until an admin calls approve (`approved=true`, `sold=false`).

**2. Purchase request.** A buyer calls `DealService.createDealRequest`. The service rejects if `property.isSold()` is true or if `buyer.balance < property.price`. It creates a `Deal` with status `PENDING` and `amount = property.price`. The buyer's balance is not debited at this step.

**3. Seller decision.** Accept calls `acceptDeal`, which requires status `PENDING`, sets `ACCEPTED`, then immediately calls `finalizeDeal`. Reject sets `REJECTED` with no balance changes.

**4. Finalization** (`finalizeDeal`, `@Transactional` on `DealService`). The service loads buyer, seller, property, and the first admin user in the database. Brokerage is a fixed `100.0` per side (buyer pays `price + 100`, seller receives `price - 100`, admin receives `200`).

If `buyer.balance < price + 100`, it throws and rolls back the transaction.

Otherwise it mutates balances on the in-memory `User` entities, sets `property.sold = true`, sets deal status `COMPLETED` with `completedAt`, saves users/property/deal, then calls `TransactionService.createTransaction` five times (buyer debit for price, seller credit for price, buyer debit for fee, seller debit for fee, admin credit for total fees). All created transactions are stored with status `SUCCESS`.

**5. Email.** `EmailService` sends plain-text messages to buyer and seller on successful completion.

`TransactionService.processPayment` exists as a separate buyer-only debit helper but the deal flow above does not call it; the live purchase path is deal accept → `finalizeDeal`.

**Concurrent actions on the same listing.** `createDealRequest` only checks `isSold()` at request time. Multiple buyers can each create a `PENDING` deal on the same unsold property. `finalizeDeal` does not re-check whether the property was already marked sold before moving money. If two sellers accepted two pending deals on one property (or a race between two accept calls), the code as written could apply balance changes more than once. There is no row-level lock, optimistic versioning, or unique constraint preventing duplicate pending deals per property.

## Setup

### Prerequisites

Java 17, Maven, Node.js, PostgreSQL.

### Database

Create database `realestate` (matches default JDBC URL in `backend/src/main/resources/application.properties`).

### Backend

Configure via `application.properties` or environment variables:

| Variable | Used for |
|---|---|
| `DB_USERNAME` / `DB_PASSWORD` | PostgreSQL credentials (defaults `postgres` / `root`) |
| `JWT_SECRET` | JWT signing |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | Gmail SMTP for deal emails |
| `ALLOWED_ORIGINS` | CORS (`app.cors.allowed-origins`, default `http://localhost:3000,http://localhost:8080`) |

```bash
cd backend
mvn spring-boot:run
```

On an empty database, `DataInitializer` seeds admin, seller, and buyer users with password `admin123` and sample properties.

### Frontend

Root `.env.example` documents `REACT_APP_API_BASE_URL`, but `frontend/src/services/apiService.js` currently hardcodes `const API_BASE_URL = 'http://localhost:8080'`.

```bash
cd frontend
npm install
npm start
```

Runs on port 3000 by default (Create React App).

## Architecture decisions

**Deal-based purchase instead of immediate payment.** Buyers express intent with a pending deal; funds are checked twice (at request and at accept) but only move when the seller accepts. That models seller approval but leaves a window where balance is not reserved between request and accept.

**JWT carries `userId` and `role` claims; controllers enforce ownership.** `JwtAuthFilter` puts `userId` in `Authentication.details`. Buyer and seller controllers call `validateUser(pathId)` before returning or mutating user-specific data. Property delete/update uses the same pattern in `PropertyService`.

**Admin approval gate for buyer-visible inventory.** `getApprovedProperties` and search/filter repository methods only return `approved=true` listings, so new seller submissions stay hidden until admin action.

**Fixed brokerage in code.** The ₹100-per-side fee and admin credit of `brokerageFee * 2` are literals in `finalizeDeal`, not configurable properties.

## Known limitations

**No concurrency control on deals or sold flag.** As described above, duplicate pending deals and accept races are not handled.

**Balance is not reserved when a deal is pending.** A buyer can pass the initial balance check, spend the same money elsewhere (if another code path debited it), and fail only at accept/finalize—or pass checks inconsistently across concurrent requests.

**Frontend API URL not driven by env in code.** `apiService.js` ignores `REACT_APP_API_BASE_URL` despite `.env.example`.

**Minimal automated tests.** Backend has only `RealestateApplicationTests` (context load). Frontend has the default CRA smoke test.

**Default JWT secret in properties file.** `application.properties` includes a long default `jwt.secret` if `JWT_SECRET` is unset, which is unsafe for any deployment that forgets to override it.

**Seed credentials in source.** `DataInitializer` documents password `admin123` for all seeded accounts.

## License

Not specified in repository files.
