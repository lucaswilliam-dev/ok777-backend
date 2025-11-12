# Admin Backend & Database Documentation

## 📋 Table of Contents
1. [Admin Model Structure](#admin-model-structure)
2. [Admin Authentication](#admin-authentication)
3. [Admin API Endpoints](#admin-api-endpoints)
4. [Database Schema Overview](#database-schema-overview)
5. [Admin Operations](#admin-operations)
6. [Database Functions](#database-functions)
7. [Relationships & Data Flow](#relationships--data-flow)

---

## 🔐 Admin Model Structure

### Admin Table Schema
```prisma
model Admin {
  id       Int     @id @default(autoincrement())
  email    String  @unique
  password String  // Hashed with bcrypt
  role     String  // Admin role (e.g., "admin", "super_admin")
  status   String  // Account status (e.g., "active", "suspended")

  @@map("Admins")
}
```

### Key Fields:
- **id**: Auto-incrementing primary key
- **email**: Unique identifier for admin login
- **password**: Bcrypt-hashed password
- **role**: Admin role/permission level
- **status**: Account status (active, suspended, etc.)

### Database Table: `Admins`
- **Primary Key**: `id` (serial/integer)
- **Unique Constraint**: `email`
- **No Foreign Keys**: Admin table is standalone

---

## 🔑 Admin Authentication

### Login Flow (`/api/v1/admin/signin`)

**Endpoint**: `POST /api/v1/admin/signin`

**Request Body**:
```json
{
  "email": "admin@example.com",
  "password": "password123"
}
```

**Response**:
```json
{
  "code": 200,
  "data": {
    "token": "jwt_token_here",
    "user": {
      "id": 1,
      "email": "admin@example.com",
      "role": "admin",
      "status": "active"
    }
  }
}
```

**Authentication Process** (`src/db/admin.ts:login`):
1. Find admin by email in `Admins` table
2. Compare provided password with hashed password using bcrypt
3. If match, generate JWT token using `JWTPRIVATEKEY`
4. Return token and user data

**JWT Token Structure**:
- Contains full admin object (id, email, role, status)
- Signed with `process.env.JWTPRIVATEKEY`
- Used in `Authorization` header for protected routes

### Password Change (`/api/v1/admin/change-password`)

**Endpoint**: `POST /api/v1/admin/change-password` (requires `isAdmin` middleware)

**Request Body**:
```json
{
  "password": "old_password",
  "newPassword": "new_password"
}
```

**Process**:
1. Verify current password
2. Hash new password with bcrypt
3. Update admin record

---

## 🌐 Admin API Endpoints

### Base Path: `/api/v1/admin`

### Authentication Endpoints

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| POST | `/signin` | No | Admin login |
| POST | `/change-password` | Yes (`isAdmin`) | Change admin password |

### Platform Statistics

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/stats` | Yes (`isAdmin`) | Get platform statistics (users, deposits, withdrawals, bets) |

**Response Structure**:
```json
{
  "code": 200,
  "data": {
    "summary": {
      "userCount": 1000,
      "totalDeposits": 50000,
      "totalWithdrawals": 30000,
      "totalBets": 5000,
      "totalBetAmount": 20000,
      "totalPayouts": 18000,
      "betPnL": 2000,
      "netDeposits": 20000
    },
    "timeseries": [...], // Daily data for last 7/30/90 days
    "byCurrency": [...], // Distribution by currency
    "byGame": [...] // Distribution by game type
  }
}
```

### Transaction Management

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/transactions` | Yes (`isAdmin`) | Get all transactions with filters |

**Query Parameters**:
- `page`: Page number (default: 1)
- `limit`: Items per page (default: 50)
- `search`: Search by transaction ID, user ID, or address
- `type`: Filter by type (deposit, withdrawal, etc.)
- `currency`: Filter by currency

### User Management

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/users` | Yes (`isAdmin`) | Get all users with balances (paginated) |
| GET | `/users/:id` | Yes (`isAdmin`) | Get user basic info |
| GET | `/users/:id/addresses` | Yes (`isAdmin`) | Get user wallet addresses |
| GET | `/users/:id/games` | Yes (`isAdmin`) | Get user game records |
| GET | `/users/:id/transactions` | Yes (`isAdmin`) | Get user transactions |
| GET | `/users/:id/referral-bonuses` | Yes (`isAdmin`) | Get user referral bonuses |
| POST | `/users/:id/suspend` | Yes (`isAdmin`) | Suspend user account |
| POST | `/users/:id/topup` | Yes (`isAdmin`) | Top up user balance |
| PUT | `/users/:id` | Yes (`isAdmin`) | Update user information |

**Top Up Request Body**:
```json
{
  "currency": "USDT",
  "amount": 100.50,
  "description": "Manual top-up by admin"
}
```

### Game Management

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/provider-games` | No | Get games with filters |
| GET | `/games-categories` | No | Get all game categories |
| GET | `/game-categories` | No | Get all game categories (alternative) |
| POST | `/games/add` | Yes (`isAdmin`) | Add new game |
| POST | `/provider-games/:id/toggle` | No | Toggle game enabled status |
| POST | `/provider-games/:id/category` | No | Update game category |
| PUT | `/provider-games/:id/update` | No | Update game details |
| POST | `/game-categories/add` | No | Create new category |
| PUT | `/game-categories/:id/update` | No | Update category |
| DELETE | `/game-categories/:id/delete` | No | Delete category |

**Game Filters**:
- `categoryId`: Filter by category
- `providerId`: Filter by provider
- `page`: Page number
- `limit`: Items per page
- `enabled`: Filter by enabled status
- `search`: Search by game name

### Product Management

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/products` | Yes (`isAdmin`) | Get all products |
| POST | `/products` | Yes (`isAdmin`) | Create new product |
| PUT | `/products/:id` | Yes (`isAdmin`) | Update product |
| DELETE | `/products/:id` | Yes (`isAdmin`) | Delete product |
| POST | `/products/:code/toggle` | Yes (`isAdmin`) | Toggle product status |

### Payout & Withdrawal Management

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/payouts` | No | Get payouts with filters |
| POST | `/payouts/:id/process` | Yes (`isAdmin`) | Process payout |
| GET | `/withdrawals` | No | Get withdrawal requests |
| POST | `/withdrawals/:id/process` | Yes (`isAdmin`) | Process withdrawal |

**Payout/Withdrawal Filters**:
- `page`: Page number
- `pageSize`: Items per page
- `status`: Filter by status (pending, completed, etc.)
- `currency`: Filter by currency
- `search`: Search by user ID or address

### Game Settings & Configuration

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/game-settings` | Yes (`isAdmin`) | Get game settings |
| POST | `/update-game-settings` | Yes (`isAdmin`) | Update game settings |
| GET | `/hash-games-configs` | Yes (`isAdmin`) | Get hash game configurations |
| POST | `/hash-games-configs/update/:id` | Yes (`isAdmin`) | Update hash game config |

**Game Settings Structure**:
```json
{
  "oddsNumerator": 1,
  "oddsDenominator": 2,
  "feeNumerator": 1,
  "feeDenominator": 100,
  "trxMin": 10,
  "trxMax": 1000,
  "usdtMin": 10,
  "usdtMax": 1000
}
```

### Referral System Management

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/referral-config` | Yes (`isAdmin`) | Get referral configuration |
| POST | `/referral-config` | Yes (`isAdmin`) | Update referral configuration |
| GET | `/referral-bonuses` | Yes (`isAdmin`) | Get all referral bonuses |
| GET | `/referral-stats` | Yes (`isAdmin`) | Get referral statistics |
| POST | `/referral-bonuses/expire` | Yes (`isAdmin`) | Expire old bonuses |

### Logs & Audit

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/logs` | Yes (`isAdmin`) | Get admin action logs |

**Log Query Parameters**:
- `page`: Page number
- `pageSize`: Items per page
- `userId`: Filter by user ID
- `adminId`: Filter by admin ID

---

## 🗄️ Database Schema Overview

### Core Admin-Related Tables

#### 1. **Admins** Table
```sql
CREATE TABLE "Admins" (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL,
  role TEXT NOT NULL,
  status TEXT NOT NULL
);
```

#### 2. **Users** Table
```sql
CREATE TABLE "Users" (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL,
  role TEXT NOT NULL,
  status TEXT NOT NULL,
  telegram TEXT,
  avatar TEXT,
  withdrawal_password TEXT,
  name TEXT,
  phone TEXT,
  email_verified BOOLEAN DEFAULT false,
  "referralCode" TEXT UNIQUE,
  referredById INTEGER,
  FOREIGN KEY (referredById) REFERENCES "Users"(id)
);
```

#### 3. **Balances** Table
```sql
CREATE TABLE "Balances" (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "userId" INTEGER NOT NULL,
  currency TEXT NOT NULL,
  amount NUMERIC(18,4) DEFAULT 0.0000,
  lock NUMERIC(38,18) DEFAULT 0,
  "updatedAt" TIMESTAMPTZ DEFAULT NOW(),
  FOREIGN KEY ("userId") REFERENCES "Users"(id),
  UNIQUE ("userId", currency)
);
```

#### 4. **Games** Table
```sql
CREATE TABLE "Games" (
  id SERIAL PRIMARY KEY,
  "gameCode" TEXT NOT NULL,
  "gameName" TEXT NOT NULL,
  "gameType" TEXT NOT NULL,
  "imageUrl" TEXT NOT NULL,
  "productId" INTEGER NOT NULL,
  "productCode" INTEGER NOT NULL,
  "supportCurrency" TEXT NOT NULL,
  status TEXT NOT NULL,
  "allowFreeRound" BOOLEAN NOT NULL,
  "langName" JSONB NOT NULL,
  "langIcon" JSONB NOT NULL,
  category INTEGER,
  enabled BOOLEAN DEFAULT true,
  provider TEXT,
  "coverImage" JSONB,
  "isHot" BOOLEAN DEFAULT false,
  "isNew" BOOLEAN DEFAULT false,
  "isRecommended" BOOLEAN DEFAULT false,
  "onlinePlayers" INTEGER,
  "launchParams" JSONB,
  visibility JSONB,
  aggregator TEXT,
  "createdAt" TIMESTAMPTZ DEFAULT NOW(),
  "updatedAt" TIMESTAMPTZ DEFAULT NOW()
);
```

#### 5. **GameCategories** Table
```sql
CREATE TABLE "GameCategories" (
  id SERIAL PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  "createdAt" TIMESTAMPTZ DEFAULT NOW(),
  "updatedAt" TIMESTAMPTZ DEFAULT NOW()
);
```

#### 6. **Products** Table
```sql
CREATE TABLE "Products" (
  id SERIAL PRIMARY KEY,
  provider TEXT NOT NULL,
  currency TEXT NOT NULL,
  status TEXT NOT NULL,
  "providerId" INTEGER NOT NULL,
  code INTEGER UNIQUE NOT NULL,
  name TEXT NOT NULL,
  "gameType" TEXT NOT NULL,
  title TEXT NOT NULL,
  enabled BOOLEAN NOT NULL,
  image TEXT,
  "createdAt" TIMESTAMPTZ DEFAULT NOW(),
  "updatedAt" TIMESTAMPTZ DEFAULT NOW()
);
```

#### 7. **Transactions** Table
```sql
CREATE TABLE "Transactions" (
  id SERIAL PRIMARY KEY,
  "userId" INTEGER NOT NULL,
  address TEXT NOT NULL,
  currency TEXT NOT NULL,
  amount NUMERIC(38,18) NOT NULL,
  "txId" TEXT NOT NULL,
  "createdAt" TIMESTAMPTZ DEFAULT NOW(),
  type TEXT NOT NULL,
  FOREIGN KEY ("userId") REFERENCES "Users"(id)
);
```

#### 8. **Logs** Table (Admin Actions)
```sql
CREATE TABLE "Logs" (
  id SERIAL PRIMARY KEY,
  "userId" INTEGER NOT NULL,
  "adminId" INTEGER NOT NULL,
  type TEXT NOT NULL,
  description TEXT NOT NULL,
  "createdAt" TIMESTAMPTZ DEFAULT NOW()
);
```

#### 9. **ReferralBonuses** Table
```sql
CREATE TABLE "ReferralBonuses" (
  id SERIAL PRIMARY KEY,
  "userId" INTEGER NOT NULL,
  "fromUserId" INTEGER NOT NULL,
  amount INTEGER NOT NULL,
  currency TEXT NOT NULL,
  status TEXT DEFAULT 'pending',
  "triggerType" TEXT DEFAULT 'deposit',
  "expiresAt" TIMESTAMPTZ,
  "createdAt" TIMESTAMPTZ DEFAULT NOW(),
  "updatedAt" TIMESTAMPTZ DEFAULT NOW(),
  FOREIGN KEY ("userId") REFERENCES "Users"(id),
  FOREIGN KEY ("fromUserId") REFERENCES "Users"(id)
);
```

#### 10. **ReferralConfigs** Table
```sql
CREATE TABLE "ReferralConfigs" (
  id INTEGER PRIMARY KEY DEFAULT 1,
  "depositBonusPercent" INTEGER DEFAULT 5,
  "betBonusPercent" INTEGER DEFAULT 2,
  "firstDepositBonus" INTEGER DEFAULT 10,
  "firstBetBonus" INTEGER DEFAULT 5,
  "signupBonus" INTEGER DEFAULT 5,
  "maxBonusPerUser" INTEGER DEFAULT 1000,
  "bonusExpiryDays" INTEGER DEFAULT 30,
  enabled BOOLEAN DEFAULT true,
  "updatedAt" TIMESTAMPTZ DEFAULT NOW()
);
```

---

## ⚙️ Admin Operations

### 1. User Management Operations

#### Get Users with Balances
```typescript
// Function: getUsersWithBalances(page, limit)
// Returns: { data: users[], meta: { total, page, limit, totalPages } }
// Includes: user info + all currency balances
```

#### Suspend User
```typescript
// Function: suspendUser(userId)
// Updates: User.status = 'suspended'
```

#### Top Up User Balance
```typescript
// Function: topUpUserBalance(adminId, userId, currency, amount, description)
// Process:
//   1. Upsert balance (increment amount)
//   2. Create log entry
//   3. Returns updated balance
// Uses: Database transaction for atomicity
```

#### Update User Info
```typescript
// Function: updateUserInfo(userId, data)
// Can update: name, email, status, role, phone
```

### 2. Game Management Operations

#### Add Game
```typescript
// Function: addGame(data)
// Process:
//   1. Check if game exists by gameCode
//   2. If exists, return existing game
//   3. If not, create new game
//   4. Delete from TempGames if exists
```

#### Toggle Game Enabled
```typescript
// Function: toggleGameEnabled(gameId)
// Toggles: Game.enabled (true <-> false)
```

#### Update Game Category
```typescript
// Function: updateGameCategory(gameId, category)
// Updates: Game.category
```

### 3. Transaction Operations

#### Get Transactions
```typescript
// Function: getTransactions(limit, page, search, type, currency)
// Filters:
//   - Search: transaction ID, user ID, or address
//   - Type: deposit, withdrawal, etc.
//   - Currency: USDT, TRX, etc.
// Returns: Paginated transaction list
```

### 4. Platform Statistics

#### Get Platform Stats
```typescript
// Function: getPlatformStats(range = 30)
// Range options: 7, 30, or 90 days
// Returns:
//   - Summary: totals for users, deposits, withdrawals, bets
//   - Timeseries: daily breakdown
//   - ByCurrency: distribution by currency
//   - ByGame: distribution by game type
```

### 5. Payout & Withdrawal Processing

#### Process Payout
```typescript
// Function: processPayout(payoutId)
// Process:
//   1. Check payout exists and is pending
//   2. If userId exists: top up user balance
//   3. If no userId: execute blockchain withdrawal
//   4. Update status to "completed"
```

#### Process Withdrawal
```typescript
// Function: processWithdraw(id)
// Supports:
//   - Tron: TRX, USDT
//   - Ethereum: ETH, USDT
//   - Solana: SOL, USDC
// Process:
//   1. Check withdrawal exists and is pending
//   2. Execute blockchain withdrawal
//   3. Update status to "completed"
```

---

## 🔧 Database Functions

### Authentication Functions

#### `login(email, password)`
- Location: `src/db/admin.ts:11`
- Finds admin by email
- Compares password with bcrypt
- Generates JWT token
- Returns: `{ token, user }`

#### `changePassword(userId, password, newPassword)`
- Location: `src/db/admin.ts:34`
- Verifies current password
- Hashes new password
- Updates user password

### User Management Functions

#### `getUserBasicInfo(userId)`
- Returns: User with balances
- Includes: id, email, name, status, role, phone, balances[]

#### `getUserAddresses(userId)`
- Returns: All wallet addresses for user
- Includes: id, publicKey, blockchain

#### `getUserGameRecords(userId, limit, page)`
- Returns: Paginated bet records
- Ordered by: createdAt DESC

#### `getUserTransactions(userId, limit, page)`
- Returns: Paginated transaction list
- Includes: pagination metadata

#### `getUsersWithBalances(page, limit)`
- Returns: All users with their balances
- Paginated response

### Game Management Functions

#### `getAllGames()`
- Returns: All games from database

#### `getGames(options)`
- Options: categoryId, page, limit, enabled, search, providerId
- Returns: Filtered and paginated games

#### `addGame(data)`
- Checks for duplicates by gameCode
- Creates new game if doesn't exist
- Removes from TempGames

#### `toggleGameEnabled(gameId)`
- Toggles enabled status

#### `updateGameCategory(gameId, category)`
- Updates game category

#### `updateGame(id, data)`
- Updates any game fields

### Category Management

#### `getAllCategories()`
- Returns: All game categories

#### `createCategory(name)`
- Creates new category

#### `updateCategory(id, name)`
- Updates category name

#### `deleteCategory(id)`
- Deletes category

### Product Management

#### `getAllProducts()`
- Returns: All products

#### `createProduct(data)`
- Creates new product

#### `updateProduct(id, data)`
- Updates product

#### `deleteProduct(id)`
- Deletes product

#### `toggleProductStatus(productCode, enabled)`
- Toggles product enabled status

### Transaction Functions

#### `getTransactions(limit, page, search, type, currency)`
- Advanced filtering and pagination
- Returns: Paginated transaction list

### Statistics Functions

#### `getPlatformStats(range)`
- Complex aggregation queries
- Daily timeseries data
- Currency and game distributions
- Range: 7, 30, or 90 days

### Configuration Functions

#### `readAllConfigs()`
- Returns: All HashGameConfig records

#### `updateConfigById(id, newConfig)`
- Updates hash game configuration

#### `getGameSettings()`
- Returns: GameSettings (id: 1)

#### `updateGameSettings(data)`
- Updates game settings

### Logging Functions

#### `getLogs({ page, pageSize, userId, adminId })`
- Returns: Paginated admin action logs
- Filters by user or admin

---

## 🔗 Relationships & Data Flow

### Admin → User Operations
```
Admin (via JWT) → User Management
  ├── View users
  ├── Suspend users
  ├── Update user info
  ├── Top up balances
  └── View user details
```

### Admin → Game Operations
```
Admin → Game Management
  ├── View games
  ├── Add games
  ├── Update games
  ├── Toggle enabled status
  ├── Update categories
  └── Manage categories
```

### Admin → Transaction Flow
```
Admin → Transaction Management
  ├── View all transactions
  ├── Filter by type/currency
  ├── Search transactions
  └── Process payouts/withdrawals
```

### Admin → Platform Statistics
```
Admin → Statistics
  ├── User counts
  ├── Deposit/withdrawal totals
  ├── Bet statistics
  ├── Daily timeseries
  └── Currency/game distributions
```

### Database Relationships

```
Admins (standalone)
  └── No foreign keys

Users
  ├── → Balances (1:many)
  ├── → Wallets (1:many)
  ├── → Transactions (1:many)
  ├── → Deposits (1:many)
  ├── → Wagers (1:many)
  ├── → ReferralBonuses (1:many, as user)
  └── → ReferralBonuses (1:many, as fromUser)

Games
  └── → GameCategory (via category field)

Products
  └── → Games (via productCode)

Logs
  ├── → Users (via userId)
  └── → Admins (via adminId)
```

---

## 🔒 Security & Permissions

### Authentication Middleware

#### `isAdmin` Middleware
- Location: `src/utils/jwt.ts:29`
- Checks: JWT token in Authorization header
- Verifies: Token signature and role === "admin"
- Returns: 403 if not admin

### Password Security
- Passwords hashed with bcrypt
- No plain text storage
- JWT tokens for session management

### Protected Routes
- Most admin endpoints require `isAdmin` middleware
- Token must be valid and admin role verified
- Token contains admin ID for logging actions

---

## 📊 Key Database Queries

### Platform Statistics Query
```sql
-- Daily deposits and withdrawals
SELECT
  to_char(date_trunc('day', "createdAt"), 'YYYY-MM-DD') AS day,
  SUM(CASE WHEN type = 'deposit' THEN amount ELSE 0 END) AS deposits,
  SUM(CASE WHEN type = 'withdrawal' THEN amount ELSE 0 END) AS withdrawals
FROM "Transactions"
WHERE "createdAt" >= $1
GROUP BY 1
ORDER BY 1 ASC;
```

### User with Balances Query
```sql
SELECT u.*, b.currency, b.amount
FROM "Users" u
LEFT JOIN "Balances" b ON u.id = b."userId"
ORDER BY u.id DESC
LIMIT $1 OFFSET $2;
```

### Game Filtering Query
```sql
SELECT *
FROM "Games"
WHERE status = 'ACTIVATED'
  AND (category = $1 OR $1 IS NULL)
  AND (enabled = $2 OR $2 IS NULL)
  AND (LOWER("gameName") LIKE LOWER($3) OR $3 IS NULL)
ORDER BY "createdAt" DESC
LIMIT $4 OFFSET $5;
```

---

## 🎯 Best Practices

1. **Always use transactions** for operations that modify multiple tables
2. **Validate input** before database operations
3. **Log admin actions** for audit trail
4. **Use pagination** for large datasets
5. **Check permissions** before sensitive operations
6. **Handle errors gracefully** with proper error messages
7. **Use indexes** on frequently queried fields (email, userId, etc.)

---

## 📝 Notes

- Admin table is separate from User table
- Admin operations are logged in Logs table
- JWT tokens contain full admin object
- Most operations support pagination
- Game operations check for duplicates before insertion
- Balance operations use database transactions for atomicity

