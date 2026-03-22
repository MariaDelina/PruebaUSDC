# Despliegue de Contratos Harmony — Soroban / Stellar

## Pre-requisitos

```bash
# Instalar Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Agregar target WebAssembly
rustup target add wasm32-unknown-unknown

# Instalar Stellar CLI
cargo install --locked stellar-cli --features opt
```

## Compilar

```bash
cd contracts

# Compilar ambos contratos (optimizados para producción)
stellar contract build

# Los .wasm quedan en:
# factory/target/wasm32-unknown-unknown/release/harmony_factory.wasm
# organization/target/wasm32-unknown-unknown/release/harmony_organization.wasm
```

## Correr tests

```bash
cd contracts
cargo test
```

## Desplegar en Testnet

```bash
# Configurar red
stellar network add testnet \
  --rpc-url https://soroban-testnet.stellar.org \
  --network-passphrase "Test SDF Network ; September 2015"

# Crear keypair del admin (guarda la secret key de forma segura)
stellar keys generate harmony-admin --network testnet --fund

# 1. Desplegar Factory
stellar contract deploy \
  --wasm factory/target/wasm32-unknown-unknown/release/harmony_factory.wasm \
  --source harmony-admin \
  --network testnet

# Guarda el CONTRACT_ID devuelto como FACTORY_CONTRACT_ID

# 2. Inicializar Factory
stellar contract invoke \
  --id $FACTORY_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- initialize \
  --admin $(stellar keys address harmony-admin)

# 3. Desplegar Organization (una por empresa)
stellar contract deploy \
  --wasm organization/target/wasm32-unknown-unknown/release/harmony_organization.wasm \
  --source harmony-admin \
  --network testnet

# Guarda el CONTRACT_ID como ORG_CONTRACT_ID

# 4. Inicializar Organization
#    USDC testnet: CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA
stellar contract invoke \
  --id $ORG_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- initialize \
  --owner $(stellar keys address harmony-admin) \
  --reward_asset CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA

# 5. Registrar Organization en Factory
stellar contract invoke \
  --id $FACTORY_CONTRACT_ID \
  --source harmony-admin \
  --network testnet \
  -- register_org \
  --org_address $ORG_CONTRACT_ID
```

## Guardar en .env del backend

```env
STELLAR_STUB=false
FACTORY_CONTRACT_ID=<id del factory>
ORG_CONTRACT_ID=<id de la organización>
STELLAR_NETWORK=testnet
STELLAR_ADMIN_SECRET=<secret key del admin — NUNCA en git>
```

## Seguridad en producción

- Nunca commitear la secret key del admin
- Usar Azure Key Vault o similar para guardar la secret key
- Rotar el admin key después del primer deploy
- Auditar el contrato con `stellar contract read` para verificar estado
- En mainnet: siempre verificar el wasm hash antes de inicializar
