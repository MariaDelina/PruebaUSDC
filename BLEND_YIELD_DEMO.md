# Harmony + Blend Protocol: Yield Automatico sobre USDC

**Fecha de prueba:** 22 de marzo de 2026
**Red:** Stellar Testnet (Soroban)
**Pool de Blend:** `CCEBVDYM32YNYCVNRXQKDFFPISJJCV557CDZEIRBEE4NCV4KHPQ44HGF`
**Wallet Admin:** `GBEP4XMMRPFAI7NNTOGAMOBF6ELKD5WYMRLONPFZV6Z2TNWIQLOKMRQ7`

---

## 1. Problema que resuelve

Cuando un empresario deposita USDC en Harmony para incentivar a sus operarios, ese dinero queda **idle** (sin producir) durante todo el periodo (dias o semanas) hasta que se distribuye.

**Con Blend Protocol**, ese USDC idle se deposita automaticamente en un pool de lending DeFi en Stellar, generando rendimiento (~5-7% APY). Al cerrar el periodo, se retira el USDC + yield. El empresario gana por proveer liquidez sin hacer nada extra.

---

## 2. Flujo del Sistema

```
EMPRESARIO                          HARMONY (Backend)                    BLEND PROTOCOL (Soroban)
    |                                      |                                      |
    |--- Deposita USDC (fondo) ----------->|                                      |
    |                                      |--- Auto-deposit USDC --------------->|
    |                                      |    (SupplyCollateral)                 |
    |                                      |<-- Recibe bTokens -------------------|
    |                                      |                                      |
    |   [El USDC genera yield en Blend mientras el periodo esta abierto]          |
    |                                      |                                      |
    |                                      |--- Lider cierra periodo              |
    |                                      |--- Calcular puntos                   |
    |                                      |                                      |
    |                                      |--- Auto-withdraw USDC + yield ------>|
    |                                      |    (WithdrawCollateral)               |
    |                                      |<-- USDC original + yield ------------|
    |                                      |                                      |
    |                                      |--- Distribuir a workers              |
    |                                      |    (Etherfuse: USDC -> MXN)          |
    |                                      |                                      |
    |<-- Yield queda en wallet admin ------|                                      |
    |    (ganancia del empresario)          |                                      |
```

**Clave:** Los workers reciben exactamente su monto calculado. El yield extra es ganancia del empresario.

---

## 3. Prueba E2E Verificable en Blockchain

### Paso 1: Obtener USDC de Blend (via DEX Stellar)

Se adquirieron 10 USDC del pool de Blend mediante un path payment en el DEX de Stellar:

| Campo | Valor |
|-------|-------|
| Operacion | Path Payment (XLM -> USDC) |
| Monto enviado | ~544 XLM |
| Monto recibido | 10.0000000 USDC |
| TX Hash | `e742be77a352fe62f216c47ce2bd55614daaf94ebd13ca6716d36343d44e087a` |
| Verificar | [Ver en Stellar Expert](https://stellar.expert/explorer/testnet/tx/e742be77a352fe62f216c47ce2bd55614daaf94ebd13ca6716d36343d44e087a) |

---

### Paso 2: Confirmar Fondo -> Auto-deposit en Blend

Al confirmar el fondo del periodo, el backend automaticamente deposito 3 USDC en el pool de Blend via Soroban:

| Campo | Valor |
|-------|-------|
| Operacion | `SupplyCollateral` (Soroban invoke) |
| Monto depositado | **3.0000000 USDC** |
| Pool Blend | `CCEBVDYM32YN...Q44HGF` |
| TX Hash | `e095cc90526a6f18f9e77ea69e4c601ba986743ae3ab85afb8beea6eed81369d` |
| Verificar | [Ver en Stellar Expert](https://stellar.expert/explorer/testnet/tx/e095cc90526a6f18f9e77ea69e4c601ba986743ae3ab85afb8beea6eed81369d) |
| Fecha | 2026-03-22 01:37:23 (Colombia) |

**Resultado:** El USDC salio de la wallet admin y se convirtio en bTokens dentro del pool de Blend. El USDC ahora esta generando yield.

---

### Paso 3: Distribuir -> Auto-withdraw de Blend con Yield

Al distribuir el fondo, el backend automaticamente retiro el USDC + yield del pool de Blend:

| Campo | Valor |
|-------|-------|
| Operacion | `WithdrawCollateral` (Soroban invoke) |
| Monto depositado original | 3.0000000 USDC |
| **Monto retirado** | **3.0909742 USDC** |
| **Yield ganado** | **+0.0909742 USDC (~3.03%)** |
| TX Hash | `9e21f14213c3e58e3ce1a4cb9ead6384495bf30a68ec755f7bebd2cfcb79ce67` |
| Verificar | [Ver en Stellar Expert](https://stellar.expert/explorer/testnet/tx/9e21f14213c3e58e3ce1a4cb9ead6384495bf30a68ec755f7bebd2cfcb79ce67) |
| Fecha | 2026-03-22 01:37:34 (Colombia) |

**Resultado:** El empresario recupero su USDC original + 0.09 USDC de yield. Los workers recibieron su pago via Etherfuse (USDC -> MXN) sin afectarse.

---

### Paso 4: Verificacion On-Chain

**Balance final de la wallet admin despues de la prueba:**

| Activo | Balance |
|--------|---------|
| USDC (Blend) | 9.8293023 |
| USDC (Etherfuse) | 142.0322813 |
| XLM | 9,230.07 |

**Registro en base de datos (harmony_blend_positions):**

| Campo | Valor |
|-------|-------|
| position_id | 2 |
| periodo_id | 22 |
| estado | **Withdrawn** |
| monto_deposited | 3.0000000 |
| monto_withdrawn | 3.0909742 |
| yield_earned | **0.0909742** |
| supply_tx_hash | `e095cc90...81369d` |
| withdraw_tx_hash | `9e21f142...79ce67` |

---

## 4. Arquitectura Tecnica

### Componentes involucrados

```
Frontend (React)                    Backend (Node.js/Express)             Blockchain (Stellar/Soroban)
+------------------+               +------------------------+            +---------------------+
| HarmonyConfig    |               | harmony.controller.js  |            | Blend Pool Contract |
| - Toggle Blend   |--API--------->| - confirmarFondo()     |--Soroban-->| - submit()          |
| - Pool address   |               |   hook: supplyUSDC()   |            |   SupplyCollateral   |
| - USDC contract  |               | - distribuirFondo()    |            |   WithdrawCollateral |
|                  |               |   hook: withdrawUSDC() |            |                     |
| Fondo.jsx        |               |                        |            | Stellar DEX         |
| - Yield badge    |<--API---------| blend.service.js       |            | (path payments)     |
+------------------+               | - supplyUSDC()         |            +---------------------+
                                   | - withdrawUSDC()       |
                                   | - getPosition()        |
                                   +------------------------+
                                            |
                                   +------------------------+
                                   | harmony.model.js       |
                                   | - insertBlendPosition  |
                                   | - updateBlendWithdraw  |
                                   | - getBlendPositions    |
                                   +------------------------+
                                            |
                                   +------------------------+
                                   | PostgreSQL (Azure)     |
                                   | harmony_blend_positions|
                                   +------------------------+
```

### Archivos clave

| Archivo | Funcion |
|---------|---------|
| `back/src/services/blend.service.js` | Interaccion con Blend Protocol via Soroban |
| `back/src/services/stellar.service.js` | Pipeline Soroban (build -> sign -> submit) |
| `back/src/controllers/harmony.controller.js` | Hooks automaticos en confirmarFondo y distribuirFondo |
| `back/src/models/harmony.model.js` | CRUD de posiciones Blend en PostgreSQL |
| `back/database/migrations/018_blend_yield.sql` | Schema de la tabla harmony_blend_positions |
| `front/src/pages/harmony/admin/HarmonyConfig.jsx` | UI para activar/configurar Blend |
| `front/src/pages/harmony/leader/Fondo.jsx` | Indicador visual de yield por periodo |

### Contratos Soroban utilizados

| Contrato | Direccion | Uso |
|----------|-----------|-----|
| Blend Pool V2 | `CCEBVDYM32YNYCVNRXQKDFFPISJJCV557CDZEIRBEE4NCV4KHPQ44HGF` | Pool de lending donde se deposita USDC |
| USDC (SAC) | `CAQCFVLOBK5GIULPNZRGATJJMIZL5BSP7X5YJVMGCPTUEPFM4AVSRCJU` | Token USDC en Soroban |

### SDK y dependencias

| Paquete | Version | Uso |
|---------|---------|-----|
| `@blend-capital/blend-sdk` | ^1.x | Interaccion con pools de Blend |
| `@stellar/stellar-sdk` | ^12.x | Transacciones Stellar y Soroban |

---

## 5. Resiliencia y Fallbacks

El sistema esta disenado para **nunca bloquear la distribucion** si Blend falla:

| Escenario | Comportamiento |
|-----------|----------------|
| Blend desactivado (`blend_enabled: false`) | USDC queda en wallet admin, distribucion normal |
| Deposit falla (ej: USDC insuficiente) | Se loguea warning, fondo se confirma normal |
| Withdraw falla (ej: pool sin liquidez) | Posicion marcada como `Failed`, distribucion usa balance del admin |
| Blend SDK no disponible | Lazy-loading con try/catch, sistema opera sin Blend |

---

## 6. Proyeccion de Rendimiento

Con Blend Protocol en mainnet (USDC de Circle, ~5-7% APY):

| Fondo depositado | Periodo (dias) | Yield estimado |
|-------------------|----------------|----------------|
| $100 USDC | 15 dias | +$0.21 - $0.29 USDC |
| $500 USDC | 30 dias | +$2.05 - $2.88 USDC |
| $1,000 USDC | 30 dias | +$4.11 - $5.75 USDC |
| $5,000 USDC | 30 dias | +$20.55 - $28.77 USDC |
| $10,000 USDC | 90 dias | +$123.29 - $172.60 USDC |

**El empresario gana rendimiento pasivo por el simple hecho de usar el sistema de incentivos.**

---

## 7. Links de Verificacion

Todas las transacciones son verificables publicamente en Stellar Testnet:

- **Deposit (Supply):** [stellar.expert/tx/e095cc90...](https://stellar.expert/explorer/testnet/tx/e095cc90526a6f18f9e77ea69e4c601ba986743ae3ab85afb8beea6eed81369d)
- **Withdraw + Yield:** [stellar.expert/tx/9e21f142...](https://stellar.expert/explorer/testnet/tx/9e21f14213c3e58e3ce1a4cb9ead6384495bf30a68ec755f7bebd2cfcb79ce67)
- **Wallet Admin:** [stellar.expert/account/GBEP4X...](https://stellar.expert/explorer/testnet/account/GBEP4XMMRPFAI7NNTOGAMOBF6ELKD5WYMRLONPFZV6Z2TNWIQLOKMRQ7)
- **Blend Pool:** [stellar.expert/contract/CCEBVD...](https://stellar.expert/explorer/testnet/contract/CCEBVDYM32YNYCVNRXQKDFFPISJJCV557CDZEIRBEE4NCV4KHPQ44HGF)

---

*Documento generado automaticamente — Harmony WMS + Blend Protocol Integration*
