# Harmony WMS

> **Warehouse Management System con incentivos Web3 sobre Stellar/Soroban.**
> Gestión integral de ventas, alistamiento, empaque, inventario y facturación — con un sistema de recompensas on-chain para operarios financiado en USDC, yield automático vía Blend Protocol y cash-out a cuenta bancaria vía Etherfuse.

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

## Tabla de Contenidos

1. [Descripcion General](#descripcion-general)
2. [Arquitectura del Sistema](#arquitectura-del-sistema)
3. [Stack Tecnologico](#stack-tecnologico)
4. [Estructura del Proyecto](#estructura-del-proyecto)
5. [Modulo WMS — Gestion de Almacen](#modulo-wms--gestion-de-almacen)
6. [Modulo Harmony — Incentivos Web3](#modulo-harmony--incentivos-web3)
7. [Smart Contracts (Soroban/Rust)](#smart-contracts-sorobanrust)
8. [Blend Protocol — Yield Automatico](#blend-protocol--yield-automatico)
9. [Etherfuse — Cash-Out a Cuenta Bancaria](#etherfuse--cash-out-a-cuenta-bancaria)
10. [Base de Datos](#base-de-datos)
11. [API REST](#api-rest)
12. [Roles y Permisos (RBAC)](#roles-y-permisos-rbac)
13. [Seguridad](#seguridad)
14. [Guia de Inicio Rapido](#guia-de-inicio-rapido)
15. [Variables de Entorno](#variables-de-entorno)
16. [Despliegue en Azure](#despliegue-en-azure)
17. [Datos de Prueba](#datos-de-prueba)
18. [Verificacion On-Chain](#verificacion-on-chain)

---

## Descripcion General

**Harmony WMS** es una plataforma empresarial que combina un sistema de gestion de almacen completo (WMS) con un modulo de incentivos basado en blockchain (Harmony). La idea central: los operarios de bodega ganan recompensas en USDC por su desempeno, distribuidas de forma transparente a traves de smart contracts en Stellar.

### Que problema resuelve

| Problema | Solucion Harmony |
|----------|-----------------|
| Operarios sin motivacion por falta de incentivos claros | Sistema de puntos + recompensas USDC proporcionales al desempeno |
| Fondos de incentivos idle (sin producir) durante semanas | Auto-deposit en Blend Protocol genera yield ~5-7% APY |
| Dificultad para que operarios accedan a cripto | Etherfuse convierte USDC a moneda local y deposita en cuenta bancaria |
| Falta de transparencia en distribucion de bonos | Todo queda registrado on-chain en Stellar, verificable publicamente |
| Sistemas WMS desconectados de incentivos | Plataforma unificada: misma app para gestion de bodega y recompensas |

### Caracteristicas Principales

- **Gestion de Ordenes** con flujo multi-etapa (8 estados)
- **Picking optimizado** por ruta de bodega para minimizar desplazamientos
- **Control de inventario** por ubicacion fisica (estanteria/fila/nivel)
- **Reserva de stock** automatica al aprobar ordenes
- **Reportes de averias** con evidencia fotografica
- **Recepciones de mercancia** con asignacion a ubicaciones
- **Metricas de desempeno** por operario y equipo (KPIs)
- **Facturacion** integrada al flujo de pedidos
- **Incentivos on-chain** con smart contracts Soroban
- **Yield automatico** via Blend Protocol sobre fondos idle
- **Cash-out fiat** via Etherfuse (USDC a cuenta bancaria)

---

## Arquitectura del Sistema

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           CLIENTE (Browser)                                  │
│                                                                              │
│  React 19 + Vite + TailwindCSS + Zustand + React Router + Freighter Wallet  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  WMS: Dashboard, Ordenes, Picking, Empaque, Inventario, Facturacion    │ │
│  │  Harmony: Actividades, Metricas, Trabajadores, Fondo, Live, Config    │ │
│  └──────────────────────────────────┬──────────────────────────────────────┘ │
└─────────────────────────────────────┼────────────────────────────────────────┘
                                      │ HTTP / REST API (Axios + JWT)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      BACKEND (Node.js / Express)                             │
│                                                                              │
│  ┌──────────┐  ┌───────────────┐  ┌──────────────────┐  ┌────────────────┐ │
│  │  Routes   │→│  Controllers  │→│     Models        │→│  PostgreSQL    │ │
│  │  /api/*   │  │  (logica de   │  │  (queries SQL)   │  │  (wms_db)     │ │
│  └──────────┘  │  negocio)     │  └──────────────────┘  └────────────────┘ │
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
│  13 tablas WMS       │              │  Factory Contract ──► Registro orgs  │
│  6 tablas Harmony    │              │  Org Contract ──► Tareas, puntos,    │
│  1 tabla Blend       │              │                    periodos, claims  │
│                      │              │  Blend Pool ──► Yield sobre USDC     │
│  TZ: America/Bogota  │              │  Etherfuse ──► USDC → MXN (fiat)    │
└──────────────────────┘              └──────────────────────────────────────┘
```

---

## Stack Tecnologico

| Capa | Tecnologia | Version | Proposito |
|------|-----------|---------|-----------|
| **Frontend** | React | 19.1 | Framework UI |
| | Vite | 7.1 | Build tool y dev server |
| | TailwindCSS | 3.4 | Estilos utilitarios |
| | Zustand | 5.0 | Estado global |
| | React Router | 7.9 | Enrutamiento SPA |
| | React Hook Form | 7.63 | Formularios |
| | Lucide React | 0.544 | Iconografia |
| | Freighter API | 6.0 | Wallet Stellar |
| **Backend** | Node.js | 20+ | Runtime |
| | Express | 4.18 | Framework HTTP |
| | pg (node-postgres) | 8.11 | Driver PostgreSQL |
| | jsonwebtoken | 9.0 | Autenticacion JWT |
| | bcrypt | 5.1 | Hashing de passwords |
| | Winston | 3.19 | Logging estructurado |
| | Helmet | - | Headers de seguridad |
| | express-rate-limit | - | Proteccion contra brute-force |
| | Multer | 2.0 | Subida de archivos |
| **Database** | PostgreSQL | 14+ | Base de datos relacional |
| **Blockchain** | Stellar SDK | 14.6 | Transacciones Stellar |
| | Soroban SDK | 21.7 | Smart contracts (Rust) |
| | Blend SDK | 3.2 | Integracion DeFi |
| **Fiat** | Etherfuse FX API | - | Conversion USDC a moneda local |
| **Infra** | Azure | - | Cloud (Static Web Apps + App Service) |

---

## Estructura del Proyecto

```
harmony-wms/
├── back/                                 # Backend (Node.js / Express)
│   ├── server.js                         # Entry point
│   ├── src/
│   │   ├── app.js                        # Config Express (CORS, middlewares, rutas)
│   │   ├── config/
│   │   │   ├── db.js                     # Pool PostgreSQL + timezone Colombia
│   │   │   ├── dns-fix.js               # Fix DNS para dominios Stellar
│   │   │   └── logger.js                # Winston logger
│   │   ├── controllers/
│   │   │   ├── auth.controller.js        # Login, registro, JWT
│   │   │   ├── orden.controller.js       # Ciclo de vida de ordenes
│   │   │   ├── harmony.controller.js     # Incentivos Web3
│   │   │   ├── inventario.controller.js  # Consultas de stock
│   │   │   ├── bodega.controller.js      # Gestion de bodegas
│   │   │   ├── producto.controller.js    # CRUD productos
│   │   │   ├── cliente.controller.js     # CRUD clientes
│   │   │   ├── ubicacion.controller.js   # Ubicaciones de bodega
│   │   │   ├── recepcion.controller.js   # Recepciones de mercancia
│   │   │   ├── averia.controller.js      # Reportes de danos
│   │   │   ├── desempeno.controller.js   # KPIs y metricas
│   │   │   ├── proveedor.controller.js   # CRUD proveedores
│   │   │   ├── transferencia.controller.js # Transferencias inter-bodega
│   │   │   └── upload.controller.js      # Subida de archivos
│   │   ├── models/                       # Data access layer (static methods + SQL)
│   │   │   ├── auth.model.js
│   │   │   ├── orden.model.js            # getOptimizedPickingList()
│   │   │   ├── harmony.model.js          # Wallets, periodos, tareas, puntos
│   │   │   ├── desempeno.model.js        # Calculos KPI
│   │   │   └── ...                       # (uno por dominio)
│   │   ├── services/
│   │   │   ├── blend.service.js          # Blend Protocol (supply/withdraw USDC)
│   │   │   ├── stellar.service.js        # Pipeline Soroban (build→sign→submit)
│   │   │   └── etherfuse.service.js      # USDC → fiat via Etherfuse FX
│   │   ├── routes/                       # Definicion de endpoints REST
│   │   ├── middlewares/
│   │   │   ├── auth.js                   # JWT + RBAC (verifyToken, checkRole)
│   │   │   ├── errorHandler.js           # Manejo centralizado de errores
│   │   │   └── upload.js                 # Multer config
│   │   └── utils/
│   │       └── picking-routes.js         # Algoritmo de optimizacion de ruta
│   ├── database/
│   │   ├── schema.sql                    # Esquema maestro
│   │   ├── seed.sql                      # Datos de prueba
│   │   ├── seed_ordenes_demo.sql         # Ordenes demo para actividades
│   │   ├── seed_ordenes_flujo.sql        # Ordenes para practicar el flujo
│   │   └── migrations/                   # Migraciones incrementales
│   ├── scripts/                          # Utilidades (setup, generadores, harmony)
│   ├── tests/                            # Tests
│   └── uploads/                          # Archivos subidos por usuarios
│
├── front/                                # Frontend (React / Vite)
│   ├── src/
│   │   ├── App.jsx                       # Router + rutas protegidas por rol
│   │   ├── main.jsx                      # Entry point React
│   │   ├── store/
│   │   │   └── authStore.js              # Zustand (user, token, isAuthenticated)
│   │   ├── services/
│   │   │   └── api.js                    # Axios + interceptores JWT
│   │   ├── pages/
│   │   │   ├── auth/                     # Login, Register
│   │   │   ├── dashboard/                # Dashboard principal
│   │   │   ├── ordenes/                  # Ordenes, AprobarOrdenes
│   │   │   ├── actividades/              # Alistamiento (picking), Empaque (packing)
│   │   │   ├── almacenes/                # Bodegas, inventario, transferencias
│   │   │   ├── productos/                # Catalogo de productos
│   │   │   ├── ubicaciones/              # Mapa fisico de bodega
│   │   │   ├── recepciones/              # Entrada de mercancia
│   │   │   ├── clientes/                 # Gestion de clientes
│   │   │   ├── proveedores/              # Gestion de proveedores
│   │   │   ├── averias/                  # Reportes de danos + evidencia
│   │   │   ├── facturacion/              # Facturacion + historico
│   │   │   ├── desempeno/                # KPIs, rankings, actividades
│   │   │   └── harmony/                  # Modulo Web3
│   │   │       ├── worker/               # MisOrdenes, MiRendimiento
│   │   │       ├── leader/               # Actividades, Metricas, Fondo, Live
│   │   │       └── admin/                # HarmonyConfig
│   │   └── components/
│   │       ├── layout/Layout.jsx         # Sidebar + header
│   │       └── common/                   # Componentes reutilizables
│   └── public/
│       ├── staticwebapp.config.json      # Azure Static Web Apps routing
│       └── web.config                    # Azure App Service (IIS) routing
│
├── contracts/                            # Smart Contracts (Soroban / Rust)
│   ├── Cargo.toml                        # Workspace config
│   ├── factory/src/lib.rs                # Factory: registro de organizaciones
│   ├── organization/src/lib.rs           # Org: miembros, tareas, periodos, claims
│   └── DEPLOY.md                         # Guia de deploy en Stellar
│
├── CLAUDE.md                             # Instrucciones para Claude Code
├── BLEND_YIELD_DEMO.md                   # Demo verificable de Blend yield
├── TESTING.md                            # Documentacion de testing
└── README.md                             # Este archivo
```

---

## Modulo WMS — Gestion de Almacen

### Flujo de Ordenes de Venta

El nucleo del WMS es un workflow de 8 estados con transiciones controladas por rol:

```
  [VENDEDOR]                        [JEFE BODEGA]
      │                                  │
      │  Crea orden                      │
      ▼                                  │
 ┌──────────────────┐   Aprueba         │
 │   Pendiente      │─────────────►  ┌──────────────┐
 │   Aprobacion     │                │   Aprobada   │
 └──────────────────┘   Rechaza      └──────┬───────┘
          │                │                │
          ▼                │         Asigna operario
 ┌──────────────────┐◄────┘                │
 │   Rechazada      │                      ▼
 └──────────────────┘
                           [OPERARIO]    ┌──────────────────┐
                                         │  En_Alistamiento │ ← Picking optimizado
                                         └────────┬─────────┘
                                                  │
                                           Finaliza picking
                                                  │
                                                  ▼
                                         ┌──────────────────┐
                                         │   En_Empaque     │ ← Packing + cajas
                                         └────────┬─────────┘
                                                  │
                           [FACTURACION]   Finaliza empaque
                                │                 │
                                │                 ▼
                                │        ┌──────────────────┐
                                │◄───────│ Lista_Facturar   │
                                │        └──────────────────┘
                                │
                           Genera factura
                                │
                                ▼
                         ┌─────────────┐
                         │  Facturada  │ ← Estado final
                         └─────────────┘
```

### Picking Optimizado

La funcion `getOptimizedPickingList(orden_id)` ordena los productos por ruta de bodega para minimizar el desplazamiento del operario:

- Ordenamiento: `orden_ruta` → `estanteria` → `fila` → `nivel`
- Soporte para multiples ubicaciones por producto (primaria vs. secundaria)
- Estadisticas: total items, ubicaciones a visitar, productos sin ubicacion

### Otros Modulos WMS

| Modulo | Descripcion |
|--------|-------------|
| **Inventario** | Stock por ubicacion fisica, reservas automaticas al aprobar ordenes |
| **Ubicaciones** | Mapa de bodega (estanteria/fila/nivel), asignacion de productos |
| **Recepciones** | Entrada de mercancia con asignacion a ubicaciones |
| **Averias** | Reportes de dano con tipos (Dano, Faltante, Rotura, Vencimiento) + fotos |
| **Bodegas** | Multi-bodega con transferencias inter-bodega |
| **Proveedores** | CRUD de proveedores |
| **Clientes** | Base de datos de clientes (NIT/CC, razon social, ciudad) |
| **Facturacion** | Cierre del ciclo: numeracion consecutiva, historico |
| **Desempeno** | KPIs por operario: ordenes procesadas, tiempos promedio, rankings |

---

## Modulo Harmony — Incentivos Web3

Harmony es el modulo de incentivos que conecta el desempeno operativo del WMS con recompensas on-chain en Stellar.

### Como funciona

```
1. EMPRESARIO crea un periodo de recompensas
   └─► Deposita USDC como fondo (ej: $500 USDC para el mes)
       └─► Auto-deposit en Blend Protocol (genera yield mientras esta idle)

2. SUPERVISOR asigna actividades/tareas a los WORKERS
   └─► Cada actividad tiene un template con puntos base

3. WORKERS completan tareas con evidencia
   └─► El sistema registra la actividad automaticamente desde el WMS

4. SUPERVISOR revisa y aprueba/rechaza tareas
   └─► Puede aplicar multiplicador de puntos (0% - 200%)

5. Se cierra el periodo
   └─► Auto-withdraw de Blend (USDC original + yield)
   └─► Se calculan los puntos totales de cada worker

6. Se distribuye el fondo proporcionalmente
   └─► Worker con 30% de los puntos → recibe 30% del fondo
   └─► Pago directo en USDC a su wallet Stellar
   └─► O conversion automatica a moneda local via Etherfuse

7. El yield de Blend queda para el empresario como ganancia pasiva
```

### Vistas por Rol

| Rol | Paginas Disponibles | Funciones |
|-----|---------------------|-----------|
| **Worker** | Mis Ordenes, Mi Rendimiento | Ver tareas asignadas, progreso, puntos acumulados |
| **Leader** | Actividades, Metricas, Trabajadores, Fondo, Live | Gestionar tareas, revisar evidencia, manejar periodos |
| **Admin** | HarmonyConfig | Configurar contratos, tokens, Blend, Etherfuse |

---

## Smart Contracts (Soroban/Rust)

Dos contratos desplegados en Stellar (testnet), escritos en Rust con Soroban SDK 21.7:

### Factory Contract (`contracts/factory/src/lib.rs`)

Punto de entrada del ecosistema. Registro centralizado de organizaciones.

| Funcion | Acceso | Descripcion |
|---------|--------|-------------|
| `initialize(admin)` | Una vez | Setup inicial |
| `register_org(org_address)` | Admin | Registrar nueva organizacion |
| `get_org(org_id)` | Publico | Consultar organizacion por ID |
| `get_org_count()` | Publico | Total de organizaciones |
| `transfer_admin(new_admin)` | Admin + nuevo | Transferir admin (requiere firma de ambos) |

### Organization Contract (`contracts/organization/src/lib.rs`)

Contrato principal de Harmony. Gestiona miembros, actividades, periodos y distribucion de recompensas.

**Roles on-chain:** `Owner` | `Supervisor` | `Worker`

**Estado de periodos (state machine):**
```
Open → Closed → Distributed
```

**Estado de tareas:**
```
Assigned → Completed → Approved / Rejected / Skipped
```

| Funcion | Acceso | Descripcion |
|---------|--------|-------------|
| `add_member(address, role)` | Owner | Agregar miembro a la organizacion |
| `create_activity_template(name, points)` | Owner/Supervisor | Crear tipo de actividad |
| `assign_task(worker, template_id, period_id)` | Supervisor | Asignar tarea |
| `complete_task(task_id, evidence_url)` | Worker (propio) | Marcar tarea completada |
| `review_task(task_id, multiplier, state)` | Supervisor | Aprobar/rechazar con multiplicador |
| `open_period(start, end, asset, fund)` | Owner | Crear periodo de recompensas |
| `close_period(period_id)` | Owner | Cerrar periodo |
| `distribute_period(period_id)` | Owner | Calcular y asignar recompensas |
| `claim_reward(period_id)` | Worker | Reclamar su parte on-chain |
| `sweep_expired_claims(period_id)` | Owner | Recuperar claims no reclamados (>1 ano) |
| `pause() / unpause()` | Owner | Control de emergencia |

**Garantias de seguridad:**
- `require_auth()` en todas las operaciones que mutan estado
- Guard de inicializacion (imposible reinicializar)
- State machine estricta — solo transiciones hacia adelante
- Patron CEI (Checks-Effects-Interactions) en claims
- Overflow checks en aritmetica
- Multiplicador acotado [0%, 200%] (basis points)
- Prevencion de double-claim (marca antes de transferir)
- TTL extendido a 365 dias

---

## Blend Protocol — Yield Automatico

Cuando un empresario deposita USDC como fondo de incentivos, ese dinero normalmente queda idle (sin producir) durante todo el periodo. Con la integracion de Blend Protocol, el USDC se deposita automaticamente en un pool de lending DeFi en Stellar, generando rendimiento.

### Flujo

```
EMPRESARIO                         HARMONY (Backend)                   BLEND PROTOCOL (Soroban)
    |                                     |                                     |
    |--- Deposita USDC (fondo) ---------> |                                     |
    |                                     |--- Auto-deposit USDC -------------> |
    |                                     |    (SupplyCollateral)                |
    |                                     |<-- Recibe bTokens -----------------|
    |                                     |                                     |
    |   [ USDC genera yield en Blend mientras el periodo esta abierto ]        |
    |                                     |                                     |
    |                                     |--- Lider cierra periodo             |
    |                                     |--- Auto-withdraw USDC + yield ---> |
    |                                     |    (WithdrawCollateral)              |
    |                                     |<-- USDC original + yield ----------|
    |                                     |                                     |
    |                                     |--- Distribuir a workers             |
    |<-- Yield queda en wallet admin -----|                                     |
    |    (ganancia del empresario)         |                                     |
```

### Prueba E2E verificada en Testnet

| Paso | Operacion | Monto | TX Hash |
|------|-----------|-------|---------|
| 1 | Adquisicion USDC (DEX) | 10 USDC | `e742be77...087a` |
| 2 | Supply a Blend | 3 USDC | `e095cc90...369d` |
| 3 | Withdraw + yield | **3.0909742 USDC** | `9e21f142...ce67` |
| | **Yield ganado** | **+0.0909742 USDC (~3.03%)** | |

### Resiliencia

| Escenario | Comportamiento |
|-----------|----------------|
| Blend desactivado | USDC queda en wallet admin, distribucion normal |
| Deposit falla | Warning en log, fondo se confirma sin Blend |
| Withdraw falla | Posicion `Failed`, distribucion usa balance del admin |
| Blend SDK no disponible | Lazy-loading con try/catch, sistema opera sin Blend |

### Proyeccion de Rendimiento (Mainnet, ~5-7% APY)

| Fondo | Periodo | Yield estimado |
|-------|---------|----------------|
| $100 USDC | 15 dias | +$0.21 - $0.29 |
| $500 USDC | 30 dias | +$2.05 - $2.88 |
| $1,000 USDC | 30 dias | +$4.11 - $5.75 |
| $5,000 USDC | 30 dias | +$20.55 - $28.77 |
| $10,000 USDC | 90 dias | +$123.29 - $172.60 |

---

## Etherfuse — Cash-Out a Cuenta Bancaria

Etherfuse FX permite que los operarios reciban sus recompensas en moneda local directamente en su cuenta bancaria, sin necesidad de manejar cripto.

### Flujo

```
Worker gana recompensa
    └─► Backend calcula monto en USDC
        └─► Llama a Etherfuse FX API
            └─► Etherfuse convierte USDC → MXN (o moneda local)
                └─► Deposito directo a cuenta bancaria del worker
```

### Datos bancarios del worker

Cada usuario puede registrar sus datos bancarios en su perfil:
- Nombre del banco
- Numero de cuenta / CLABE
- Moneda de preferencia

El backend almacena estos datos y los usa al momento de la distribucion para ejecutar el pago via Etherfuse automaticamente.

---

## Base de Datos

### Tablas WMS (13 tablas)

| Tabla | Descripcion |
|-------|-------------|
| `usuarios` | Usuarios con roles (Vendedor, Jefe_Bodega, Operario, Facturacion, Admin) |
| `clientes` | Base de clientes (NIT/CC, razon social, contacto, ciudad) |
| `productos` | Catalogo de productos (codigo, precios, stock) |
| `ordenes_venta` | Ordenes con workflow de 8 estados |
| `orden_detalles` | Items de cada orden (cantidad pedida/alistada/empacada) |
| `ubicaciones` | Posiciones fisicas de bodega (estanteria/fila/nivel/orden_ruta) |
| `inventario_ubicaciones` | Stock por ubicacion + reservas |
| `bodegas` | Multi-bodega |
| `inventario_bodegas` | Stock consolidado por bodega |
| `recepciones` / `recepcion_detalles` | Entrada de mercancia |
| `averias` | Reportes de dano con evidencia |
| `proveedores` | Proveedores |
| `desempeno_actividades` | Registro de actividades para KPIs |

### Tablas Harmony (6 tablas)

| Tabla | Descripcion |
|-------|-------------|
| `harmony_config` | Configuracion global (contratos, tokens, Blend, Etherfuse) |
| `harmony_periodos` | Periodos de recompensas con estado y fondo |
| `harmony_plantillas_actividad` | Templates de actividades con puntos base |
| `harmony_tareas` | Asignaciones individuales de tareas |
| `harmony_puntos_periodo` | Puntos ganados y montos ajustados por worker/periodo |
| `harmony_historial_cambios` | Log de auditoria |

### Tabla Blend (1 tabla)

| Tabla | Descripcion |
|-------|-------------|
| `harmony_blend_positions` | Posiciones en Blend: depositos, retiros, yield, tx hashes |

### Configuracion

- **Timezone:** `America/Bogota` (UTC-5) configurada automaticamente en cada conexion
- **SSL:** Habilitado para compatibilidad con Azure PostgreSQL
- **Pool:** Max 10 conexiones, idle timeout 20s
- **Transacciones:** Soporte ACID via `getClient()` con BEGIN/COMMIT/ROLLBACK

---

## API REST

### Autenticacion

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/auth/login` | Login (email + password) → JWT token |
| POST | `/api/auth/register` | Registro de nuevo usuario |
| GET | `/api/auth/profile` | Perfil del usuario autenticado |
| POST | `/api/auth/refresh` | Refrescar JWT token |

### Ordenes de Venta

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/ordenes` | Crear orden |
| GET | `/api/ordenes` | Listar ordenes (filtros por estado, vendedor, fecha) |
| GET | `/api/ordenes/:id` | Detalle de una orden |
| PUT | `/api/ordenes/:id` | Actualizar orden |
| POST | `/api/ordenes/:id/aprobar` | Aprobar orden |
| POST | `/api/ordenes/:id/rechazar` | Rechazar orden |
| POST | `/api/ordenes/:id/alistamiento` | Iniciar/finalizar picking |
| POST | `/api/ordenes/:id/empaque` | Iniciar/finalizar packing |
| POST | `/api/ordenes/:id/facturar` | Facturar orden |

### Inventario y Ubicaciones

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/api/inventario/ubicaciones` | Inventario por ubicacion |
| GET | `/api/inventario/bodega/:id` | Inventario de una bodega |
| GET | `/api/ubicaciones` | Listar ubicaciones |
| POST | `/api/ubicaciones` | Crear ubicacion |

### Harmony (Web3)

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/api/harmony/config` | Obtener configuracion Harmony |
| POST | `/api/harmony/config` | Actualizar configuracion |
| GET | `/api/harmony/usuarios-wallets` | Usuarios con wallets Stellar |
| POST | `/api/harmony/periodo` | Crear periodo de recompensas |
| POST | `/api/harmony/periodo/:id/confirmar-fondo` | Confirmar fondo (+ auto-deposit Blend) |
| POST | `/api/harmony/periodo/:id/cerrar` | Cerrar periodo |
| POST | `/api/harmony/distribuir` | Distribuir recompensas (+ auto-withdraw Blend) |

### Otros Modulos

Endpoints similares (CRUD) para: productos, clientes, proveedores, recepciones, averias, bodegas, transferencias, desempeno, upload.

**Total: 70+ endpoints REST.**

---

## Roles y Permisos (RBAC)

```
┌───────────────────┬──────────┬────────────┬──────────┬──────────┬───────────────┐
│     Accion        │ Vendedor │Jefe_Bodega │ Operario │Facturac. │Administrador  │
├───────────────────┼──────────┼────────────┼──────────┼──────────┼───────────────┤
│ Crear ordenes     │    ✓     │     ✓      │    ✗     │    ✗     │      ✓        │
│ Aprobar/Rechazar  │    ✗     │     ✓      │    ✗     │    ✗     │      ✓        │
│ Picking           │    ✗     │     ✓      │    ✓     │    ✗     │      ✓        │
│ Packing           │    ✗     │     ✓      │    ✓     │    ✗     │      ✓        │
│ Facturar          │    ✗     │     ✓      │    ✗     │    ✓     │      ✓        │
│ Inventario (ver)  │    ✓     │     ✓      │    ✓     │    ✓     │      ✓        │
│ Inventario (edit) │    ✗     │     ✓      │    ✗     │    ✗     │      ✓        │
│ Clientes          │    ✓     │     ✓      │    ✗     │    ✓     │      ✓        │
│ Desempeno (propio)│    ✗     │     ✗      │    ✓     │    ✗     │      ✓        │
│ Desempeno (global)│    ✗     │     ✓      │    ✗     │    ✗     │      ✓        │
│ Averias           │    ✓     │     ✓      │    ✓     │    ✓     │      ✓        │
│ Harmony config    │    ✗     │     ✗      │    ✗     │    ✗     │      ✓        │
│ Harmony leader    │    ✗     │     ✓      │    ✗     │    ✗     │      ✓        │
│ Harmony worker    │    ✗     │     ✗      │    ✓     │    ✗     │      ✓        │
└───────────────────┴──────────┴────────────┴──────────┴──────────┴───────────────┘
```

---

## Seguridad

| Capa | Mecanismo |
|------|-----------|
| **Autenticacion** | JWT (24h) + refresh tokens (7d) |
| **Passwords** | bcrypt con salt |
| **Autorizacion** | RBAC con `checkRole()` middleware |
| **HTTP** | Helmet (headers de seguridad) |
| **Rate Limiting** | 500 req/15min global, 20 req/15min en auth |
| **SQL Injection** | Queries parametrizadas (pg driver) |
| **CORS** | Whitelist configurable |
| **Archivos** | Multer con filtro de tipos y tamano maximo |
| **Smart Contracts** | `require_auth()`, state machine, CEI pattern, overflow checks |
| **Datos sensibles** | `.env` no versionado, SSL para PostgreSQL en Azure |

---

## Guia de Inicio Rapido

### Prerrequisitos

- Node.js 20+
- PostgreSQL 14+
- Git

### 1. Clonar el repositorio

```bash
git clone https://github.com/MariaDelina/PruebaUSDC.git
cd PruebaUSDC
```

### 2. Configurar la base de datos

```bash
psql -U postgres -c "CREATE DATABASE wms_db;"
psql -U postgres -d wms_db -f back/database/schema.sql
psql -U postgres -d wms_db -f back/database/seed.sql                # Datos iniciales
psql -U postgres -d wms_db -f back/database/seed_ordenes_flujo.sql  # Ordenes de prueba
```

### 3. Backend

```bash
cd back
npm install
cp .env.example .env   # Editar con tus credenciales
npm run dev             # http://localhost:3000
```

### 4. Frontend

```bash
cd front
npm install
npm run dev             # http://localhost:5173
```

### 5. Credenciales de prueba

| Rol | Email | Password |
|-----|-------|----------|
| Administrador | admin@wms.com | admin123 |
| Vendedor | vendedor@wms.com | vendedor123 |
| Jefe Bodega | jefe@wms.com | jefe123 |
| Operario | operario1@wms.com | operario123 |
| Facturacion | facturacion@wms.com | facturacion123 |

---

## Variables de Entorno

### Backend (`back/.env`)

```env
# Servidor
PORT=3000
NODE_ENV=development
LOG_LEVEL=debug

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=tu_password
DB_NAME=wms_db

# JWT
JWT_SECRET=clave_secreta_minimo_32_caracteres
JWT_EXPIRES_IN=24h
JWT_REFRESH_EXPIRES_IN=7d

# CORS
CORS_ORIGIN=http://localhost:5173

# Inventario
ENABLE_INVENTORY_RESERVATION=true

# Stellar / Harmony
STELLAR_STUB=true                     # true = simula sin red real
STELLAR_NETWORK=testnet               # testnet | mainnet
STELLAR_ADMIN_SECRET=S...             # Clave privada admin
FACTORY_CONTRACT_ID=C...              # Factory contract en Stellar
ORG_CONTRACT_ID=C...                  # Organization contract
```

### Frontend (`front/.env`)

```env
VITE_API_URL=http://localhost:3000/api
```

---

## Despliegue en Azure

### Frontend → Azure Static Web Apps

```bash
cd front
npm run build
# Deploy dist/ a Azure Static Web Apps
# El archivo staticwebapp.config.json maneja el routing SPA
```

### Backend → Azure App Service (Linux)

```bash
cd back
# CI/CD configurado en azure-pipelines.yml
# Trigger: push a main/master
# Deploy automatico a Azure Web App
```

### Base de Datos → Azure Database for PostgreSQL

- SSL requerido (ya configurado en `db.js`)
- Timezone se configura automaticamente en cada conexion

---

## Datos de Prueba

El proyecto incluye varios seeds para poblar la BD con datos de prueba:

| Archivo | Contenido |
|---------|-----------|
| `seed.sql` | 6 usuarios, 5 clientes, 15 productos, 10 ubicaciones, inventario, 6 ordenes |
| `seed_ordenes_demo.sql` | 3 ordenes en estado Aprobada (3, 5 y 6 productos) |
| `seed_ordenes_flujo.sql` | 3 ordenes para practicar el flujo completo |

### Ordenes de Flujo (para practicar)

| Orden | Estado | Productos | Para practicar |
|-------|--------|-----------|----------------|
| ORD-FLUJO-001 | Pendiente_Aprobacion | 2 | Aprobar o rechazar |
| ORD-FLUJO-002 | Aprobada | 3 | Iniciar alistamiento (picking) |
| ORD-FLUJO-003 | Pendiente_Aprobacion | 3 (con 10% dto) | Flujo completo: aprobar → alistar → empacar → facturar |

---

## Verificacion On-Chain

Todas las transacciones de Harmony son verificables en Stellar Testnet:

| Recurso | Link |
|---------|------|
| Wallet Admin | [stellar.expert/account/GBEP4X...](https://stellar.expert/explorer/testnet/account/GBEP4XMMRPFAI7NNTOGAMOBF6ELKD5WYMRLONPFZV6Z2TNWIQLOKMRQ7) |
| Blend Pool | [stellar.expert/contract/CCEBVD...](https://stellar.expert/explorer/testnet/contract/CCEBVDYM32YNYCVNRXQKDFFPISJJCV557CDZEIRBEE4NCV4KHPQ44HGF) |
| Supply TX | [stellar.expert/tx/e095cc90...](https://stellar.expert/explorer/testnet/tx/e095cc90526a6f18f9e77ea69e4c601ba986743ae3ab85afb8beea6eed81369d) |
| Withdraw TX | [stellar.expert/tx/9e21f142...](https://stellar.expert/explorer/testnet/tx/9e21f14213c3e58e3ce1a4cb9ead6384495bf30a68ec755f7bebd2cfcb79ce67) |

---

## Licencia

Proyecto academico / demostrativo. Desarrollado para la gestion de almacen con incentivos Web3 sobre Stellar.

---

<p align="center">
  <b>Harmony WMS</b> — Gestion de almacen + incentivos transparentes on-chain
</p>
