# Harmony WMS

> **Warehouse Management System with Web3 incentives on Stellar/Soroban.**
> End-to-end sales, picking, packing, inventory and invoicing management — with an on-chain reward system for warehouse operators funded in USDC, automatic yield via Blend Protocol, and bank account cash-out via Etherfuse.

<p align="center">
  <img src="https://img.shields.io/badge/Backend-Node.js%2FExpress-339933?logo=node.js&logoColor=white" />
  <img src="https://img.shields.io/badge/Frontend-React%2019-61DAFB?logo=react&logoColor=black" />
  <img src="https://img.shields.io/badge/Database-PostgreSQL%2014+-4169E1?logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Blockchain-Stellar%2FSoroban-7C3AED?logo=stellar&logoColor=white" />
  <img src="https://img.shields.io/badge/DeFi-Blend%20Protocol-06B6D4" />
  <img src="https://img.shields.io/badge/Fiat-Etherfuse-F59E0B" />
  <img src="https://img.shields.io/badge/Deploy-Azure-0078D4?logo=microsoftazure&logoColor=white" />
</p>

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Tech Stack](#tech-stack)
4. [Project Structure](#project-structure)
5. [WMS Module — Warehouse Management](#wms-module--warehouse-management)
6. [Harmony Module — Web3 Incentives](#harmony-module--web3-incentives)
7. [Smart Contracts (Soroban/Rust)](#smart-contracts-sorobanrust)
8. [Blend Protocol — Automatic Yield](#blend-protocol--automatic-yield)
9. [Etherfuse — Bank Account Cash-Out](#etherfuse--bank-account-cash-out)
10. [Database](#database)
11. [REST API](#rest-api)
12. [Roles & Permissions (RBAC)](#roles--permissions-rbac)
13. [Security](#security)
14. [Quick Start Guide](#quick-start-guide)
15. [Environment Variables](#environment-variables)
16. [Azure Deployment](#azure-deployment)
17. [Test Data](#test-data)
18. [On-Chain Verification](#on-chain-verification)

---

## Overview

**Harmony WMS** is an enterprise platform that combines a full-featured Warehouse Management System (WMS) with a blockchain-based incentive module (Harmony). The core idea: warehouse operators earn USDC rewards based on their performance, distributed transparently through smart contracts on Stellar.

### Problem & Solution

| Problem | Harmony Solution |
|---------|-----------------|
| Operators lack motivation due to unclear incentives | Points system + USDC rewards proportional to performance |
| Incentive funds sit idle (not earning) for weeks | Auto-deposit into Blend Protocol generates ~5-7% APY yield |
| Difficulty for operators to access crypto | Etherfuse converts USDC to local currency and deposits into bank account |
| Lack of transparency in bonus distribution | Everything is recorded on-chain on Stellar, publicly verifiable |
| WMS systems disconnected from incentives | Unified platform: same app for warehouse management and rewards |

### Key Features

- **Order Management** with multi-stage workflow (8 states)
- **Optimized Picking** by warehouse route to minimize travel distance
- **Inventory Control** by physical location (shelf/row/level)
- **Automatic Stock Reservation** upon order approval
- **Damage Reports** with photographic evidence
- **Goods Receipt** with location assignment
- **Performance Metrics** per operator and team (KPIs)
- **Invoicing** integrated into the order workflow
- **On-chain Incentives** with Soroban smart contracts
- **Automatic Yield** via Blend Protocol on idle funds
- **Fiat Cash-out** via Etherfuse (USDC to bank account)

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            CLIENT (Browser)                                  │
│                                                                              │
│  React 19 + Vite + TailwindCSS + Zustand + React Router + Freighter Wallet  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  WMS: Dashboard, Orders, Picking, Packing, Inventory, Invoicing       │ │
│  │  Harmony: Activities, Metrics, Workers, Fund, Live, Config            │ │
│  └──────────────────────────────────┬──────────────────────────────────────┘ │
└─────────────────────────────────────┼────────────────────────────────────────┘
                                      │ HTTP / REST API (Axios + JWT)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       BACKEND (Node.js / Express)                            │
│                                                                              │
│  ┌──────────┐  ┌───────────────┐  ┌──────────────────┐  ┌────────────────┐ │
│  │  Routes   │→│  Controllers  │→│     Models        │→│  PostgreSQL    │ │
│  │  /api/*   │  │  (business    │  │  (SQL queries)   │  │  (wms_db)     │ │
│  └──────────┘  │  logic)       │  └──────────────────┘  └────────────────┘ │
│                └───────┬───────┘                                             │
│                        │                                                     │
│  ┌────────────────────┐│┌───────────────────────┐┌──────────────────────┐   │
│  │  Middlewares       │││  Services              ││  Utils               │   │
│  │  auth.js (JWT+RBAC)│││  blend.service.js      ││  picking-routes.js   │   │
│  │  errorHandler.js   │││  stellar.service.js    ││                      │   │
│  │  upload.js (Multer)│││  etherfuse.service.js  ││                      │   │
│  └────────────────────┘│└───────────┬───────────┘└──────────────────────┘   │
└────────────────────────┼────────────┼────────────────────────────────────────┘
                         │            │
          ┌──────────────┘            └──────────────┐
          ▼                                          ▼
┌──────────────────────┐              ┌──────────────────────────────────────┐
│  PostgreSQL (Azure)  │              │       Stellar / Soroban              │
│                      │              │                                      │
│  13 WMS tables       │              │  Factory Contract ──► Org registry   │
│  6 Harmony tables    │              │  Org Contract ──► Tasks, points,     │
│  1 Blend table       │              │                    periods, claims   │
│                      │              │  Blend Pool ──► Yield on USDC        │
│  TZ: America/Bogota  │              │  Etherfuse ──► USDC → MXN (fiat)    │
└──────────────────────┘              └──────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Frontend** | React | 19.1 | UI framework |
| | Vite | 7.1 | Build tool & dev server |
| | TailwindCSS | 3.4 | Utility-first styling |
| | Zustand | 5.0 | Global state management |
| | React Router | 7.9 | SPA routing |
| | React Hook Form | 7.63 | Form handling |
| | Lucide React | 0.544 | Icons |
| | Freighter API | 6.0 | Stellar wallet |
| **Backend** | Node.js | 20+ | Runtime |
| | Express | 4.18 | HTTP framework |
| | pg (node-postgres) | 8.11 | PostgreSQL driver |
| | jsonwebtoken | 9.0 | JWT authentication |
| | bcrypt | 5.1 | Password hashing |
| | Winston | 3.19 | Structured logging |
| | Helmet | - | Security headers |
| | express-rate-limit | - | Brute-force protection |
| | Multer | 2.0 | File uploads |
| **Database** | PostgreSQL | 14+ | Relational database |
| **Blockchain** | Stellar SDK | 14.6 | Stellar transactions |
| | Soroban SDK | 21.7 | Smart contracts (Rust) |
| | Blend SDK | 3.2 | DeFi integration |
| **Fiat** | Etherfuse FX API | - | USDC to local currency conversion |
| **Infra** | Azure | - | Cloud (Static Web Apps + App Service) |

---

## Project Structure

```
harmony-wms/
├── back/                                 # Backend (Node.js / Express)
│   ├── server.js                         # Entry point
│   ├── src/
│   │   ├── app.js                        # Express config (CORS, middlewares, routes)
│   │   ├── config/
│   │   │   ├── db.js                     # PostgreSQL pool + Colombia timezone
│   │   │   ├── dns-fix.js               # DNS fix for Stellar domains
│   │   │   └── logger.js                # Winston logger
│   │   ├── controllers/
│   │   │   ├── auth.controller.js        # Login, registration, JWT
│   │   │   ├── orden.controller.js       # Order lifecycle
│   │   │   ├── harmony.controller.js     # Web3 incentives
│   │   │   ├── inventario.controller.js  # Stock queries
│   │   │   ├── bodega.controller.js      # Warehouse management
│   │   │   ├── producto.controller.js    # Product CRUD
│   │   │   ├── cliente.controller.js     # Customer CRUD
│   │   │   ├── ubicacion.controller.js   # Warehouse locations
│   │   │   ├── recepcion.controller.js   # Goods receipt
│   │   │   ├── averia.controller.js      # Damage reports
│   │   │   ├── desempeno.controller.js   # KPIs & metrics
│   │   │   ├── proveedor.controller.js   # Supplier CRUD
│   │   │   ├── transferencia.controller.js # Inter-warehouse transfers
│   │   │   └── upload.controller.js      # File uploads
│   │   ├── models/                       # Data access layer (static methods + SQL)
│   │   │   ├── auth.model.js
│   │   │   ├── orden.model.js            # getOptimizedPickingList()
│   │   │   ├── harmony.model.js          # Wallets, periods, tasks, points
│   │   │   ├── desempeno.model.js        # KPI calculations
│   │   │   └── ...                       # (one per domain)
│   │   ├── services/
│   │   │   ├── blend.service.js          # Blend Protocol (supply/withdraw USDC)
│   │   │   ├── stellar.service.js        # Soroban pipeline (build→sign→submit)
│   │   │   └── etherfuse.service.js      # USDC → fiat via Etherfuse FX
│   │   ├── routes/                       # REST endpoint definitions
│   │   ├── middlewares/
│   │   │   ├── auth.js                   # JWT + RBAC (verifyToken, checkRole)
│   │   │   ├── errorHandler.js           # Centralized error handling
│   │   │   └── upload.js                 # Multer config
│   │   └── utils/
│   │       └── picking-routes.js         # Route optimization algorithm
│   ├── database/
│   │   ├── schema.sql                    # Master schema
│   │   ├── seed.sql                      # Test data
│   │   ├── seed_ordenes_demo.sql         # Demo orders for activities
│   │   ├── seed_ordenes_flujo.sql        # Orders for workflow practice
│   │   └── migrations/                   # Incremental migrations
│   ├── scripts/                          # Utilities (setup, generators, harmony)
│   ├── tests/                            # Tests
│   └── uploads/                          # User-uploaded files
│
├── front/                                # Frontend (React / Vite)
│   ├── src/
│   │   ├── App.jsx                       # Router + role-protected routes
│   │   ├── main.jsx                      # React entry point
│   │   ├── store/
│   │   │   └── authStore.js              # Zustand (user, token, isAuthenticated)
│   │   ├── services/
│   │   │   └── api.js                    # Axios + JWT interceptors
│   │   ├── pages/
│   │   │   ├── auth/                     # Login, Register
│   │   │   ├── dashboard/                # Main dashboard
│   │   │   ├── ordenes/                  # Orders, Order Approval
│   │   │   ├── actividades/              # Picking, Packing
│   │   │   ├── almacenes/                # Warehouses, inventory, transfers
│   │   │   ├── productos/                # Product catalog
│   │   │   ├── ubicaciones/              # Physical warehouse map
│   │   │   ├── recepciones/              # Goods receipt
│   │   │   ├── clientes/                 # Customer management
│   │   │   ├── proveedores/              # Supplier management
│   │   │   ├── averias/                  # Damage reports + evidence
│   │   │   ├── facturacion/              # Invoicing + history
│   │   │   ├── desempeno/                # KPIs, rankings, activities
│   │   │   └── harmony/                  # Web3 module
│   │   │       ├── worker/               # MyOrders, MyPerformance
│   │   │       ├── leader/               # Activities, Metrics, Fund, Live
│   │   │       └── admin/                # HarmonyConfig
│   │   └── components/
│   │       ├── layout/Layout.jsx         # Sidebar + header
│   │       └── common/                   # Reusable components
│   └── public/
│       ├── staticwebapp.config.json      # Azure Static Web Apps routing
│       └── web.config                    # Azure App Service (IIS) routing
│
├── contracts/                            # Smart Contracts (Soroban / Rust)
│   ├── Cargo.toml                        # Workspace config
│   ├── factory/src/lib.rs                # Factory: organization registry
│   ├── organization/src/lib.rs           # Org: members, tasks, periods, claims
│   └── DEPLOY.md                         # Stellar deployment guide
│
├── CLAUDE.md                             # Claude Code instructions
├── BLEND_YIELD_DEMO.md                   # Verifiable Blend yield demo
├── TESTING.md                            # Testing documentation
└── README.md                             # Main README (Spanish)
```

---

## WMS Module — Warehouse Management

### Sales Order Workflow

The WMS core is an 8-state workflow with role-controlled transitions:

```
  [SALES REP]                       [WAREHOUSE MANAGER]
      │                                  │
      │  Creates order                   │
      ▼                                  │
 ┌──────────────────┐   Approves        │
 │   Pending        │─────────────►  ┌──────────────┐
 │   Approval       │                │   Approved   │
 └──────────────────┘   Rejects      └──────┬───────┘
          │                │                │
          ▼                │         Assigns operator
 ┌──────────────────┐◄────┘                │
 │   Rejected       │                      ▼
 └──────────────────┘
                           [OPERATOR]    ┌──────────────────┐
                                         │  Picking         │ ← Optimized route
                                         └────────┬─────────┘
                                                  │
                                           Finishes picking
                                                  │
                                                  ▼
                                         ┌──────────────────┐
                                         │   Packing        │ ← Packing + boxes
                                         └────────┬─────────┘
                                                  │
                           [INVOICING]     Finishes packing
                                │                 │
                                │                 ▼
                                │        ┌──────────────────┐
                                │◄───────│ Ready to Invoice │
                                │        └──────────────────┘
                                │
                           Generates invoice
                                │
                                ▼
                         ┌─────────────┐
                         │  Invoiced   │ ← Final state
                         └─────────────┘
```

### Optimized Picking

The `getOptimizedPickingList(orden_id)` function sorts products by warehouse route to minimize operator travel:

- Sort order: `route_order` → `shelf` → `row` → `level`
- Support for multiple locations per product (primary vs. secondary)
- Statistics: total items, locations to visit, products without location

### Other WMS Modules

| Module | Description |
|--------|-------------|
| **Inventory** | Stock by physical location, automatic reservations on order approval |
| **Locations** | Warehouse map (shelf/row/level), product assignment |
| **Goods Receipt** | Incoming merchandise with location assignment |
| **Damage Reports** | Damage reports with types (Damage, Missing, Breakage, Expiry) + photos |
| **Warehouses** | Multi-warehouse with inter-warehouse transfers |
| **Suppliers** | Supplier CRUD |
| **Customers** | Customer database (tax ID, business name, city) |
| **Invoicing** | Cycle closure: sequential numbering, history |
| **Performance** | KPIs per operator: orders processed, average times, rankings |

---

## Harmony Module — Web3 Incentives

Harmony is the incentive module that connects WMS operational performance with on-chain rewards on Stellar.

### How It Works

```
1. BUSINESS OWNER creates a reward period
   └─► Deposits USDC as fund (e.g., $500 USDC for the month)
       └─► Auto-deposit into Blend Protocol (generates yield while idle)

2. SUPERVISOR assigns activities/tasks to WORKERS
   └─► Each activity has a template with base points

3. WORKERS complete tasks with evidence
   └─► The system automatically records activity from the WMS

4. SUPERVISOR reviews and approves/rejects tasks
   └─► Can apply a point multiplier (0% - 200%)

5. Period is closed
   └─► Auto-withdraw from Blend (original USDC + yield)
   └─► Total points for each worker are calculated

6. Fund is distributed proportionally
   └─► Worker with 30% of points → receives 30% of the fund
   └─► Direct USDC payment to their Stellar wallet
   └─► Or automatic conversion to local currency via Etherfuse

7. Blend yield stays with the business owner as passive income
```

### Views by Role

| Role | Available Pages | Functions |
|------|----------------|-----------|
| **Worker** | My Orders, My Performance | View assigned tasks, progress, accumulated points |
| **Leader** | Activities, Metrics, Workers, Fund, Live | Manage tasks, review evidence, handle periods |
| **Admin** | HarmonyConfig | Configure contracts, tokens, Blend, Etherfuse |

---

## Smart Contracts (Soroban/Rust)

Two contracts deployed on Stellar (testnet), written in Rust with Soroban SDK 21.7:

### Factory Contract (`contracts/factory/src/lib.rs`)

Ecosystem entry point. Centralized organization registry.

| Function | Access | Description |
|----------|--------|-------------|
| `initialize(admin)` | Once | Initial setup |
| `register_org(org_address)` | Admin | Register new organization |
| `get_org(org_id)` | Public | Query organization by ID |
| `get_org_count()` | Public | Total organizations |
| `transfer_admin(new_admin)` | Admin + new | Transfer admin (requires both signatures) |

### Organization Contract (`contracts/organization/src/lib.rs`)

Main Harmony contract. Manages members, activities, periods, and reward distribution.

**On-chain roles:** `Owner` | `Supervisor` | `Worker`

**Period states (state machine):**
```
Open → Closed → Distributed
```

**Task states:**
```
Assigned → Completed → Approved / Rejected / Skipped
```

| Function | Access | Description |
|----------|--------|-------------|
| `add_member(address, role)` | Owner | Add member to organization |
| `create_activity_template(name, points)` | Owner/Supervisor | Create activity type |
| `assign_task(worker, template_id, period_id)` | Supervisor | Assign task |
| `complete_task(task_id, evidence_url)` | Worker (own) | Mark task as completed |
| `review_task(task_id, multiplier, state)` | Supervisor | Approve/reject with multiplier |
| `open_period(start, end, asset, fund)` | Owner | Create reward period |
| `close_period(period_id)` | Owner | Close period |
| `distribute_period(period_id)` | Owner | Calculate and assign rewards |
| `claim_reward(period_id)` | Worker | Claim their share on-chain |
| `sweep_expired_claims(period_id)` | Owner | Recover unclaimed rewards (>1 year) |
| `pause() / unpause()` | Owner | Emergency control |

**Security guarantees:**
- `require_auth()` on all state-mutating operations
- Initialization guard (impossible to reinitialize)
- Strict state machine — forward-only transitions
- CEI pattern (Checks-Effects-Interactions) in claims
- Overflow checks in arithmetic
- Multiplier bounded [0%, 200%] (basis points)
- Double-claim prevention (mark before transfer)
- TTL extended to 365 days

---

## Blend Protocol — Automatic Yield

When a business owner deposits USDC as an incentive fund, that money normally sits idle (not earning) during the entire period. With the Blend Protocol integration, USDC is automatically deposited into a DeFi lending pool on Stellar, generating yield.

### Flow

```
BUSINESS OWNER                     HARMONY (Backend)                   BLEND PROTOCOL (Soroban)
    |                                     |                                     |
    |--- Deposits USDC (fund) ----------> |                                     |
    |                                     |--- Auto-deposit USDC -------------> |
    |                                     |    (SupplyCollateral)                |
    |                                     |<-- Receives bTokens ---------------|
    |                                     |                                     |
    |   [ USDC generates yield in Blend while period is open ]                 |
    |                                     |                                     |
    |                                     |--- Leader closes period             |
    |                                     |--- Auto-withdraw USDC + yield ---> |
    |                                     |    (WithdrawCollateral)              |
    |                                     |<-- Original USDC + yield ----------|
    |                                     |                                     |
    |                                     |--- Distribute to workers            |
    |<-- Yield stays in admin wallet -----|                                     |
    |    (business owner profit)          |                                     |
```

### E2E Test Verified on Testnet

| Step | Operation | Amount | TX Hash |
|------|-----------|--------|---------|
| 1 | USDC Acquisition (DEX) | 10 USDC | `e742be77...087a` |
| 2 | Supply to Blend | 3 USDC | `e095cc90...369d` |
| 3 | Withdraw + yield | **3.0909742 USDC** | `9e21f142...ce67` |
| | **Yield earned** | **+0.0909742 USDC (~3.03%)** | |

### Resilience

| Scenario | Behavior |
|----------|----------|
| Blend disabled | USDC stays in admin wallet, normal distribution |
| Deposit fails | Warning logged, fund confirmed without Blend |
| Withdraw fails | Position marked `Failed`, distribution uses admin balance |
| Blend SDK unavailable | Lazy-loading with try/catch, system operates without Blend |

### Yield Projection (Mainnet, ~5-7% APY)

| Fund | Period | Estimated Yield |
|------|--------|-----------------|
| $100 USDC | 15 days | +$0.21 - $0.29 |
| $500 USDC | 30 days | +$2.05 - $2.88 |
| $1,000 USDC | 30 days | +$4.11 - $5.75 |
| $5,000 USDC | 30 days | +$20.55 - $28.77 |
| $10,000 USDC | 90 days | +$123.29 - $172.60 |

---

## Etherfuse — Bank Account Cash-Out

Etherfuse FX enables operators to receive their rewards in local currency directly into their bank account, without needing to handle crypto.

### Flow

```
Worker earns reward
    └─► Backend calculates amount in USDC
        └─► Calls Etherfuse FX API
            └─► Etherfuse converts USDC → MXN (or local currency)
                └─► Direct deposit to worker's bank account
```

### Worker Banking Details

Each user can register their banking details in their profile:
- Bank name
- Account number / CLABE
- Preferred currency

The backend stores this data and uses it during distribution to execute the payment via Etherfuse automatically.

---

## Database

### WMS Tables (13 tables)

| Table | Description |
|-------|-------------|
| `usuarios` | Users with roles (Sales Rep, Warehouse Manager, Operator, Invoicing, Admin) |
| `clientes` | Customer database (tax ID, business name, contact, city) |
| `productos` | Product catalog (code, prices, stock) |
| `ordenes_venta` | Orders with 8-state workflow |
| `orden_detalles` | Order line items (quantity ordered/picked/packed) |
| `ubicaciones` | Physical warehouse positions (shelf/row/level/route_order) |
| `inventario_ubicaciones` | Stock by location + reservations |
| `bodegas` | Multi-warehouse |
| `inventario_bodegas` | Consolidated stock per warehouse |
| `recepciones` / `recepcion_detalles` | Goods receipt |
| `averias` | Damage reports with evidence |
| `proveedores` | Suppliers |
| `desempeno_actividades` | Activity log for KPIs |

### Harmony Tables (6 tables)

| Table | Description |
|-------|-------------|
| `harmony_config` | Global configuration (contracts, tokens, Blend, Etherfuse) |
| `harmony_periodos` | Reward periods with state and fund |
| `harmony_plantillas_actividad` | Activity templates with base points |
| `harmony_tareas` | Individual task assignments |
| `harmony_puntos_periodo` | Points earned and adjusted amounts per worker/period |
| `harmony_historial_cambios` | Audit log |

### Blend Table (1 table)

| Table | Description |
|-------|-------------|
| `harmony_blend_positions` | Blend positions: deposits, withdrawals, yield, tx hashes |

### Configuration

- **Timezone:** `America/Bogota` (UTC-5) automatically set on every connection
- **SSL:** Enabled for Azure PostgreSQL compatibility
- **Pool:** Max 10 connections, idle timeout 20s
- **Transactions:** ACID support via `getClient()` with BEGIN/COMMIT/ROLLBACK

---

## REST API

### Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/login` | Login (email + password) → JWT token |
| POST | `/api/auth/register` | Register new user |
| GET | `/api/auth/profile` | Authenticated user profile |
| POST | `/api/auth/refresh` | Refresh JWT token |

### Sales Orders

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/ordenes` | Create order |
| GET | `/api/ordenes` | List orders (filter by state, sales rep, date) |
| GET | `/api/ordenes/:id` | Order details |
| PUT | `/api/ordenes/:id` | Update order |
| POST | `/api/ordenes/:id/aprobar` | Approve order |
| POST | `/api/ordenes/:id/rechazar` | Reject order |
| POST | `/api/ordenes/:id/alistamiento` | Start/finish picking |
| POST | `/api/ordenes/:id/empaque` | Start/finish packing |
| POST | `/api/ordenes/:id/facturar` | Invoice order |

### Inventory & Locations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/inventario/ubicaciones` | Inventory by location |
| GET | `/api/inventario/bodega/:id` | Warehouse inventory |
| GET | `/api/ubicaciones` | List locations |
| POST | `/api/ubicaciones` | Create location |

### Harmony (Web3)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/harmony/config` | Get Harmony configuration |
| POST | `/api/harmony/config` | Update configuration |
| GET | `/api/harmony/usuarios-wallets` | Users with Stellar wallets |
| POST | `/api/harmony/periodo` | Create reward period |
| POST | `/api/harmony/periodo/:id/confirmar-fondo` | Confirm fund (+ auto-deposit to Blend) |
| POST | `/api/harmony/periodo/:id/cerrar` | Close period |
| POST | `/api/harmony/distribuir` | Distribute rewards (+ auto-withdraw from Blend) |

### Other Modules

Similar CRUD endpoints for: products, customers, suppliers, goods receipt, damage reports, warehouses, transfers, performance, uploads.

**Total: 70+ REST endpoints.**

---

## Roles & Permissions (RBAC)

```
┌───────────────────┬───────────┬───────────┬──────────┬──────────┬───────────────┐
│     Action        │ Sales Rep │ Warehouse │ Operator │Invoicing │     Admin     │
│                   │           │  Manager  │          │          │               │
├───────────────────┼───────────┼───────────┼──────────┼──────────┼───────────────┤
│ Create orders     │     ✓     │     ✓     │    ✗     │    ✗     │      ✓        │
│ Approve/Reject    │     ✗     │     ✓     │    ✗     │    ✗     │      ✓        │
│ Picking           │     ✗     │     ✓     │    ✓     │    ✗     │      ✓        │
│ Packing           │     ✗     │     ✓     │    ✓     │    ✗     │      ✓        │
│ Invoice           │     ✗     │     ✓     │    ✗     │    ✓     │      ✓        │
│ Inventory (view)  │     ✓     │     ✓     │    ✓     │    ✓     │      ✓        │
│ Inventory (edit)  │     ✗     │     ✓     │    ✗     │    ✗     │      ✓        │
│ Customers         │     ✓     │     ✓     │    ✗     │    ✓     │      ✓        │
│ Performance (own) │     ✗     │     ✗     │    ✓     │    ✗     │      ✓        │
│ Performance (all) │     ✗     │     ✓     │    ✗     │    ✗     │      ✓        │
│ Damage reports    │     ✓     │     ✓     │    ✓     │    ✓     │      ✓        │
│ Harmony config    │     ✗     │     ✗     │    ✗     │    ✗     │      ✓        │
│ Harmony leader    │     ✗     │     ✓     │    ✗     │    ✗     │      ✓        │
│ Harmony worker    │     ✗     │     ✗     │    ✓     │    ✗     │      ✓        │
└───────────────────┴───────────┴───────────┴──────────┴──────────┴───────────────┘
```

---

## Security

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | JWT (24h) + refresh tokens (7d) |
| **Passwords** | bcrypt with salt |
| **Authorization** | RBAC with `checkRole()` middleware |
| **HTTP** | Helmet (security headers) |
| **Rate Limiting** | 500 req/15min global, 20 req/15min on auth |
| **SQL Injection** | Parameterized queries (pg driver) |
| **CORS** | Configurable whitelist |
| **File Uploads** | Multer with type filter and max size |
| **Smart Contracts** | `require_auth()`, state machine, CEI pattern, overflow checks |
| **Sensitive Data** | `.env` not versioned, SSL for PostgreSQL on Azure |

---

## Quick Start Guide

### Prerequisites

- Node.js 20+
- PostgreSQL 14+
- Git

### 1. Clone the repository

```bash
git clone https://github.com/MariaDelina/PruebaUSDC.git
cd PruebaUSDC
```

### 2. Set up the database

```bash
psql -U postgres -c "CREATE DATABASE wms_db;"
psql -U postgres -d wms_db -f back/database/schema.sql
psql -U postgres -d wms_db -f back/database/seed.sql                # Initial data
psql -U postgres -d wms_db -f back/database/seed_ordenes_flujo.sql  # Test orders
```

### 3. Backend

```bash
cd back
npm install
cp .env.example .env   # Edit with your credentials
npm run dev             # http://localhost:3000
```

### 4. Frontend

```bash
cd front
npm install
npm run dev             # http://localhost:5173
```

### 5. Test Credentials

| Role | Email | Password |
|------|-------|----------|
| Admin | admin@wms.com | admin123 |
| Sales Rep | vendedor@wms.com | vendedor123 |
| Warehouse Manager | jefe@wms.com | jefe123 |
| Operator | operario1@wms.com | operario123 |
| Invoicing | facturacion@wms.com | facturacion123 |

---

## Environment Variables

### Backend (`back/.env`)

```env
# Server
PORT=3000
NODE_ENV=development
LOG_LEVEL=debug

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=your_password
DB_NAME=wms_db

# JWT
JWT_SECRET=secret_key_minimum_32_characters
JWT_EXPIRES_IN=24h
JWT_REFRESH_EXPIRES_IN=7d

# CORS
CORS_ORIGIN=http://localhost:5173

# Inventory
ENABLE_INVENTORY_RESERVATION=true

# Stellar / Harmony
STELLAR_STUB=true                     # true = simulate without real network
STELLAR_NETWORK=testnet               # testnet | mainnet
STELLAR_ADMIN_SECRET=S...             # Admin private key
FACTORY_CONTRACT_ID=C...              # Factory contract on Stellar
ORG_CONTRACT_ID=C...                  # Organization contract
```

### Frontend (`front/.env`)

```env
VITE_API_URL=http://localhost:3000/api
```

---

## Azure Deployment

### Frontend → Azure Static Web Apps

```bash
cd front
npm run build
# Deploy dist/ to Azure Static Web Apps
# The staticwebapp.config.json file handles SPA routing
```

### Backend → Azure App Service (Linux)

```bash
cd back
# CI/CD configured in azure-pipelines.yml
# Trigger: push to main/master
# Automatic deploy to Azure Web App
```

### Database → Azure Database for PostgreSQL

- SSL required (already configured in `db.js`)
- Timezone is automatically set on every connection

---

## Test Data

The project includes several seeds to populate the database with test data:

| File | Contents |
|------|----------|
| `seed.sql` | 6 users, 5 customers, 15 products, 10 locations, inventory, 6 orders |
| `seed_ordenes_demo.sql` | 3 orders in Approved state (3, 5, and 6 products) |
| `seed_ordenes_flujo.sql` | 3 orders for practicing the complete workflow |

### Workflow Orders (for practice)

| Order | State | Products | Practice |
|-------|-------|----------|----------|
| ORD-FLUJO-001 | Pending Approval | 2 | Approve or reject |
| ORD-FLUJO-002 | Approved | 3 | Start picking |
| ORD-FLUJO-003 | Pending Approval | 3 (with 10% discount) | Full workflow: approve → pick → pack → invoice |

---

## On-Chain Verification

All Harmony transactions are verifiable on Stellar Testnet:

| Resource | Link |
|----------|------|
| Admin Wallet | [stellar.expert/account/GBEP4X...](https://stellar.expert/explorer/testnet/account/GBEP4XMMRPFAI7NNTOGAMOBF6ELKD5WYMRLONPFZV6Z2TNWIQLOKMRQ7) |
| Blend Pool | [stellar.expert/contract/CCEBVD...](https://stellar.expert/explorer/testnet/contract/CCEBVDYM32YNYCVNRXQKDFFPISJJCV557CDZEIRBEE4NCV4KHPQ44HGF) |
| Supply TX | [stellar.expert/tx/e095cc90...](https://stellar.expert/explorer/testnet/tx/e095cc90526a6f18f9e77ea69e4c601ba986743ae3ab85afb8beea6eed81369d) |
| Withdraw TX | [stellar.expert/tx/9e21f142...](https://stellar.expert/explorer/testnet/tx/9e21f14213c3e58e3ce1a4cb9ead6384495bf30a68ec755f7bebd2cfcb79ce67) |

---

## License

Academic / demonstrative project. Built for warehouse management with Web3 incentives on Stellar.

---

<p align="center">
  <b>Harmony WMS</b> — Warehouse management + transparent on-chain incentives
</p>
