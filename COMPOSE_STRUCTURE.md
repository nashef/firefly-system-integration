# Docker Compose Structure

This repository uses a modular compose file structure, with services separated by logical stacks.

## Files

### docker-compose.yml (Firefly Stack)
**Purpose:** Core blockchain infrastructure sourced from the Embers reference stack
**Services:**
- `firefly`, `firefly-2`, `firefly-3` (mainnet validators)
- `firefly-read` (mainnet read replica)
- `firefly-testnet`, `firefly-read-testnet` (testnet validator + read replica)
- `state-sync-*` helpers (optional profile for ledger uploads)
- `events-*` helpers (optional profile for AT Protocol bridge)

**Creates:** `f1r3fly` network and mounts shared assets under `./docker/`

### docker-compose.f1r3sky.yml (AT Protocol Stack)  
**Purpose:** AT Protocol social media services
**Services:**
- postgres, redis (infrastructure)
- bsky, pds, bsync, ozone (AT Protocol services)
- f1r3sky (frontend web app)

**Requires:** `f1r3fly` network (external)
**Volumes:** postgres_data, redis_data, pds_blocks, pds_tmp, pds_data

### docker-compose.embers.yml (Embers Stack)
**Purpose:** Blockchain API bridge and UI
**Services:**
- embers-api (Rust API bridging f1r3sky to f1r3node)
- embers-frontend (React 19 web UI)

**Requires:** `f1r3fly` network (external)

## Usage

```bash
# Start everything (via shardctl)
poetry run shardctl up

# Start specific stacks
docker-compose up                                          # Just blockchain
docker-compose -f docker-compose.f1r3sky.yml up           # Just AT Protocol
docker-compose -f docker-compose.embers.yml up            # Just Embers

# Combine stacks manually
docker-compose -f docker-compose.yml -f docker-compose.embers.yml up
```

## Network
All services communicate via the `f1r3fly` bridge network, created by docker-compose.yml.
