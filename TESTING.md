# Guía de Testing — Harmony + WMS

Esta guía cubre el flujo completo desde cero: verificar la migración SQL,
probar el backend con curl, desplegar contratos en Testnet y ejecutar
el flujo end-to-end real en la red Stellar.

---

## Índice

1. [Prerrequisitos](#1-prerrequisitos)
2. [Verificar migración SQL](#2-verificar-migración-sql)
3. [Levantar el backend](#3-levantar-el-backend)
4. [Pruebas de API en Stub Mode](#4-pruebas-de-api-en-stub-mode)
5. [Desplegar contratos en Testnet](#5-desplegar-contratos-en-testnet)
6. [Conectar backend con contratos reales](#6-conectar-backend-con-contratos-reales)
7. [Flujo end-to-end real](#7-flujo-end-to-end-real)
8. [Testing del frontend](#8-testing-del-frontend)
9. [Errores comunes](#9-errores-comunes)

---

## 1. Prerrequisitos

### Backend
```bash
cd back
npm install
# Instalar SDK de Stellar (para modo real, no necesario en stub)
npm install @stellar/stellar-sdk
```

### Contratos (solo para modo real)
```bash
# Instalar Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Target WebAssembly
rustup target add wasm32-unknown-unknown

# Stellar CLI
cargo install --locked stellar-cli --features opt
```

---

## 2. Verificar migración SQL

La migración `013_add_harmony_tables.sql` ya fue aplicada en Azure.
Para verificar que las tablas existen:

```bash
# Conectar a Azure PostgreSQL
PGPASSWORD="Plastico123*" psql \
  -h hogarcenter.postgres.database.azure.com \
  -U HcAdmin \
  -d hogarcenter \
  -c "\dt harmony_*"
```

**Resultado esperado:**
```
           List of relations
 Schema |          Name           | Type  |  Owner
--------+-------------------------+-------+---------
 public | harmony_config          | table | HcAdmin
 public | harmony_fondos          | table | HcAdmin
 public | harmony_periodos        | table | HcAdmin
 public | harmony_puntos_periodo  | table | HcAdmin
 public | harmony_reclamaciones   | table | HcAdmin
```

Verificar que `stellar_wallet` fue agregada a `usuarios`:
```bash
PGPASSWORD="Plastico123*" psql \
  -h hogarcenter.postgres.database.azure.com \
  -U HcAdmin \
  -d hogarcenter \
  -c "\d usuarios" | grep stellar
```

Verificar config inicial:
```bash
PGPASSWORD="Plastico123*" psql \
  -h hogarcenter.postgres.database.azure.com \
  -U HcAdmin \
  -d hogarcenter \
  -c "SELECT * FROM harmony_config;"
```

**Resultado esperado:** Una fila con `activo = false`.

---

## 3. Levantar el backend

```bash
cd back
npm run dev
```

Verificar que arranca sin errores:
```bash
curl http://localhost:3000/health
# {"status":"ok","timestamp":"..."}
```

Obtener token de admin:
```bash
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@wms.com","password":"admin123"}' \
  | jq -r '.data.token')

echo "Token: $TOKEN"
```

---

## 4. Pruebas de API en Stub Mode

Con `STELLAR_STUB=true` (por defecto), todas las operaciones simulan la blockchain.
Ideal para desarrollo sin contratos desplegados.

### 4.1 Verificar que Harmony está desactivado

```bash
curl -s http://localhost:3000/api/config | jq .
# "harmonyEnabled": false
```

### 4.2 Activar Harmony

```bash
curl -s -X PATCH http://localhost:3000/api/harmony/config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "activo": true,
    "puntos_por_picking": 15,
    "puntos_por_packing": 10
  }' | jq .
```

Verificar que está activo:
```bash
curl -s http://localhost:3000/api/config | jq '.data.harmonyEnabled'
# true
```

### 4.3 Crear un período

```bash
curl -s -X POST http://localhost:3000/api/harmony/periodos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Febrero 2026",
    "fecha_inicio": "2026-02-01",
    "fecha_fin": "2026-02-28"
  }' | jq .
```

Guarda el `periodo_id` devuelto.

### 4.4 Calcular puntos

```bash
PERIODO_ID=1  # usar el id devuelto arriba

curl -s -X POST http://localhost:3000/api/harmony/periodos/$PERIODO_ID/calcular \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Muestra cuántos trabajadores tienen puntos según las órdenes del WMS.

### 4.5 Registrar un fondo

```bash
curl -s -X POST http://localhost:3000/api/harmony/periodos/$PERIODO_ID/fondos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "monto_total": 500.0,
    "notas": "Fondo de prueba Febrero 2026"
  }' | jq .
```

### 4.6 Cerrar el período y distribuir (stub)

```bash
# Cerrar
curl -s -X PATCH http://localhost:3000/api/harmony/periodos/$PERIODO_ID/cerrar \
  -H "Authorization: Bearer $TOKEN" | jq .

# Distribuir (en stub mode devuelve STUB_DIST_xxxx como tx_hash)
curl -s -X POST http://localhost:3000/api/harmony/periodos/$PERIODO_ID/distribuir \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### 4.7 Asignar wallet a un trabajador

```bash
# Primero listar trabajadores
curl -s http://localhost:3000/api/harmony/trabajadores \
  -H "Authorization: Bearer $TOKEN" | jq '.data[].usuario_id'

# Asignar wallet (reemplaza USUARIO_ID y la clave pública)
curl -s -X PATCH http://localhost:3000/api/harmony/trabajadores/2/wallet \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"stellar_wallet": "GCEZWKCA5VLDNRLN3RPRJMRZOX3Z6G5CHCGDV8P7FFDKQVG5TYAQVF3"}' \
  | jq .
```

### 4.8 Ver rendimiento del trabajador (login con otro usuario)

```bash
TOKEN_OP=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"operario@wms.com","password":"op123"}' \
  | jq -r '.data.token')

curl -s http://localhost:3000/api/harmony/mi-rendimiento \
  -H "Authorization: Bearer $TOKEN_OP" | jq .
```

---

## 5. Desplegar contratos en Testnet

Ver también: `contracts/DEPLOY.md`

### 5.1 Compilar contratos

```bash
cd contracts
stellar contract build
```

Los `.wasm` quedan en:
- `factory/target/wasm32-unknown-unknown/release/harmony_factory.wasm`
- `organization/target/wasm32-unknown-unknown/release/harmony_organization.wasm`

Correr tests unitarios antes de desplegar:
```bash
cargo test
```

**Resultado esperado:** todos los tests en verde.

### 5.2 Configurar red y keypair

```bash
# Agregar red testnet
stellar network add testnet \
  --rpc-url https://soroban-testnet.stellar.org \
  --network-passphrase "Test SDF Network ; September 2015"

# Generar keypair del admin y fondear con Friendbot
stellar keys generate harmony-admin --network testnet --fund

# Ver dirección pública del admin
stellar keys address harmony-admin
```

### 5.3 Desplegar Factory

```bash
FACTORY_CONTRACT_ID=$(stellar contract deploy \
  --wasm factory/target/wasm32-unknown-unknown/release/harmony_factory.wasm \
  --source harmony-admin \
  --network testnet)

echo "FACTORY_CONTRACT_ID=$FACTORY_CONTRACT_ID"
```

Inicializar:
```bash
stellar contract invoke \
  --id $FACTORY_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- initialize \
  --admin $(stellar keys address harmony-admin)
```

### 5.4 Desplegar Organization

```bash
ORG_CONTRACT_ID=$(stellar contract deploy \
  --wasm organization/target/wasm32-unknown-unknown/release/harmony_organization.wasm \
  --source harmony-admin \
  --network testnet)

echo "ORG_CONTRACT_ID=$ORG_CONTRACT_ID"
```

Inicializar (USDC en testnet: `CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA`):
```bash
stellar contract invoke \
  --id $ORG_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- initialize \
  --owner $(stellar keys address harmony-admin) \
  --reward_asset CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA
```

Registrar la organización en el Factory:
```bash
stellar contract invoke \
  --id $FACTORY_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- register_org \
  --org_address $ORG_CONTRACT_ID
```

Verificar estado del contrato:
```bash
stellar contract invoke \
  --id $ORG_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- get_owner
```

---

## 6. Conectar backend con contratos reales

### 6.1 Actualizar `.env`

```env
STELLAR_STUB=false
STELLAR_NETWORK=testnet
STELLAR_ADMIN_SECRET=<secret key de harmony-admin — NUNCA en git>
ORG_CONTRACT_ID=<valor devuelto en paso 5.4>
FACTORY_CONTRACT_ID=<valor devuelto en paso 5.3>
```

> **Seguridad:** obtén la secret key con:
> ```bash
> stellar keys show harmony-admin
> ```
> Cópiala en `.env` pero **no la subas al repositorio**.

### 6.2 Actualizar config de Harmony en DB

Una vez desplegados los contratos, actualizar los IDs en la base de datos:

```bash
curl -s -X PATCH http://localhost:3000/api/harmony/config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "activo": true,
    "org_contract_address": "<ORG_CONTRACT_ID>",
    "factory_contract_address": "<FACTORY_CONTRACT_ID>",
    "reward_asset_code": "USDC",
    "reward_asset_issuer": "CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA"
  }' | jq .
```

### 6.3 Fondear el contrato de organización

Antes de distribuir, el contrato debe tener USDC.
En testnet puedes obtener USDC de prueba en: https://stellar.expert/explorer/testnet

Enviar USDC al contrato (desde cualquier wallet con USDC testnet):
```bash
stellar contract invoke \
  --id $ORG_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- fund_period \
  --caller $(stellar keys address harmony-admin) \
  --funder $(stellar keys address harmony-admin) \
  --period_id 1 \
  --amount 500000000  # 50 USDC con 7 decimales
```

---

## 7. Flujo end-to-end real

Con `STELLAR_STUB=false` y los contratos desplegados:

### Paso 1 — Crear y cerrar período
```bash
# Crear período
curl -s -X POST http://localhost:3000/api/harmony/periodos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Marzo 2026","fecha_inicio":"2026-03-01","fecha_fin":"2026-03-31"}' \
  | jq '.data.periodo_id'

# Calcular puntos
curl -s -X POST http://localhost:3000/api/harmony/periodos/2/calcular \
  -H "Authorization: Bearer $TOKEN" | jq .

# Cerrar
curl -s -X PATCH http://localhost:3000/api/harmony/periodos/2/cerrar \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### Paso 2 — Registrar y confirmar fondo
```bash
# Registrar depósito (con TX hash del fondo ya enviado al contrato)
curl -s -X POST http://localhost:3000/api/harmony/periodos/2/fondos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "monto_total": 50.0,
    "stellar_tx_hash": "<tx_hash_del_fund_period>",
    "notas": "Fondo Marzo 2026"
  }' | jq .
```

### Paso 3 — Distribuir (llamada real a Soroban)
```bash
# Esto llama distribute_rewards en el contrato y espera ~15 segundos
curl -s -X POST http://localhost:3000/api/harmony/periodos/2/distribuir \
  -H "Authorization: Bearer $TOKEN" | jq .
```

La respuesta incluirá el `tx_hash` real de la red Stellar.
Puedes verificarlo en: https://stellar.expert/explorer/testnet/tx/<tx_hash>

### Paso 4 — Worker reclama su bono

```bash
# Como trabajador (con token de operario)
curl -s -X POST http://localhost:3000/api/harmony/reclamar/2 \
  -H "Authorization: Bearer $TOKEN_OP" | jq .
```

En modo real la respuesta incluye un `xdr` — la transacción sin firmar.
El frontend debe presentar ese XDR a Freighter para que el worker firme.

### Verificar saldo on-chain
```bash
curl -s http://localhost:3000/api/harmony/... # endpoint de balance (pendiente)
# O directamente en Horizon:
curl -s "https://horizon-testnet.stellar.org/accounts/<wallet_del_worker>" \
  | jq '.balances[] | select(.asset_code=="USDC")'
```

---

## 8. Testing del frontend

### Flujo Líder (Jefe_Bodega / Administrador)

1. Ir a `http://localhost:5173`
2. Hacer login como Jefe_Bodega
3. En el menú lateral, clic en **Harmony ✦**
4. Verificar que redirige a `/harmony/mis-ordenes`

**Activar Harmony (Administrador):**
- Ir a **Harmony > Actividades** (pestaña inferior)
- Si Harmony está desactivado, verás un mensaje con código `HARMONY_DISABLED`
- Activar desde la config de Harmony

**Crear período:**
- Pestaña **Actividades** → botón "Nuevo Período"
- Llenar nombre, fecha inicio y fin
- Clic "Crear"

**Gestionar trabajadores:**
- Pestaña **Equipo** → buscar trabajador
- Clic en editar wallet → ingresar clave pública Stellar (empieza con G, 56 chars)
- Verificar validación en tiempo real

**Métricas:**
- Pestaña **Métricas** → ver ranking de trabajadores y puntos Harmony

### Flujo Trabajador (Operario)

1. Login como Operario
2. Clic en **Harmony ✦** en el menú
3. Pestaña **Mis Órdenes** → ver historial de picking/packing
4. Pestaña **Mi Rendimiento** → ver puntos y bonos asignados
5. Si hay un período distribuido, ver botón "Reclamar Bono"

---

## 9. Errores comunes

### `HARMONY_DISABLED` (403)
Harmony está desactivado en `harmony_config.activo`.
**Solución:** Activar vía `PATCH /api/harmony/config` con admin.

### `@stellar/stellar-sdk no está instalado`
**Solución:**
```bash
cd back && npm install @stellar/stellar-sdk
```

### `STELLAR_ADMIN_SECRET no configurado`
**Solución:** Agregar la secret key en `.env`.

### `Simulación fallida: contract not found`
El `ORG_CONTRACT_ID` en `.env` o en `harmony_config` es incorrecto.
**Solución:** Verificar que el contrato fue desplegado e inicializado.

### `La wallet GXXX... no existe en la red o no está fondeada`
La wallet del trabajador no tiene cuenta activa en Stellar.
**Solución:** Fondear con al menos 1 XLM (en testnet: usar Friendbot).

### `TX finalizada con error: FAILED`
La transacción fue rechazada por el contrato.
Causas comunes:
- Período no está en el estado correcto (Open → Closed → Distributed)
- El admin no es Owner en el contrato (`Unauthorized`)
- El contrato no tiene USDC para pagar (`InvalidAmount`)

Revisar el resultado en Stellar Expert:
```
https://stellar.expert/explorer/testnet/tx/<tx_hash>
```

### Timeout esperando TX
La red estaba congestionada. El TX puede haber llegado igual.
**Solución:** Buscar el hash en Stellar Expert antes de reintentar.

---

## Checklist de verificación rápida

- [ ] Tablas `harmony_*` existen en PostgreSQL
- [ ] `harmony_config` tiene una fila con `activo = false`
- [ ] Backend arranca sin errores con `STELLAR_STUB=true`
- [ ] `GET /api/config` devuelve `harmonyEnabled: false`
- [ ] Activar Harmony funciona vía API
- [ ] Crear período devuelve `periodo_id`
- [ ] Calcular puntos lee datos reales del WMS
- [ ] Distribuir en stub devuelve `tx_hash: STUB_DIST_xxx`
- [ ] SDK instalado con `npm install @stellar/stellar-sdk`
- [ ] Contratos compilados sin errores (`cargo test` en verde)
- [ ] Factory y Organization desplegados en testnet
- [ ] `STELLAR_STUB=false` + variables configuradas en `.env`
- [ ] Distribución real devuelve tx_hash verificable en Stellar Expert
- [ ] Worker recibe XDR en `reclamarBono` para firmar con Freighter
