# Migration Report: Firefly System Integration Alignment with Embers

## Overview

This report documents all changes made to align `firefly-system-integration` with the working Embers reference implementation. The primary goal was to replace the legacy multi-validator `f1r3node` cluster with the simpler, proven Embers-based Firefly stack.

## Summary of Changes

### Architecture Change
- **Before**: Multi-validator cluster (bootstrap + 3 validators + readonly) with complex ceremony setup
- **After**: Simplified Firefly stack (3 mainnet validators + read replicas + testnet) matching Embers reference

### Key Principle
All changes were made to match the **Embers reference implementation** (`embers/docker/docker-compose.yaml`), which is a known working configuration.

---

## Detailed File Changes

### 1. `docker-compose.yml` - Complete Replacement

**Original Setup:**
- Multi-validator cluster: `boot`, `validator1`, `validator2`, `validator3`, `readonly`
- Used environment variables for keys: `${BOOTSTRAP_PRIVATE_KEY}`, `${VALIDATOR1_PRIVATE_KEY}`, etc.
- Ports: 40400-40435 (spread across validators)
- Required complex genesis ceremony configuration
- Mounted volumes from `./services/f1r3node/docker/...`

**New Setup:**
- Firefly validators: `firefly`, `firefly-2`, `firefly-3` (all mainnet)
- Read replicas: `firefly-read`, `firefly-read-testnet`
- Testnet: `firefly-testnet`
- Ports: 14401-14403 (mainnet), 15401-15403 (testnet), 14413/15413 (read replicas)
- Hardcoded validator key: `6a786ec387aff99fcce1bd6faa35916bfad3686d5c98e90a89f77670f535607c`
- Mounts from `./docker/mainnet/genesis` and `./docker/certs/`
- Added optional `state-sync-*` and `events-*` services (profile-based)

**Key Differences:**
- Removed bootstrap/ceremony complexity
- All validators use the same private key (simpler for dev/testing)
- Direct genesis file mounting (no ceremony needed)
- Added testnet support
- Added sync services for AT Protocol integration

---

### 2. `docker-compose.embers.yml` - Endpoint Updates

**Changes Made:**

1. **Added environment variable defaults** (lines 10-22):
   ```yaml
   environment:
     EMBERS__MAINNET__DEPLOY_SERVICE_URL: ${EMBERS__MAINNET__DEPLOY_SERVICE_URL:-http://firefly:40401}
     EMBERS__MAINNET__PROPOSE_SERVICE_URL: ${EMBERS__MAINNET__PROPOSE_SERVICE_URL:-http://firefly:40402}
     EMBERS__MAINNET__OBSERVER_URL: ${EMBERS__MAINNET__OBSERVER_URL:-http://firefly-read:40403}
     EMBERS__MAINNET__OBSERVER_WS_API_URL: ${EMBERS__MAINNET__OBSERVER_WS_API_URL:-ws://firefly-read:40403}
     EMBERS__MAINNET__VALIDATOR_WS_API_URL: ${EMBERS__MAINNET__VALIDATOR_WS_API_URL:-ws://firefly:40403}
     EMBERS__MAINNET__SERVICE_KEY: ${EMBERS__MAINNET__SERVICE_KEY:-232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE}
     # ... same for TESTNET
   ```

2. **Removed `depends_on` blocks** (originally had dependencies on firefly services):
   - Removed because services are in different compose files
   - Services communicate via external `f1r3fly` network
   - Added comment explaining the relationship

**Rationale:**
- Provides sensible defaults that match Embers reference
- Allows override via `.env.embers` if needed
- Removes compose validation errors when running files independently

---

### 3. `.env.embers` - Hostname and Key Updates

**Changes Made:**

1. **Updated all service URLs** from `rnode.*` to `firefly`:
   - `EMBERS__MAINNET__DEPLOY_SERVICE_URL`: `http://rnode.validator1:40401` → `http://firefly:40401`
   - `EMBERS__MAINNET__PROPOSE_SERVICE_URL`: `http://rnode.validator1:40402` → `http://firefly:40402`
   - `EMBERS__MAINNET__OBSERVER_URL`: `http://rnode.readonly:40403` → `http://firefly-read:40403`
   - `EMBERS__MAINNET__VALIDATOR_WS_API_URL`: `ws://rnode.validator1:40405` → `ws://firefly:40403`
   - Same changes for TESTNET endpoints

2. **Updated SERVICE_KEY**:
   - **OLD**: `0258c0649f7d01140b766a0ea8586181896cc6f2769f11a4ee43bdb06f110658`
   - **NEW**: `232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE`
   - This matches the Embers reference implementation

3. **Updated comment** (line 39):
   - Changed from: `(rnode.validator1, rnode.readonly)`
   - Changed to: `(firefly, firefly-read, firefly-testnet, firefly-read-testnet)`

**Rationale:**
- Aligns with new service names in `docker-compose.yml`
- Uses the wallet key that's funded in the Embers genesis setup
- Ensures Embers can connect to the correct Firefly services

---

### 4. `docker-compose.f1r3sky.yml` - Added Firefly Integration

**Changes Made:**

Added environment variables to `pds` service (lines 129-134):
```yaml
environment:
  DEPLOY_SERVICE_URL: ${DEPLOY_SERVICE_URL:-http://firefly:40401}
  PROPOSE_SERVICE_URL: ${PROPOSE_SERVICE_URL:-http://firefly:40402}
  READ_NODE_URL: ${READ_NODE_URL:-http://firefly-read:40403}
  READ_NODE_WS_URL: ${READ_NODE_WS_URL:-ws://firefly-read:40403}
  DEFAULT_WALLET_KEY: ${DEFAULT_WALLET_KEY:-232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE}
  DEFAULT_WALLET_ADDRESS: ${DEFAULT_WALLET_ADDRESS:-1111EjdAxnKb5zKUc8ikuxfdi3kwSGH7BJCHKWjnVzfAF3SjCBvjh}
```

**Rationale:**
- PDS needs to know how to connect to Firefly for blockchain operations
- These were missing from the original setup
- Matches what Embers demo compose uses

---

### 5. `.env.f1r3sky` - Added Firefly Integration Block

**Changes Made:**

Added new section at end of file (lines 81-87):
```env
# Firefly node integration
DEPLOY_SERVICE_URL=http://firefly:40401
PROPOSE_SERVICE_URL=http://firefly:40402
READ_NODE_URL=http://firefly-read:40403
READ_NODE_WS_URL=ws://firefly-read:40403
DEFAULT_WALLET_KEY=232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE
DEFAULT_WALLET_ADDRESS=1111EjdAxnKb5zKUc8ikuxfdi3kwSGH7BJCHKWjnVzfAF3SjCBvjh
```

**Rationale:**
- Provides default values for PDS to connect to Firefly
- Can be overridden if needed
- Ensures both compose file and env file have same defaults

---

### 6. New Directory Structure: `./docker/`

**Created:**
```
firefly-system-integration/docker/
├── certs/
│   ├── node.key.pem
│   └── node.certificate.pem
├── mainnet/
│   └── genesis/
│       ├── 04b103b9a8225589ce98d8417a3744f712e3b1660169e969297ed822e4edd88c13111726d35172ceb8a48065cfce5917292cbe42c8f48a73965100edb094c73365.sk
│       ├── bonds.txt
│       └── wallets.txt
└── testnet/
    └── genesis/
        ├── 04b103b9a8225589ce98d8417a3744f712e3b1660169e969297ed822e4edd88c13111726d35172ceb8a48065cfce5917292cbe42c8fdi3kwSGH7BJCHKWjnVzfAF3SjCBvjh
        ├── bonds.txt
        └── wallets.txt
```

**Source:**
- All files copied from `embers/docker/` directory
- These are the working genesis files and certificates from the Embers reference

**Rationale:**
- Self-contained setup (no dependency on `services/f1r3node/`)
- Uses proven working genesis configuration
- Matches Embers reference structure

---

### 7. Documentation Updates

#### `COMPOSE_STRUCTURE.md`

**Changes:**
- Updated service list from `boot, validator1-3, readonly` to `firefly, firefly-2, firefly-3, firefly-read, firefly-testnet, firefly-read-testnet`
- Updated description to mention "sourced from Embers reference stack"
- Added mention of optional sync helpers

#### `README.md`

**Changes:**
- Updated all references from `rnode.bootstrap` to `firefly` in log commands
- Updated service list to show new Firefly nodes
- Updated port mappings:
  - **OLD**: `40411:40401` (validator1), `40413:40403` (validator1), `40453:40403` (readonly)
  - **NEW**: `14401-14403` (mainnet), `15401-15403` (testnet), `14413/15413` (read replicas)
- Updated troubleshooting sections to reference `firefly` instead of `rnode.bootstrap`

---

## Key Changes Summary

### Wallet Keys and Service Keys

**Important Note**: The keys were changed to match the Embers reference implementation, which uses a pre-funded wallet in its genesis setup.

1. **Validator Private Key** (in `docker-compose.yml`):
   - **Value**: `6a786ec387aff99fcce1bd6faa35916bfad3686d5c98e90a89f77670f535607c`
   - **Used by**: All Firefly validators (`firefly`, `firefly-2`, `firefly-3`)
   - **Source**: Embers reference implementation

2. **Service/Wallet Key** (in `.env.embers` and `.env.f1r3sky`):
   - **OLD**: `0258c0649f7d01140b766a0ea8586181896cc6f2769f11a4ee43bdb06f110658`
   - **NEW**: `232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE`
   - **Used by**: Embers API and PDS for paying transaction fees
   - **Source**: Embers reference implementation (this wallet is funded in Embers genesis)

3. **Wallet Address** (in `.env.f1r3sky`):
   - **Value**: `1111EjdAxnKb5zKUc8ikuxfdi3kwSGH7BJCHKWjnVzfAF3SjCBvjh`
   - **Source**: Embers reference implementation

**Why These Keys Changed:**
- The original keys were from the multi-validator setup
- The new keys are from the Embers reference, which has a working genesis with funded wallets
- This ensures transactions can actually execute (the wallet has funds)

---

## Service Endpoint Changes

### Before (rnode cluster):
- Mainnet Deploy: `http://rnode.validator1:40401`
- Mainnet Propose: `http://rnode.validator1:40402`
- Mainnet Observer: `http://rnode.readonly:40403`
- Mainnet WS: `ws://rnode.validator1:40405` or `ws://rnode.readonly:40405`

### After (Firefly stack):
- Mainnet Deploy: `http://firefly:40401`
- Mainnet Propose: `http://firefly:40402`
- Mainnet Observer: `http://firefly-read:40403`
- Mainnet WS: `ws://firefly:40403` or `ws://firefly-read:40403`
- Testnet Deploy: `http://firefly-testnet:40401`
- Testnet Propose: `http://firefly-testnet:40402`
- Testnet Observer: `http://firefly-read-testnet:40403`

---

## Port Mapping Changes

### Before:
- Bootstrap: 40400-40405
- Validator1: 40410-40415
- Validator2: 40420-40425
- Validator3: 40430-40435
- Readonly: 40451-40453

### After:
- Firefly (mainnet): 14401-14403
- Firefly-2 (mainnet): 14421-14424
- Firefly-3 (mainnet): 14431-14434
- Firefly-read (mainnet): 14413
- Firefly-testnet: 15401-15403
- Firefly-read-testnet: 15413

---

## What Was NOT Changed

1. **F1R3Sky services** (`docker-compose.f1r3sky.yml`):
   - All AT Protocol services (bsky, pds, bsync, etc.) remain unchanged
   - Only added Firefly connection parameters

2. **Embers Frontend** (`docker-compose.embers.yml`):
   - Build configuration unchanged
   - Port mappings unchanged
   - Only API endpoint references updated

3. **Network configuration**:
   - Still uses `f1r3fly` network (external)
   - Network structure unchanged

---

## Migration Impact

### Breaking Changes:
1. **Service names changed**: Any scripts/tools referencing `rnode.*` need updating
2. **Port mappings changed**: External access ports are different
3. **Keys changed**: If you had funds in the old wallet keys, they won't work with new setup
4. **Genesis files**: New genesis setup (copied from Embers)

### Non-Breaking:
1. **Network**: Same `f1r3fly` network
2. **Service functionality**: Same capabilities, just different topology
3. **Environment variables**: Most can be overridden via `.env` files

---

## Validation Steps

To verify the migration:

1. **Check service connectivity**:
   ```bash
   docker compose -f docker-compose.yml ps
   # Should show firefly, firefly-2, firefly-3, firefly-read, etc.
   ```

2. **Verify Embers can connect**:
   ```bash
   curl http://localhost:8080/api/service/ready
   ```

3. **Check Firefly endpoints**:
   ```bash
   curl http://localhost:14403/api/blocks
   ```

4. **Verify keys are funded** (if genesis was set up correctly):
   - The wallet `232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE` should have funds in genesis

---

## Questions & Answers

### Q: Why did you change the wallet keys?
**A**: The new keys (`232DADA5BBAFC0799D5F370DA04AF70CE438F69F954512B26D6FB5B560B81DFE`) are from the Embers reference implementation, which has a working genesis setup with funded wallets. The old keys may not have had funds in the genesis, which would cause transaction failures.

### Q: Why use the same validator key for all validators?
**A**: This matches the Embers reference implementation. For development/testing, this simplifies setup. In production, you'd want unique keys per validator.

### Q: Why remove the bootstrap/ceremony setup?
**A**: The Embers reference uses a simpler direct genesis approach that doesn't require ceremony. This is easier to set up and maintain.

### Q: Can I use my old keys?
**A**: Yes, but you'd need to:
1. Update the validator keys in `docker-compose.yml`
2. Update the service keys in `.env.embers` and `.env.f1r3sky`
3. Ensure those wallets are funded in your genesis setup

### Q: What if I had funds in the old wallet?
**A**: You'd need to either:
1. Keep using the old keys (update the compose/env files)
2. Transfer funds from old wallet to new wallet
3. Update genesis to fund the new wallet

---

## Files Modified

1. `docker-compose.yml` - Complete replacement
2. `docker-compose.embers.yml` - Added env defaults, removed depends_on
3. `docker-compose.f1r3sky.yml` - Added Firefly env vars to PDS
4. `.env.embers` - Updated hostnames and SERVICE_KEY
5. `.env.f1r3sky` - Added Firefly integration block
6. `COMPOSE_STRUCTURE.md` - Updated service descriptions
7. `README.md` - Updated commands and port references

## Files Created

1. `docker/certs/node.key.pem`
2. `docker/certs/node.certificate.pem`
3. `docker/mainnet/genesis/*` (3 files)
4. `docker/testnet/genesis/*` (3 files)

---

## References

- **Embers Reference**: `embers/docker/docker-compose.yaml`
- **Embers Demo**: `embers/docker/demo-docker-compose.yaml`
- **Original Backup**: `backups/system-integration/docker-compose.yml`

---

## Date

Migration completed: December 9, 2025
