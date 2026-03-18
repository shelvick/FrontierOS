# Sigil — Complete Project Brief

> This document contains everything a new Claude session needs to pick up this project
> and run with it. No other context required.

## What Is This?

**Sigil** is a diplomacy-centered infrastructure control and mapping tool for EVE Frontier,
built as an entry for the EVE Frontier x Sui Hackathon 2026.

- **Hackathon**: March 11-31, 2026. $80,000 prize pool. Theme: "A Toolkit for Civilization"
- **Submission**: GitHub repo + DeepSurge registration. Submissions close March 31 11:59pm UTC.
- **Judging**: 75% expert panel (concept, feasibility, design, implementation, player utility,
  creativity, UX, visual presentation) + 25% community vote (April 1-15).
- **Bonus points**: Deploy to live Stillness server within 14 days post-hackathon.
- **Full rules**: https://evefrontier.com/en/eve-froniter-hackathon-event-rules
- **Registration**: https://deepsurge.xyz/evefrontier2026 (registered)

**Prize categories we're targeting:**
- Overall Winners (1st: $15k, 2nd: $7.5k, 3rd: $5k) — drawn from any category
- Utility ($5k) — "mods that materially change how players survive, coordinate, explore, or compete"
- Technical Implementation ($5k) — "clean architecture, smart use of Frontier systems, scalability"
- Live Frontier Integration ($5k) — our Move extensions deploy to the game world and players
  interact with them directly. This makes us eligible despite being an "external tool" — the
  smart contracts ARE in-game code.

## EVE Frontier — Game Context

EVE Frontier is a blockchain-based hardcore space survival MMO by CCP Games (makers of EVE Online).
Currently in early alpha, running in 3-month "Cycles" that reset the universe.

**Core gameplay loop**: Mine → Refine → Craft → Build → Survive → Expand. Everything costs fuel.
Cycle 5 introduces Orbital Zones (server-side PvE ecosystems replacing dungeons) that distribute
resources, NPCs ("Feral AI"), and loot across solar systems.

**Smart Assemblies** are programmable in-game structures players deploy in space:
- **Network Node** — power source, provides fuel/energy to connected assemblies
- **Smart Gate** — stargate for fast travel, configurable access rules
- **Smart Turret** — automated defense, configurable friend/foe targeting (3 types: Autocannon, Plasma, Railgun)
- **Smart Storage Unit** — storage with inventory management, can act as trading post
- **Nursery** — manufacturing facility for Shells (clone bodies) — new in Cycle 5
- **Nest** — storage facility for Shells — new in Cycle 5
- **Smart Hangar** — ship storage

**Cycle 5 new gameplay — Shell Industry**: Players manufacture clone bodies ("Shells") and equip
them with "Crowns" (skill/memory constructs). Shell destruction = permadeath for that clone's
capabilities. Shell manufacturing, Crowns, and combat mechanics are server-side — NOT on-chain.
On-chain artifacts: Nursery is likely a base `Assembly` (no extension support), Nest is likely a
`StorageUnit` variant (HAS extension support — we could write tribe-access controls for Shell storage).

**Tribes** are player organizations (like corps/alliances in EVE Online). ~45 exist currently.

**The biggest unsolved player problem**: Tribe coordination. No tool exists for shared resource
tracking, assembly management, logistics coordination, or diplomacy. EF-Map (ef-map.com) dominates
the solo-player tool space (maps, blueprint calculator, killboard) with 55k lines of code. Don't
compete with it — solve the tribe problem instead.

**Cycle 5 "Shroud of Fear" + Sui migration**: Cycle 5 launches **March 11, 2026** — the same
day as the hackathon. This is the Sui migration launch: on-chain assets move from EVM to Sui
on day 1. Package IDs should be available immediately. The Sui deployment uses **standard Sui
Testnet** (`fullnode.testnet.sui.io:443`). Docker-based local world setup is also available
via builder-scaffold for offline development.

## Tech Stack

- **Sui Move** — on-chain smart contracts (gate/turret extensions, standings enforcement)
- **Elixir / Phoenix / LiveView** — real-time server-rendered UI, zero JS framework
- **Req** — HTTP client for Sui GraphQL queries
- **Pure Elixir BCS + Ed25519** — transaction encoding and signing (no Node.js sidecar)
- **ETS** — primary data layer for cached blockchain state (assemblies, characters, standings, gates)
- **Postgres** — deferred to Slice 3 for user-generated data (alert storage, acknowledgment lifecycle). Not needed for Slices 1-2.
- **OTP** — GenServer-per-assembly monitoring, DynamicSupervisor, PubSub fan-out
- **Fly.io** — deployment (free tier) — deferred until first vertical slice is complete (~Day 5-6)
- **Full TDD workflow** throughout (Elixir: ExUnit, Move: `sui move test`)

## Why Elixir Is the Right Choice

This isn't just "I know Elixir." The architecture genuinely benefits:

| Capability | Why Elixir Wins |
|---|---|
| GenServer per assembly | Monitor 1000+ structures concurrently, each with own state/lifecycle |
| OTP supervision | Self-healing — if a monitor crashes, it restarts without losing the system |
| Phoenix PubSub | State change in any assembly → instant push to all connected tribe members |
| LiveView | Real-time dashboard with zero JS. Server pushes HTML diffs over WebSocket |
| ETS | Sub-microsecond reads for cached game state |
| Broadway | Back-pressured pipeline for blockchain event ingestion at scale |
| Fault tolerance | Flaky RPC endpoint? Circuit breaker, retry, keep running. Other entries crash. |

---

## Sui Blockchain — What You Need to Know

### Data Access Pattern

There is **no REST API** on Sui. All data access is via GraphQL:

```
Your Elixir App
    → HTTP POST to Sui GraphQL endpoint
    → Query on-chain Move objects by type
    → Get JSON back
```

### Endpoints

| Purpose | URL |
|---|---|
| GraphQL (testnet) | `https://sui-testnet.mystenlabs.com/graphql` |
| GraphQL (mainnet) | `https://sui-mainnet.mystenlabs.com/graphql` |
| JSON-RPC (testnet, deprecated July 2026) | `https://fullnode.testnet.sui.io:443` |

EVE Frontier uses **standard Sui Testnet**. Move.toml dependencies reference `testnet-v1.66.2`.
Confirmed: no custom fork or separate network.

### Sui CLI Installation

Recommended: **`suiup`** version manager (latest: testnet-v1.67.1, March 3 2026):
```bash
cargo install --git https://github.com/Mystenlabs/suiup.git --locked
suiup install sui@testnet
```

Alternatives: pre-built binaries from GitHub releases, or `cargo install --locked --git
https://github.com/MystenLabs/sui.git --branch testnet sui --features tracing`.

Linux prerequisites: `curl git cmake gcc libssl-dev pkg-config libclang-dev libpq-dev build-essential`.

### Rate Limits

~100 requests per 30 seconds on public Sui endpoints. For more:
- Chainstack — 3M free requests
- Ankr — free tier at `https://www.ankr.com/rpc/sui/`
- PublicNode — `https://sui-rpc.publicnode.com`

### No Local Blockchain Needed

Zero storage required. Everything via public HTTP endpoints. Our 10-20GB disk is fine.

### Real-Time Data

No push mechanism exists on Sui — everything is poll-based:
- **GraphQL subscriptions**: Not yet available on Sui
- **WebSocket subscriptions**: Deprecated (July 2024)
- **gRPC streaming**: Available but requires generating Elixir client from proto files

**Two polling approaches (both needed):**

| | State Polling | Event Polling |
|---|---|---|
| **What** | "Give me this object's current fields" | "Give me all events of type X since timestamp Y" |
| **Tells you** | What IS (current fuel, online/offline, standings) | What HAPPENED (who changed what, denied jumps, failed txs) |
| **Cost** | One query per object per cycle | One query catches all events of a type across all objects |
| **Best for** | Monitoring a few objects for many types of changes | Monitoring many objects for specific change types |
| **GenServer fit** | Natural — one GenServer per assembly, each polls its own state | Separate poller(s) for event streams, fan out via PubSub |

**Key insight**: Some things only exist in events, not state. A denied gate jump doesn't
change any on-chain state — it's a failed transaction. You only know about it from the
transaction log. Players will want to know when someone tries and fails to jump through
their gate. This requires event polling.

Events are emitted by Move functions via `event::emit<T>(data)` and recorded in transaction
effects. Query them via GraphQL with cursor-based pagination:

```graphql
query EventsByType($eventType: String!, $first: Int, $after: String) {
  events(filter: { eventType: $eventType }, first: $first, after: $after) {
    nodes {
      type { repr }
      sender { address }
      contents { json }
      timestamp
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

Store `endCursor` between polls — pass as `after` to get only new events since last check.
Filter supports `eventType` (fully qualified type), `sender`, `afterCheckpoint`, `beforeCheckpoint`.
Cannot combine `module` and `eventType` in the same filter.

#### Complete Event Catalog (27 event types from world-contracts)

**Assembly lifecycle** (all assembly types):
- `StatusChangedEvent` — online/offline/anchor/unanchor. Fields: `assembly_id`, `assembly_key`, `status`, `action`
- `MetadataChangedEvent` — name/desc/url changes. Fields: `assembly_id`, `assembly_key`, `name`, `description`, `url`
- `ExtensionAuthorizedEvent` — extension registered on gate/turret/storage. Fields: `assembly_id`, `assembly_key`, `extension_type`, `previous_extension`, `owner_cap_id`

**Gate-specific**:
- `GateCreatedEvent` — gate anchored. Fields: `assembly_id`, `assembly_key`, `owner_cap_id`, `type_id`, `location_hash`, `status`
- `GateLinkedEvent` / `GateUnlinkedEvent` — gate pair linked/unlinked. Fields: `source_gate_id/key`, `destination_gate_id/key`
- `JumpEvent` — successful gate jump. Fields: `source_gate_id/key`, `destination_gate_id/key`, `character_id/key`

**Turret-specific**:
- `TurretCreatedEvent` — turret anchored. Fields: `turret_id`, `turret_key`, `owner_cap_id`, `type_id`
- `PriorityListUpdatedEvent` — targeting list updated. Fields: `turret_id`, `priority_list`

**Energy/Fuel** (NetworkNode):
- `NetworkNodeCreatedEvent` — node anchored. Fields: `network_node_id`, capacity, burn rate, energy
- `FuelEvent` — deposit/withdraw/burning. Fields: `assembly_id`, `type_id`, `old_quantity`, `new_quantity`, `is_burning`, `action`
- `StartEnergyProductionEvent` / `StopEnergyProductionEvent` — energy state changes
- `EnergyReservedEvent` / `EnergyReleasedEvent` — assembly connects/disconnects

**Storage**:
- `StorageUnitCreatedEvent` — storage anchored
- `ItemMintedEvent` / `ItemBurnedEvent` / `ItemDepositedEvent` / `ItemWithdrawnEvent` / `ItemDestroyedEvent`

**Other**:
- `CharacterCreatedEvent` — new player registered
- `KillmailCreatedEvent` — kill recorded. Fields: killer/victim IDs, `loss_type`, `kill_timestamp`, `solar_system_id`
- `OwnerCapCreatedEvent` / `OwnerCapTransferred` — capability lifecycle
- `FuelEfficiencySetEvent` / `FuelEfficiencyRemovedEvent` — config changes

**Key for monitoring (Slice 3)**: `StatusChangedEvent`, `FuelEvent`, `ExtensionAuthorizedEvent`,
`JumpEvent`, `KillmailCreatedEvent`. Note: denied gate jumps are **failed transactions**, not
events — they won't appear in the event stream.

### Elixir SDK Situation

No usable Sui SDK exists for Elixir. The existing packages are abandoned/unusable:
- `ex_sui` — 79 downloads, README says "DO NOT USE"
- `web3_sui_ex` — abandoned since 2023
- `web3_move_ex` — abandoned since 2023, uses deprecated JSON-RPC

**Our approach**: Raw `Req.post!/2` with GraphQL query strings. Simple, reliable, no dependencies.

---

## On-Chain Data Model (from Move Source)

Source: https://github.com/evefrontier/world-contracts (`contracts/world/sources/`)

All objects queryable via Sui GraphQL with type filter `{WORLD_PACKAGE_ID}::module::Struct`.

### Queryable Objects (Read)

```
Assembly          assembly::Assembly
  - id: UID
  - key: TenantItemId
  - owner_cap_id: ID
  - type_id: u64
  - status: AssemblyStatus
  - location: Location
  - energy_source_id: Option<ID>
  - metadata: Option<Metadata>

Gate              gate::Gate
  - id: UID
  - key: TenantItemId
  - owner_cap_id: ID
  - type_id: u64
  - linked_gate_id: Option<ID>
  - status: AssemblyStatus
  - location: Location
  - energy_source_id: Option<ID>
  - metadata: Option<Metadata>
  - extension: Option<TypeName>

Turret            turret::Turret
  - id: UID
  - key: TenantItemId
  - owner_cap_id: ID
  - type_id: u64
  - status: AssemblyStatus
  - location: Location
  - energy_source_id: Option<ID>
  - metadata: Option<Metadata>
  - extension: Option<TypeName>

NetworkNode       network_node::NetworkNode
  - id: UID
  - key: TenantItemId
  - owner_cap_id: ID
  - type_id: u64
  - status: AssemblyStatus
  - location: Location
  - fuel: Fuel
  - energy_source: EnergySource
  - metadata: Option<Metadata>
  - connected_assembly_ids: vector<ID>

Character         character::Character
  - id: UID
  - key: TenantItemId
  - tribe_id: u32
  - character_address: address
  - metadata: Option<Metadata>
  - owner_cap_id: ID

StorageUnit       storage_unit::StorageUnit
  - id: UID
  - key: TenantItemId
  - owner_cap_id: ID
  - type_id: u64
  - status: AssemblyStatus
  - location: Location
  - inventory_keys: vector<ID>      (dynamic field keys — main inventory uses owner_cap_id)
  - energy_source_id: Option<ID>
  - metadata: Option<Metadata>
  - extension: Option<TypeName>

Killmail          killmail::Killmail
  - id: UID
  - killmail_id: TenantItemId
  - killer_character_id: TenantItemId
  - victim_character_id: TenantItemId
  - kill_timestamp: u64
  - loss_type: LossType
  - solar_system_id: TenantItemId

JumpPermit        gate::JumpPermit
  - id: UID
  - character_id: ID
  - route_hash: vector<u8>
  - expires_at_timestamp_ms: u64

ObjectRegistry    object_registry::ObjectRegistry
  - Shared singleton mapping game item IDs → Sui object IDs
```

### Primitive Type Definitions (Inner Structs)

```
Location          location::Location
  - location_hash: vector<u8>           (32-byte Poseidon2 hash — NOT raw x/y/z)

Metadata          metadata::Metadata
  - assembly_id: ID
  - name: String
  - description: String
  - url: String

Fuel              fuel::Fuel
  - max_capacity: u64
  - burn_rate_in_ms: u64
  - type_id: Option<u64>
  - unit_volume: Option<u64>
  - quantity: u64
  - is_burning: bool
  - previous_cycle_elapsed_time: u64
  - burn_start_time: u64
  - last_updated: u64

EnergySource      energy::EnergySource
  - max_energy_production: u64
  - current_energy_production: u64
  - total_reserved_energy: u64

AssemblyStatus    status::AssemblyStatus
  - status: Status                       (enum: NULL | OFFLINE | ONLINE)

TenantItemId      in_game_id::TenantItemId
  - item_id: u64
  - tenant: String
```

### Example GraphQL Queries

**Query all Smart Gates:**
```graphql
query AllGates($cursor: String) {
  objects(
    filter: { type: "<WORLD_PACKAGE_ID>::gate::Gate" }
    first: 50
    after: $cursor
  ) {
    pageInfo { hasNextPage endCursor }
    nodes {
      address
      asMoveObject { contents { json type { repr } } }
    }
  }
}
```

**Query specific object by ID:**
```graphql
query GetObject($id: SuiAddress!) {
  object(address: $id) {
    address
    version
    digest
    asMoveObject { contents { json type { repr } } }
  }
}
```

**Query objects owned by address (for OwnerCap lookup):**
```graphql
query OwnedObjects($owner: SuiAddress!, $type: String) {
  objects(filter: { owner: $owner, type: $type }) {
    nodes {
      address
      asMoveObject { contents { json } }
    }
  }
}
```

**Elixir pattern:**
```elixir
defmodule Sigil.Sui.Client do
  @graphql_url "https://sui-testnet.mystenlabs.com/graphql"

  def query(query_string, variables \\ %{}) do
    case Req.post(@graphql_url, json: %{query: query_string, variables: variables}) do
      {:ok, %{status: 200, body: %{"data" => data}}} -> {:ok, data}
      {:ok, %{body: %{"errors" => errors}}} -> {:error, errors}
      {:error, reason} -> {:error, reason}
    end
  end
end
```

### Writable Operations — Complete API Surface (from Move Source)

Every function a player or dApp can call on-chain, organized by authorization level.
All from `contracts/world/sources/` unless noted. Source: https://github.com/evefrontier/world-contracts

#### TIER 1: Player Wallet Only (OwnerCap)

No gas pool needed. Player signs with their own key. **These are Sigil's core operations.**

**Gate (`world::gate`)**

| Function | Arguments | What it does |
|---|---|---|
| `authorize_extension<Auth>` | `gate: &mut Gate, owner_cap: &OwnerCap<Gate>` | Registers your custom Move extension on this gate. Once set, the gate's default `jump()` is blocked — travelers must get a JumpPermit from your extension. Can be called again to swap to a different extension. |
| `online` | `gate: &mut Gate, network_node: &mut NetworkNode, energy_config: &EnergyConfig, owner_cap: &OwnerCap<Gate>` | Powers on the gate. Reserves energy from the connected network node. Gate must have an energy source set. Both gates in a pair must be online for jumps to work. |
| `offline` | `gate: &mut Gate, network_node: &mut NetworkNode, energy_config: &EnergyConfig, owner_cap: &OwnerCap<Gate>` | Powers off the gate. Releases reserved energy back to the network node. While offline, no jumps possible. |
| `unlink_gates` | `source_gate: &mut Gate, destination_gate: &mut Gate, source_owner_cap: &OwnerCap<Gate>, destination_owner_cap: &OwnerCap<Gate>` | Breaks the link between a gate pair. Requires OwnerCap for BOTH gates (i.e., you must own both). After unlinking, neither gate can be used for jumps until re-linked. |
| `update_metadata_name` | `gate: &mut Gate, owner_cap: &OwnerCap<Gate>, name: String` | Changes the gate's display name (visible in-game and on-chain). |
| `update_metadata_description` | `gate: &mut Gate, owner_cap: &OwnerCap<Gate>, description: String` | Changes the gate's description text. |
| `update_metadata_url` | `gate: &mut Gate, owner_cap: &OwnerCap<Gate>, url: String` | Changes the gate's associated URL (e.g., link to your tool). |

**Turret (`world::turret`)**

| Function | Arguments | What it does |
|---|---|---|
| `authorize_extension<Auth>` | `turret: &mut Turret, owner_cap: &OwnerCap<Turret>` | Registers your custom targeting logic on this turret. The game server will call YOUR package's `get_target_priority_list` instead of the default. |
| `online` | `turret: &mut Turret, network_node: &mut NetworkNode, energy_config: &EnergyConfig, owner_cap: &OwnerCap<Turret>` | Powers on the turret. Reserves energy. Turret begins defending when online. |
| `offline` | `turret: &mut Turret, network_node: &mut NetworkNode, energy_config: &EnergyConfig, owner_cap: &OwnerCap<Turret>` | Powers off the turret. Releases energy. Turret stops shooting. |
| `update_metadata_*` | Same pattern as Gate | Changes name/description/url. |

**Storage Unit (`world::storage_unit`)**

| Function | Arguments | What it does |
|---|---|---|
| `authorize_extension<Auth>` | `storage_unit: &mut StorageUnit, owner_cap: &OwnerCap<StorageUnit>` | Registers your custom access control on this storage unit. Extension-mediated deposit/withdraw replaces direct access to the main inventory. |
| `online` | `storage_unit: &mut StorageUnit, network_node: &mut NetworkNode, energy_config: &EnergyConfig, owner_cap: &OwnerCap<StorageUnit>` | Powers on. Must be online for any inventory operations. |
| `offline` | Same pattern | Powers off. All inventory operations blocked while offline. |
| `deposit_by_owner<T>` | `storage_unit: &mut StorageUnit, item: Item, character: &Character, owner_cap: &OwnerCap<T>, ctx: &mut TxContext` | Deposits an item into your personal inventory slot within this storage unit. Sender must match `character.character_address`. T can be `StorageUnit` or `Character` OwnerCap. |
| `withdraw_by_owner<T>` | `storage_unit: &mut StorageUnit, character: &Character, owner_cap: &OwnerCap<T>, type_id: u64, quantity: u32, ctx: &mut TxContext` → `Item` | Withdraws items from your personal inventory slot. Returns an `Item` object (in-transit form). Specify item type and how many. |
| `update_metadata_*` | Same pattern as Gate | Changes name/description/url. |

**Assembly — generic (`world::assembly`) — covers Nursery, Nest (non-extensible), Hangar**

| Function | Arguments | What it does |
|---|---|---|
| `online` | `assembly: &mut Assembly, network_node: &mut NetworkNode, energy_config: &EnergyConfig, owner_cap: &OwnerCap<Assembly>` | Powers on. No `authorize_extension` — generic assemblies are NOT extensible. |
| `offline` | Same pattern | Powers off. |
| `update_metadata_*` | Same pattern | Changes name/description/url. |

**Network Node (`world::network_node`)**

| Function | Arguments | What it does |
|---|---|---|
| `online` | `nwn: &mut NetworkNode, owner_cap: &OwnerCap<NetworkNode>, clock: &Clock` | Powers on the node. Starts burning fuel and producing energy. All connected assemblies can then go online. |
| `offline` | `nwn: &mut NetworkNode, fuel_config: &FuelConfig, owner_cap: &OwnerCap<NetworkNode>, clock: &Clock` → `OfflineAssemblies` | Powers off the node. Returns a hot potato — you MUST then call `offline_connected_gate/turret/storage_unit/assembly` for each connected assembly in the same PTB, then `destroy_offline_assemblies`. |
| `update_metadata_*` | Same pattern | Changes name/description/url. |

**Character (`world::character`)**

| Function | Arguments | What it does |
|---|---|---|
| `update_metadata_*` | Same pattern (uses `OwnerCap<Character>`) | Changes character name/description/url. |
| `borrow_owner_cap<T>` | `character: &mut Character, owner_cap_ticket: Receiving<OwnerCap<T>>, ctx: &TxContext` → `(OwnerCap<T>, ReturnOwnerCapReceipt)` | Borrows an OwnerCap that was transferred to this character's object. Returns the cap + a hot potato receipt that forces you to return it in the same PTB. This is how you get access to OwnerCaps for gates/turrets/etc. |
| `return_owner_cap<T>` | `character: &Character, owner_cap: OwnerCap<T>, receipt: ReturnOwnerCapReceipt` | Returns a borrowed OwnerCap. Consumes the hot potato receipt. Must be called in the same PTB as borrow. |

#### TIER 2: Extension Witness (Our Move Contracts)

These are functions WE implement in our deployed packages. Authorized by typed witness pattern.

**Gate extension — `frontier_gate` (called by travelers)**

| Function | Arguments | What it does |
|---|---|---|
| `request_permit` | `standings_table: &StandingsTable, source_gate: &Gate, destination_gate: &Gate, character: &Character, clock: &Clock, ctx: &mut TxContext` | A traveler calls this to request permission to jump. Reads the traveler's tribe from their Character, looks up their standing in our StandingsTable. If friendly/neutral: calls `gate::issue_jump_permit<FrontierGateAuth>()` to create a JumpPermit and send it to the traveler. If hostile: asserts and fails (no permit, can't jump). |

The world contract's underlying function our extension calls:

| Function | Arguments | What it does |
|---|---|---|
| `gate::issue_jump_permit<Auth>` | `source_gate: &Gate, destination_gate: &Gate, character: &Character, _: Auth, expires_at_timestamp_ms: u64, ctx: &mut TxContext` | Creates a `JumpPermit` object with a bidirectional route hash and expiry timestamp. Transfers it to the character's wallet address. Both gates must have the same extension registered. Only callable by the registered extension (witness pattern enforces this). |

**Turret extension — `frontier_turret` (called by game server)**

| Function | Arguments | What it does |
|---|---|---|
| `get_target_priority_list` | `turret: &Turret, owner_character: &Character, target_candidate_list: vector<u8>, receipt: OnlineReceipt` → `vector<u8>` | Game server calls this when ships enter turret range. Receives BCS-encoded list of targets (with tribe, HP, aggressor status, ship type). We decode it, check each target's tribe against our StandingsTable, and return a priority-weighted list: hostile ships get high weight (shoot first), friendly/neutral get zero (ignore). Must consume the OnlineReceipt hot potato. |

Helper functions available to our extension:

| Function | Arguments | What it does |
|---|---|---|
| `turret::verify_online` | `turret: &Turret` → `OnlineReceipt` | Creates a proof that the turret is online. Returns hot potato. |
| `turret::destroy_online_receipt<Auth>` | `receipt: OnlineReceipt, _: Auth` | Consumes the hot potato. Only callable by registered extension. |
| `turret::unpack_candidate_list` | `candidate_list_bytes: vector<u8>` → `vector<TargetCandidate>` | Decodes BCS bytes into typed target list. Fields per target: item_id, type_id, group_id, character_id (u32), character_tribe (u32), hp_ratio, shield_ratio, armor_ratio, is_aggressor (bool), priority_weight, behaviour_change (enum). |
| `turret::new_return_target_priority_list` | `target_item_id: u64, priority_weight: u64` → `ReturnTargetPriorityList` | Constructs a return entry. Higher weight = shoot first. |

**Storage Unit extension — potential `frontier_storage` (for Nest tribe access)**

| Function | Arguments | What it does |
|---|---|---|
| `deposit_item<Auth>` | `storage_unit: &mut StorageUnit, character: &Character, item: Item, _: Auth, ctx: &mut TxContext` | Extension-controlled deposit into the main inventory. Your extension decides who can deposit what. Item must have been withdrawn from THIS storage unit (parent_id check). |
| `withdraw_item<Auth>` | `storage_unit: &mut StorageUnit, character: &Character, _: Auth, type_id: u64, quantity: u32, ctx: &mut TxContext` → `Item` | Extension-controlled withdrawal. Your extension decides who can take what. Returns an Item (in-transit). |
| `deposit_to_owned<Auth>` | `storage_unit: &mut StorageUnit, character: &Character, item: Item, _: Auth, ctx: &mut TxContext` | Deposits into a SPECIFIC PLAYER's personal inventory slot — the target player does NOT need to be the transaction sender. Creates the inventory slot if it doesn't exist. Enables: guild hangars, async trading, reward distribution. |

**Our StandingsTable (shared object — functions we define)**

| Function | Arguments | What it does |
|---|---|---|
| `create` | `ctx: &mut TxContext` → `StandingsTable` | Creates a new standings table owned by the caller. Contains `Table<u32, u8>` mapping tribe_id → standing. |
| `set_standing` | `table: &mut StandingsTable, tribe_id: u32, standing: u8, ctx: &TxContext` | Sets one tribe's standing. 0=hostile, 1=neutral, 2=friendly. Owner-only. |
| `get_standing` | `table: &StandingsTable, tribe_id: u32` → `u8` | Reads a tribe's standing. Returns 1 (neutral) if tribe not in table. Public read — anyone can call. |
| `batch_set_standings` | `table: &mut StandingsTable, ids: vector<u32>, standings: vector<u8>, ctx: &TxContext` | Sets multiple tribes' standings in one call. Owner-only. |

#### TIER 3: Gas Pool Co-sign (AdminACL)

Requires CCP's gas pool to co-sign. On testnet, gas is free from faucet but the AdminACL
check still applies — you need the gas pool's address in the `authorized_sponsors` table.

**Gate**

| Function | Key extra arguments | What it does |
|---|---|---|
| `jump` | `source_gate, destination_gate, character, admin_acl, ctx` | Default jump (no extension). Blocked if any extension is configured. Emits JumpEvent. |
| `jump_with_permit` | `source_gate, destination_gate, character, jump_permit: JumpPermit, admin_acl, clock, ctx` | Jump using a permit from an extension. Validates permit (not expired, correct route, correct character), consumes it (single-use), then jumps. Emits JumpEvent. |
| `link_gates` | `source_gate, destination_gate, gate_config, server_registry, admin_acl, source_owner_cap, dest_owner_cap, distance_proof: vector<u8>, clock, ctx` | Links two gates into a pair for travel. Requires: both OwnerCaps, same type_id, server-signed distance proof (gates must be within max range), neither already linked. |
| `anchor` | `registry, network_node, character, admin_acl, item_id: u64, type_id: u64, location_hash: vector<u8>, ctx` → `Gate` | Creates a new gate object in space. Must then call `share_gate()`. Server controls placement. |
| `share_gate` | `gate: Gate, admin_acl, ctx` | Makes the gate a shared object (accessible by anyone). Called right after anchor. |
| `unanchor` | `gate: Gate, network_node, energy_config, admin_acl, ctx` | Destroys a gate. Releases energy, disconnects from network node, deletes the object. Gate must not be linked. |
| `update_energy_source` | `gate, network_node, admin_acl, ctx` | Reassigns the gate to a different network node. Gate must be offline. |
| `set_max_distance` | `gate_config, admin_acl, type_id: u64, max_distance: u64, ctx` | Sets the maximum link distance for a gate type. Server config. |

**Turret**

| Function | Key extra arguments | What it does |
|---|---|---|
| `anchor` | `registry, network_node, character, admin_acl, item_id, type_id, location_hash, ctx` → `Turret` | Creates a new turret. Then call `share_turret()`. |
| `share_turret` | `turret: Turret, admin_acl, ctx` | Makes the turret a shared object. |
| `unanchor` | `turret: Turret, network_node, energy_config, admin_acl, ctx` | Destroys a turret. |
| `update_energy_source` | `turret, network_node, admin_acl, ctx` | Reassigns to a different network node. Must be offline. |

**Storage Unit**

| Function | Key extra arguments | What it does |
|---|---|---|
| `anchor` | `registry, network_node, character, admin_acl, item_id, type_id, max_capacity: u64, location_hash, ctx` → `StorageUnit` | Creates a new storage unit with a given inventory capacity. Then call `share_storage_unit()`. |
| `share_storage_unit` | `storage_unit, admin_acl, ctx` | Makes it a shared object. |
| `unanchor` | `storage_unit, network_node, energy_config, admin_acl, ctx` | Destroys a storage unit. Burns all inventory, deletes the object. |
| `update_energy_source` | `storage_unit, network_node, admin_acl, ctx` | Reassigns to a different network node. |
| `game_item_to_chain_inventory<T>` | `storage_unit, admin_acl, character, owner_cap, item_id: u64, type_id: u64, volume: u64, quantity: u32, ctx` | Bridges items FROM the game server TO on-chain inventory. Mints on-chain item entries. Sender must be character owner. |
| `chain_item_to_game_inventory<T>` | `storage_unit, server_registry, character, owner_cap, type_id: u64, quantity: u32, location_proof: vector<u8>, clock, ctx` | Bridges items FROM on-chain TO game server. Burns on-chain entries. Requires location proof (must be near the storage unit). |

**Network Node**

| Function | Key extra arguments | What it does |
|---|---|---|
| `deposit_fuel` | `nwn, admin_acl, owner_cap, type_id: u64, volume: u64, quantity: u64, clock, ctx` | Adds fuel to the node. Fuel burns over time to produce energy. |
| `withdraw_fuel` | `nwn, admin_acl, owner_cap, type_id: u64, quantity: u64, ctx` | Removes fuel from the node. |
| `anchor` | `registry, character, admin_acl, item_id, type_id, location_hash, fuel_max_capacity: u64, fuel_burn_rate_in_ms: u64, max_energy_production: u64, ctx` → `NetworkNode` | Creates a new network node with fuel/energy capacity. Then call `share_network_node()`. |
| `connect_assemblies` | `nwn, admin_acl, assembly_ids: vector<ID>, ctx` → `UpdateEnergySources` | Connects assemblies to this node for power. Returns hot potato — must call `update_energy_source_connected_*` for each assembly, then `destroy_update_energy_sources`. |
| `unanchor` | `nwn, admin_acl, ctx` → `HandleOrphanedAssemblies` | Begins destroying a node. Returns hot potato — must offline all connected assemblies, then call `destroy_network_node`. |
| `update_fuel` | `nwn, fuel_config, admin_acl, clock, ctx` → `OfflineAssemblies` | Recalculates fuel state. If fuel depleted, takes node offline and returns hot potato to cascade-offline all connected assemblies. |

**Character**

| Function | Key extra arguments | What it does |
|---|---|---|
| `create_character` | `registry, admin_acl, game_character_id: u32, tenant: String, tribe_id: u32, character_address: address, name: String, ctx` → `Character` | Creates a new on-chain character. CCP-only in practice (part of game registration flow). |
| `update_tribe` | `character, admin_acl, tribe_id: u32, ctx` | Changes a character's tribe. Server-authoritative. |
| `update_address` | `character, admin_acl, character_address: address, ctx` | Changes a character's wallet address. Server-authoritative. |
| `delete_character` | `character, admin_acl, ctx` | Deletes a character. Server-authoritative. |

#### TIER 4: CCP Only (GovernorCap)

| Function | What it does |
|---|---|
| `add_sponsor_to_acl` | Adds an address to the AdminACL allowlist. Only the deployer (CCP) can call this. |

#### OwnerCap Borrow/Return Pattern (How to Use OwnerCaps in a PTB)

OwnerCaps live inside Character objects (transferred to the character's UID address).
To use one, you borrow it and return it within the same Programmable Transaction Block:

```
PTB [
  1. borrow_owner_cap<Gate>(character, receiving_ticket, ctx) → (OwnerCap<Gate>, ReturnReceipt)
  2. authorize_extension<FrontierGateAuth>(gate, owner_cap)  → registers extension
  3. return_owner_cap(character, owner_cap, receipt)          → returns cap, consumes receipt
]
```

The `ReturnOwnerCapReceipt` hot potato ensures you can't keep the cap — it must be returned
in the same transaction. This prevents theft of OwnerCaps.

#### `tribe_permit.move` — CCP's Example (Compare to Ours)

CCP's `extension_examples/tribe_permit.move` uses an `ExtensionConfig` shared object with
dynamic fields for rules, and an `AdminCap` for configuration. Their `issue_jump_permit`
checks `character.tribe() == tribe_cfg.tribe` (single tribe match — one tribe allowed).
Our `frontier_gate` uses a `StandingsTable` with `Table<u32, u8>` (multi-tribe standings
with hostile/neutral/friendly levels). Ours is strictly more powerful — theirs is a
single-tribe whitelist, ours is a full diplomacy system.

#### What's NOT Writable (Server-Side Only)

- **Location** — Poseidon2 hash, set at anchor time by server, never updated after
- **Ship stats / combat / HP** — all server-side game logic
- **Solar system data** — not on-chain
- **Orbital Zones / Shells / Crowns** — server-side Cycle 5 features
- **Killmails** — created by server (admin-only), read-only for us
- **Tribe assignments** — `update_tribe` requires AdminACL (server decides tribe membership)

#### TypeScript Reference Scripts (builder-scaffold)

For implementation reference, `ts-scripts/` in builder-scaffold contains:

| Directory | Scripts |
|---|---|
| `character/` | `create-character.ts` |
| `assembly/` | `create-assembly.ts`, `online.ts` |
| `gate/` | `create-gates.ts`, `link-gates.ts`, `jump.ts` |
| `smart_gate/` | `authorise-gate.ts`, `configure-*.ts` |
| `turret/` | `anchor.ts`, `online.ts`, `get-priority-list.ts` |
| `storage-unit/` | `create-storage-unit.ts`, `deposit-*.ts`, `withdraw-*.ts`, `online.ts` |
| `assets/` | `game-item-to-chain.ts`, `chain-item-to-game.ts` |
| `utils/` | `derive-object-id.ts`, `proof.ts` |

### Transaction Mechanics

Sui transactions require:
1. **BCS serialization** — Binary Canonical Serialization of the TransactionData struct
2. **Intent signing** — prepend intent bytes (3 bytes: scope=0, version=0, app_id=0) to BCS bytes
3. **Ed25519 signature** — sign the intent message with `:crypto.sign(:eddsa, :sha512, msg, [privkey, :ed25519])`
4. **Signature encoding** — `<<0x00>> <> signature <> public_key` (scheme byte + 64-byte sig + 32-byte pubkey)
5. **Submit** — via GraphQL mutation `executeTransaction` (NOT `executeTransactionBlock`)

**GraphQL mutation for transaction submission:**
```graphql
mutation ExecuteTransaction($tx: Base64!, $sigs: [Base64!]!) {
  executeTransaction(transactionDataBcs: $tx, signatures: $sigs) {
    effects {
      status
      transactionBlock { digest }
      gasEffects { gasSummary { computationCost storageCost storageRebate } }
    }
  }
}
```
- `transactionDataBcs`: BCS-serialized unsigned TransactionData, Base64-encoded
- `signatures`: Array of `flag || signature || pubkey` bytes, Base64-encoded
- Blocks until finality (~2-3s), then returns effects
- Request timeout: 74 seconds. Payload limit: 175KB.

**Sponsored transactions** (game server pays gas): Sui protocol-level primitive. Two signatures
required (user + sponsor). CCP runs a gas pool service (forked from MystenLabs/sui-gas-pool).
Whether third-party dApps can use CCP's gas pool is undocumented — build without sponsorship
first, add as a layer later if the endpoint is published.

### AdminACL — Access Control for Write Operations (CRITICAL)

Source: `contracts/world/sources/access/access_control.move`

The world contracts use a three-tier capability hierarchy:

```
GovernorCap (CCP — deployer, root authority)
    │
    ▼
AdminACL (shared object — table of authorized sponsor addresses)
    │
    ▼
OwnerCap<T> (per-object — player holds these for their assemblies)
```

**The `verify_sponsor` function** is called by some (not all) world contract functions:

```move
public fun verify_sponsor(admin_acl: &AdminACL, ctx: &TxContext) {
    let sponsor_opt = tx_context::sponsor(ctx);
    let authorized_address = if (option::is_some(&sponsor_opt)) {
        *option::borrow(&sponsor_opt)    // check gas sponsor's address
    } else {
        ctx.sender()                      // OR check sender's address
    };
    assert!(admin_acl.authorized_sponsors.contains(authorized_address), EUnauthorizedSponsor);
}
```

This checks whether the gas sponsor (or sender) is in CCP's allowlist. CCP's gas pool
address IS in this list. So operations requiring `verify_sponsor` work when routed through
CCP's gas pool — the gas pool co-signs as sponsor and the check passes. This is the same
flow the game client and all standard dApps use.

Only `GovernorCap` holder (CCP) can add addresses to the allowlist via `add_sponsor_to_acl()`.

#### Operation-by-Operation AdminACL Requirements

**Player wallet ONLY (no gas pool co-sign needed):**

| Operation | Module | Requires |
|---|---|---|
| `authorize_extension` | gate, turret, storage_unit | OwnerCap only |
| `online` / `offline` | gate, turret, storage_unit | OwnerCap only |
| `update_metadata_name/description/url` | gate, turret, storage_unit | OwnerCap only |
| `unlink_gates` (by owner) | gate | OwnerCap for both gates |
| `deposit_by_owner` / `withdraw_by_owner` | storage_unit | OwnerCap only |
| Extension config/logic | builder's own package | Builder's AdminCap |
| `issue_jump_permit` | gate (via extension) | Extension Auth witness |

**Requires gas pool co-sign (AdminACL `verify_sponsor`):**

| Operation | Module | Why |
|---|---|---|
| `jump` / `jump_with_permit` | gate | Gameplay action, server tracks jumps |
| `link_gates` | gate | Needs server distance proof (has TODO to remove) |
| `anchor` / `share_*` | all assembly types | Structural creation, server authority |
| `unanchor` / `unanchor_orphan` | all assembly types | Structural destruction |
| `update_energy_source` | all assembly types | Server operation |
| `game_item_to_chain_inventory` | storage_unit | Bridge operation |
| `set_max_distance` | gate | Server config |

**Key insight**: All Sigil core features (configure extensions, online/offline, metadata,
owner inventory management) work with player wallet only. Gas-pool-required operations are
gameplay actions (jumping, building, destroying) and server-authority operations. The command
center use case is architecturally sound without needing CCP's gas pool.

If CCP's gas pool IS available to third-party dApps (likely for the hackathon), then ALL
operations become available, expanding Sigil to cover gate linking, assembly creation, etc.

### Extension Authorization Pattern (Sigil Uses This)

Smart Assembly extensions use a **typed witness** pattern, NOT the AdminACL:

1. Builder deploys a Move package defining a zero-sized witness struct (e.g., `FrontierGateAuth`)
2. Player calls `gate::authorize_extension<FrontierGateAuth>()` with their OwnerCap — NO AdminACL
3. Once registered, default `jump()` is **blocked entirely** — must use `jump_with_permit()`
4. Extension issues `JumpPermit` objects using its Auth witness — NO AdminACL

#### Gate Extension Interface (Permit-Based, NOT Allow/Deny)

The gate extension does NOT do a simple "check and return bool." It works through **JumpPermit
issuance** — a two-step flow:

**Step 1: Traveler requests a permit from the extension**
```move
// In YOUR extension package — called by the traveler
public fun request_permit(
    standings_table: &StandingsTable,
    source_gate: &Gate,
    destination_gate: &Gate,
    character: &Character,
    clock: &Clock,
    ctx: &mut TxContext,
) {
    // Check standings — deny hostile tribes
    let tribe_id = character::tribe(character);
    let standing = get_standing(standings_table, tribe_id);
    assert!(standing != HOSTILE, EAccessDenied);
    // Issue permit via world contract (proves we are the registered extension)
    gate::issue_jump_permit<FrontierGateAuth>(
        source_gate, destination_gate, character,
        FrontierGateAuth {},  // witness instance
        clock::timestamp_ms(clock) + 300_000, // 5 min expiry (configurable)
        ctx,
    );
}
```

**Step 2: Traveler jumps with the permit** (standard world contract call)
```move
gate::jump_with_permit(source_gate, dest_gate, character, permit, admin_acl, clock, ctx)
```

The world contract's `issue_jump_permit` function:
```move
public fun issue_jump_permit<Auth: drop>(
    source_gate: &Gate, destination_gate: &Gate, character: &Character,
    _: Auth,  // witness proving caller is the registered extension
    expires_at_timestamp_ms: u64, ctx: &mut TxContext,
)
```
It verifies `type_name::with_defining_ids<Auth>()` matches `source_gate.extension`, creates a
`JumpPermit` with a bidirectional route hash, and transfers it to the character's address.

#### Turret Extension Interface (BCS-Encoded I/O)

The turret extension receives BCS-encoded target candidates and returns BCS-encoded priority list:

```move
// In YOUR extension package — called by the game server
public fun get_target_priority_list(
    turret: &Turret,
    _: &Character,                      // owner character
    target_candidate_list: vector<u8>,  // BCS-encoded vector<TargetCandidate>
    receipt: OnlineReceipt,             // hot potato — MUST consume
) -> vector<u8> {                       // BCS-encoded vector<ReturnTargetPriorityList>
    assert!(receipt.turret_id() == object::id(turret), EInvalidReceipt);
    let candidates = turret::unpack_candidate_list(target_candidate_list);
    let mut results = vector::empty<ReturnTargetPriorityList>();
    // Filter by standings — high priority for hostile, zero for friendly
    // ... standings logic here ...
    let result = bcs::to_bytes(&results);
    turret::destroy_online_receipt(receipt, FrontierTurretAuth {}); // consume hot potato
    result
}
```

**TargetCandidate** input fields: `item_id: u64`, `type_id: u64`, `group_id: u64`,
`character_id: u32`, `character_tribe: u32`, `hp_ratio: u64`, `shield_ratio: u64`,
`armor_ratio: u64`, `is_aggressor: bool`, `priority_weight: u64`,
`behaviour_change: BehaviourChangeReason` (enum: UNSPECIFIED/ENTERED/STARTED_ATTACK/STOPPED_ATTACK)

**ReturnTargetPriorityList** output fields: `target_item_id: u64`, `priority_weight: u64`

**OnlineReceipt** is a hot potato (no `drop`/`store`) — must be consumed via
`turret::destroy_online_receipt<Auth: drop>(receipt, auth_witness)`.

Ship group IDs: Shuttle=31, Corvette=237, Frigate=25, Destroyer=420, Cruiser=26,
Combat Battlecruiser=419. Turret types: Autocannon=92402, Plasma=92403, Howitzer/Railgun=92484.
(Press materials say "Railgun" but contract code says "Howitzer" — same turret, type_id 92484.)

**Combat data available to turret extensions** (via TargetCandidate): hp_ratio, shield_ratio,
armor_ratio, is_aggressor, ship group_id. Cycle 5's combat overhaul (light vs heavy ship
differentiation) changes the *values* but the data interface is unchanged.

#### Extension Examples in world-contracts

The `extension_examples/` directory contains CCP's own reference implementations:
- **`turret.move`** — type-specific targeting (Autocannon→small ships, Plasma→mid, Howitzer→large)
- **`tribe_permit.move`** — tribe-based gate access control (very similar to our `frontier_gate`)
- **`corpse_gate_bounty.move`** — bounty system using gates and corpses
- **`config.move`** — configuration management pattern for extensions

**`tribe_permit.move` is directly relevant** — it's CCP's own example of the exact pattern
Sigil uses. **Fully analyzed**: checks `character.tribe() == config.tribe` (single u32
match, one tribe per gate). Our standings-based approach is strictly more powerful (graduated
access, multi-tribe, per-tribe updates) but uses the same typed-witness + dynamic field patterns.

**Important constraint**: Each assembly supports only **ONE extension at a time**. Attaching a
new extension replaces the previous one. This means a tribe can't stack gate extensions — our
single `frontier_gate` extension must handle all access logic.

#### Sigil Move Extensions

- **`frontier_gate`** — gate extension that reads a `StandingsTable` shared object.
  Issues `JumpPermit`s to travelers from friendly/neutral tribes. Denies hostile tribes
  (assert fails, no permit issued → can't jump).
- **`frontier_turret`** — turret extension that reads the SAME `StandingsTable`.
  Returns high priority_weight for hostile tribe ships, zero for friendly/neutral
  (effectively: target hostile, ignore friendly).
- **`StandingsTable`** — a Sui shared object using `Table<u32, u8>` (tribe ID → standing).
  Standing values: 0=hostile, 1=neutral, 2=friendly. Default for unknown tribes: neutral (1).
  Owner-only write access. Functions: `create(ctx)`, `set_standing(table, tribe_id, standing, ctx)`,
  `get_standing(table, tribe_id): u8`, `batch_set_standings(table, ids, standings, ctx)`.
  **Per-tribe**: tribe leader creates one StandingsTable instance. Members enroll their
  assemblies' extensions to reference that table's object ID. The table owner (and authorized
  roles — see on-chain roles below) controls diplomatic policy for all enrolled assemblies.

**Universal contract deployment**: `frontier_gate`, `frontier_turret`, and `standings_table`
are published **once** as a single Move package. All tribes use the same contract code —
the per-tribe part is the StandingsTable object ID passed as a parameter. One `sui client
publish`, everyone benefits.

This is the architectural keystone: a single diplomacy change in Sigil propagates to
ALL infrastructure automatically. The on-chain contract enforces it — no drift possible.

### Architectural Philosophy: Thin Contracts, Rich Elixir

**Move contracts are passive.** They cannot make HTTP calls, contact external services, or
call back to Elixir. They only execute when someone submits a transaction that calls them.
They can read/write on-chain state and emit events. That's it.

**Elixir is the nervous system.** It polls, monitors, correlates, alerts, and drives the UI.
The BEAM's GenServer-per-assembly pattern is where the real architectural advantage lives —
thousands of concurrent monitors, self-healing supervision, real-time PubSub fan-out.

**Design principle**: Keep contracts as thin as possible — shared state, simple enforcement
rules, and public surface for ecosystem composability. All complex logic, user interaction,
monitoring, and orchestration lives in Elixir. Deploy one universal contract package — all
tribes use the same code, creating per-tribe StandingsTable instances parameterized by
object ID.

### Sponsored Transaction Flow (for gas-pool-required operations)

```
1. Sigil constructs transaction kind (MoveCall + args)
2. Sigil sends tx kind bytes to CCP's gas pool API:
   POST /v1/reserve_gas {gas_budget, reserve_duration_secs}
   → returns: sponsor_address, reservation_id, gas_coin ObjectRefs
3. Sigil wraps tx with GasData (sponsor as gas owner)
4. Player signs the full TransactionData
5. Sigil sends to gas pool:
   POST /v1/execute_tx {reservation_id, tx_bytes, user_signature}
   → gas pool co-signs and submits to Sui
   → returns: SuiTransactionBlockEffects
```

The gas pool is a Rust service backed by Redis. CCP's fork: https://github.com/evefrontier/sui-gas-pool
Original: https://github.com/MystenLabs/sui-gas-pool

### World API REST Endpoints (Stillness server)

Base URL: `https://world-api-stillness.live.tech.evefrontier.com`

**Status as of Day 1**: Static reference endpoints (types, solarsystems, tribes) return 200.
Some game-state endpoints (killmails) now return 404 — likely mid-migration to Sui. The
Stillness blockchain gateway (EVM-era) is fully decommissioned (ECONNREFUSED). Use this API
for **static reference data only**; live game state should come from Sui event subscriptions.

All endpoints support pagination via `limit` (default 100) and `offset` query parameters.

| Endpoint | Status (Day 1) | Records | Key Fields |
|---|---|---|---|
| `/v2/types` | **LIVE** | 336 | id, name, categoryName, mass, volume, groupName, iconUrl |
| `/v2/solarsystems` | **LIVE** | 24,502 | id, name, location{x,y,z}, constellationId, regionId |
| `/v2/tribes` | **LIVE** | 47 | id, name, nameShort, memberCount, taxRate, foundedAt |
| `/v2/smartassemblies` | **LIVE** | 35,804 | id, type, name, state, owner{address,name}, solarSystem{location} |
| `/v2/killmails` | **404** | — | Was: id, victim, killer, solarSystemId, time |
| `/v2/smartcharacters` | **LIVE** | 5,172 | address, name, id |
| `/health` | **LIVE** | — | `{"ok": true}` |

`/v2/smartassemblies` supports type filter: `?type=SmartGate` (143), `?type=SmartTurret` (3,160),
`?type=SmartStorageUnit` (3,784), `?type=NetworkNode` (5,282).

**Location data nuance**: The `solarSystem.location` coordinates on assemblies are the
**solar system's position**, NOT the assembly's position within that system. All assemblies
in the same system share identical x/y/z. So the API reveals which star system an assembly
is in, but not where within that system. Solar system coordinates themselves (from
`/v2/solarsystems`) are universe-map positions — useful for rendering a galaxy map but
not for locating individual players or structures.

**Location privacy on Sui**: CCP's blog states *"The world will start with full location
obfuscation."* The `eve-frontier-proximity-zk-poc` repo contains a working ZK proof system
(Poseidon Merkle trees, Groth16 circuits) for verifying proximity without revealing
coordinates. Full stack: Seal (access controls) + Walrus (storage) + ZK proofs. This is
designed but NOT yet deployed — the PoC says on-chain verification is "planned for future
integration." Expect Sui-era APIs to restrict location data compared to the Stillness API.

**Hackathon note**: The hackathon runs on a dedicated **Utopia server** on Sui testnet.
`world-api-utopia.live.tech.evefrontier.com` does NOT resolve as of Day 1 — Utopia may not
have its own World API. Instead, interact with the Utopia world via Sui testnet RPC directly
using the Utopia package ID (see "Deployed Package IDs" below). Use Stillness World API for
**static reference data** (types, system names) but do NOT assume it reflects Utopia game state.

### Sui Address Transparency (Privacy Model)

**Sui has NO built-in privacy layer.** Every transaction exposes:
- **Sender address** — who submitted it
- **Gas payer** — same or different if sponsored (sponsorship does NOT hide sender)
- **Function called** — exact package/module/function
- **Arguments** — object IDs and pure values (decodable)
- **Objects touched** — character, gate, inventory objects
- **State changes** — what was created, modified, transferred, deleted
- **Object ownership** — every object's current owner address is publicly queryable

**What's private**: Seed phrase, local wallet structure, real-world identity. Nothing else.

**Multiple addresses** are easy to create but get linked through behavior: fund transfers
between them, touching the same objects, same gas sponsor patterns, timing correlations.

**Architectural implication for Sigil**: Any feature involving player anonymity (e.g.,
intel scouting) must use Sigil's platform wallet as a proxy. If a scout transacts
directly on-chain, their wallet address is visible and can be traced back to their character
via OwnerCap ownership. Sigil posting on behalf of scouts (identity stored only in
Postgres) is the only practical anonymity model on a transparent chain.

### Deployed Package IDs (Sui Testnet)

Found in `Published.toml` in the `world-contracts` repo (Sui's automated address management).
All on Sui Testnet (chain-id `4c78adac`), all at version 1, built with toolchain v1.67.1.

| Environment | Package ID |
|---|---|
| **testnet_stillness** | `0x28b497559d65ab320d9da4613bf2498d5946b2c0ae3597ccfda3072ce127448c` |
| **testnet_utopia** | `0xd12a70c74c1e759445d6f209b01d43d860e97fcf2ef72ccbbd00afd828043f75` |
| **testnet_internal** | `0x353988e063b4683580e3603dbe9e91fefd8f6a06263a646d43fd3a2f3ef6b8c1` |
| **testnet** (generic/local) | `0x920e577e1bf078bad19385aaa82e7332ef92b4973dcf8534797b129f9814d631` |

**For Sigil hackathon development**: Use `testnet_utopia` package ID for Utopia game state,
`testnet_stillness` for Stillness. Set via `WORLD_PACKAGE_ID` env var (or `VITE_EVE_WORLD_PACKAGE_ID`
in the dapp-kit).

**Note**: The package ID alone is not sufficient — you also need shared object IDs (ObjectRegistry,
AdminACL, GovernorCap, ServerAddressRegistry) created at deploy time. These are in
`extracted-object-ids.json` from CCP's deployment. For local dev, `setup-world.sh` generates them.
For live servers, query them on-chain via `getObjectsByType` or obtain from Discord.

**Sui RPC endpoint**: `https://fullnode.testnet.sui.io:443`

### Local Development Environment

The `builder-scaffold` repo (`~/builder-scaffold/`) provides a complete offline dev environment
with Docker. No game client needed for developing and testing Move contracts or Sui interactions.

**Starting the environment**:
```bash
cd ~/builder-scaffold/docker && sudo docker compose up -d
# Wait ~20s for node to initialize and fund accounts
~/builder-scaffold/sui-local.sh status    # Check container status
~/builder-scaffold/sui-local.sh pause     # Zero-CPU when not in use
~/builder-scaffold/sui-local.sh resume    # Resume containers
```

**Deploying world contracts** (needed after each restart — `--force-regenesis`):
```bash
sudo docker exec docker-sui-dev-1 bash -c "cd /workspace/world-contracts && \
  pnpm deploy-world localnet && \
  pnpm configure-world localnet && \
  pnpm create-test-resources localnet"
```

**Ports**: 9000 (Sui RPC), 9125 (Sui GraphQL + indexer), 5432 (PostgreSQL, internal).

**Connecting Sigil**:
```bash
# World package ID (changes on every container restart / regenesis)
export SUI_LOCALNET_PACKAGE_ID=$(jq -r '.world.packageId' \
  ~/builder-scaffold/docker/world-contracts/deployments/localnet/extracted-object-ids.json)

# Sigil contracts package ID (from `sui client test-publish` output)
export SUI_LOCALNET_SIGIL_PACKAGE_ID=<package id from Pub.localnet.toml>

# Server-side signing key (hex private key from keystore — see below)
export SUI_LOCALNET_SIGNER_KEY=<64-char hex private key>

EVE_WORLD=localnet iex -S mix phx.server
```

**Environment variables:**

| Variable | Purpose | How to get it |
|----------|---------|---------------|
| `EVE_WORLD` | Switches GraphQL endpoint + package IDs | Set to `localnet` |
| `SUI_LOCALNET_PACKAGE_ID` | World contracts package ID | `jq -r '.world.packageId' ~/builder-scaffold/docker/world-contracts/deployments/localnet/extracted-object-ids.json` |
| `SUI_LOCALNET_SIGIL_PACKAGE_ID` | Sigil contracts package ID | From `Pub.localnet.toml` after `sui client test-publish` |
| `SUI_LOCALNET_SIGNER_KEY` | Hex private key for server-side tx signing | Decode keystore Base64: `echo <base64_entry> \| base64 -d \| xxd -p` (skip first byte = scheme flag) |

**Getting the signer key**: The keystore is at `/root/.sui/sui.keystore` inside the Docker
container (JSON array of Base64 strings). Each entry is 1 flag byte + 32 key bytes. Decode
with: `echo 'ABkf8XH1bBvruXP6wRfRir3fnDO9LdfV2A9aRvrH69I/' | base64 -d | tail -c 32 | xxd -p -c 32`

**WARNING**: Do NOT use `sui keytool convert` for the hex key — it may produce different
bytes than the keystore. Always extract from the keystore directly.

**Note**: On localnet, diplomacy transactions are signed server-side and submitted via
JSON-RPC (Slush can't resolve gas coins on local networks, and the GraphQL indexer is too
slow for mutations). On testnet/mainnet, the wallet handles signing normally via
`Transaction.fromKind()`.

**Authentication**: Use Slush wallet (not EVE Vault) with an imported localnet private key.
Sigil's ZkLoginVerifier auto-detects Ed25519 signatures and verifies directly.

#### Localnet Wallets

| Role | Address | Private Key |
|------|---------|-------------|
| ADMIN | `0x47951e3c2ca0930c350fa820a7b760dcbd11459a834e8d448d14c1db94fddb4a` | `suiprivkey1qzgllknf4rq8v9zgnwuydt08uect9lvd62rzmxxrtwfvtlzwav4lwlgle2m` |
| PLAYER_A | `0x459e95a7c847d6971dcaef7990ee84302f44e815e9d5953a63eb9a0d1f5c3b5f` | `suiprivkey1qrsnzakk9a2lltgaq6n3mqtqnpq4lrcxnp5ypulzsm2gx4rla3whw3aup2y` |
| PLAYER_B | `0xd1495097940286120b68eaa5fd199330f72b61484f7d4e8ec0692922b12533ad` | `suiprivkey1qr7ec947cvtydy9h5ztrepep8xy0ns69jtqzru9lchruz7ls3c84v4glced` |

**Note**: Keys change on every `docker compose down -v` (volume wipe). Check `docker/.env.sui` for current values.

Keys persist across restarts (Docker volume). ADMIN owns the GovernorCap. Keys are also
available inside the container at `docker/.env.sui`.

#### Localnet Characters

| Wallet | Character Object | Game Character ID | Name | Tribe |
|--------|-----------------|-------------------|------|-------|
| PLAYER_A (`0x459e...`) | `0x5d079483f095e4fefb43ec3ec754b961497a95587855306e5518dfe18c38212c` | 811880 | frontier-character-a | 100 |
| PLAYER_B (`0xd149...`) | `0xbc89cc8846b3e26ce447b866008bbf9d89c302d5f9fe554882164b2387555b17` | 900000001 | frontier-character-a | 100 |

Characters are **shared objects** on Sui. The wallet link is via the `character_address` field
inside the Character, not Sui ownership. OwnerCaps for assemblies are held by the Character
object, not the wallet.

#### Localnet Assets (owned by PLAYER_A's Character `0x5d07...`)

| Type | Object ID | Item ID | Status | Notes |
|------|-----------|---------|--------|-------|
| Gate | `0x52448651ea6ac65cb98ce537e8b1292c09158fd20d19f3fa2454f666d656fd34` | 90185 | Online | Linked pair |
| Gate | `0x1a5cac9cba0c67bca43daccdfcad1449320b9d183deec75a414c30df9e9695b0` | 90186 | Online | Linked pair |
| NetworkNode | `0x53e79101d38621eae5cd4df01c53d92ce76a2773c37ebe3e79a19a2adeb94173` | 5550000012 | Online | 1/10000 fuel |
| StorageUnit | `0x17e6c81b927b8a6ee7977ade7ac2892a8ff57908a8d082bcca64f507aace989d` | 888800006 | Online | Has inventory item 444000001 |

PLAYER_B's character has no assets. Additional objects can be created via npm scripts inside
the container (e.g. `pnpm create-assembly`, `pnpm create-gates`).

#### Localnet World Contract IDs

World Package: `0x3b4c2fb1653b5ab363545955bef205ac972fd75624746c729049aa573a513498`

| Object | ID |
|--------|----|
| GovernorCap | `0x425d1ccaa834cd50c94b9c5d1123d7086066344439e7883a8d7b3e2bcaf39093` |
| ServerAddressRegistry | `0x239828372eeb13287758619455e22175930dc6898675382ff6b5ce71973ef730` |
| AdminAcl | `0x216b22c5b30621b9efeedca7072aa365bda6dfdfe665006bd02f6a32d8ae6e74` |
| ObjectRegistry | `0x9f4715b13d8a09a7d13a19501760d54285bcd28592b2c26aa5e153e59bc2cf75` |
| EnergyConfig | `0x651d432d654ab4c4c941751c36d86eda0203d41029504a8962dc4f2139752956` |
| FuelConfig | `0xf78dfb3e3ba5e8c57c699605c4a1b00317a3c30b68badb0770b85476ba84e8e7` |
| GateConfig | `0x98451adbb4d9ed1611b006c8a74b27fc426fb0357f873813ec16027799b9e73f` |

**Note**: These IDs change on every container restart (`--force-regenesis`). The `eve_worlds`
config entry for `"localnet"` in `config/config.exs` must be updated if the package ID changes.

**Object ID derivation**: EVE Frontier uses deterministic IDs. Each in-game resource has an
`item_id` + `type_id` + `tenant`. The `TenantItemId` maps to on-chain Sui objects via the
`ObjectRegistry`. Since derivation is deterministic, you can pre-compute object IDs off-chain
without querying the chain (see `ts-scripts/utils/derive-object-id.ts` for reference).

### dapp-kit TypeDoc Reference

**LIVE** at `http://sui-docs.evefrontier.com/` — `@evefrontier/dapp-kit` v0.1.5.

Key constants and functions (from TypeDoc):
- `getEveWorldPackageId()` — reads `VITE_EVE_WORLD_PACKAGE_ID` env var
- `DEFAULT_TENANT` = `"stillness"` (selectable via `?tenant=` URL param)
- `GRAPHQL_ENDPOINTS` — per-network GraphQL URLs (testnet, devnet, mainnet)
- `executeGraphQLQuery()` — generic GraphQL helper
- `getObjectsByType()` — query objects by Move type
- `getAssemblyWithOwner()` — assembly + owner data
- `getWalletCharacters()` — characters owned by a wallet address
- `getSingletonObjectByType()` — single-instance object queries

**Sponsored transaction actions**: `online`, `offline`, `update-metadata`, `link-smart-gate`,
`unlink-smart-gate`.

**TYPEIDS enum** (numeric game type IDs):
| Type | ID |
|---|---|
| Lens | 77518 |
| Transaction Chip | 79193 |
| Common Ore | 77800 |
| Metal Rich Ore | 77810 |
| Smart Storage Unit | 77917 |
| Protocol Depot | 85249 |
| Gatekeeper (Gate) | 83907 |
| Salt | 83839 |

### Object Ownership Model

Game objects (Assemblies, Gates, etc.) are **shared objects** on Sui — globally accessible, not
"owned" by an address in Sui's ownership model. To find a player's assemblies:
1. Query `OwnerCap` objects owned by the player's Sui address
2. Each OwnerCap links to the assembly it controls via `owner_cap_id` fields

### Object ID Derivation

Game item IDs can be deterministically mapped to Sui object addresses:
1. BCS-serialize a `TenantItemId` struct: `{id: u64, tenant: string}`
2. Construct type tag: `{packageId}::in_game_id::TenantItemId`
3. Call derivation function against the ObjectRegistry shared object
Reference implementation: `ts-scripts/utils/derive-object-id.ts`

### Location Data (CONFIRMED: Hashed, Not Raw Coordinates)

Location is stored as a **Poseidon2 hash** (32 bytes). Raw x/y/z coordinates never appear
on-chain. The `Location` struct contains only `location_hash: vector<u8>`.

```move
public struct Location has store {
    location_hash: vector<u8>,  // 32-byte Poseidon2 hash, enforced by assert
}
```

Distance verification requires **server-signed proofs** containing source/target IDs,
location hashes, computed distance, and an Ed25519 signature from a registered server
address. The `verify_distance()` function checks the signature against the
`ServerAddressRegistry` shared object.

**Implications for Sigil**: No spatial map from on-chain data alone. Use topology-only
views (gate network graph, connectivity). If the EVM REST API is still running, it may
provide raw coordinates for solar systems — cache locally if available.

---

## Key Resources

### Official
- **Builder docs**: https://docs.evefrontier.com (SPA — use browser MCP if WebFetch fails)
- **World contracts (Move source)**: https://github.com/evefrontier/world-contracts
- **Builder scaffold**: https://github.com/evefrontier/builder-scaffold
- **EVE Vault (wallet)**: https://github.com/evefrontier/evevault
- **Gas pool**: https://github.com/evefrontier/sui-gas-pool
- **Hackathon page**: https://deepsurge.xyz/evefrontier2026

### Sui
- **Sui docs**: https://docs.sui.io
- **GraphQL RPC**: https://docs.sui.io/guides/developer/advanced/graphql-rpc
- **GraphQL mutation reference**: https://docs.sui.io/references/sui-api/sui-graphql/beta/reference/operations/mutations/execute-transaction
- **Event querying**: https://docs.sui.io/guides/developer/sui-101/using-events
- **Sponsored transactions**: https://docs.sui.io/concepts/transactions/sponsored-transactions
- **BCS spec**: https://docs.sui.io/concepts/sui-move-concepts/collections (BCS encoding)
- **BCS test vectors**: https://github.com/MystenLabs/ts-sdks/tree/main/packages/bcs/tests
- **BCS reference implementation**: https://github.com/diem/bcs (original Rust spec)
- **suiup version manager**: https://github.com/MystenLabs/suiup
- **gRPC proto files**: https://github.com/MystenLabs/sui-apis/tree/main/proto

### Community
- **EF-Map** (dominant community tool): https://ef-map.com
- **EFCopilot**: https://efcopilot.com
- **EVE Frontier Tools (data extraction)**: https://github.com/VULTUR-EveFrontier/eve-frontier-tools

### Elixir Packages (for mix.exs)

```elixir
defp deps do
  [
    {:phoenix, "~> 1.7"},
    {:phoenix_live_view, "~> 1.0"},
    {:phoenix_live_dashboard, "~> 0.8"},
    {:ecto_sql, "~> 3.12"},      # Deferred to Slice 3 (alert persistence)
    {:postgrex, "~> 0.19"},      # Deferred to Slice 3 (alert persistence)
    {:req, "~> 0.5"},
    {:jason, "~> 1.4"},
    {:blake2, "~> 1.0"},  # Blake2b-256 for transaction digests (REQUIRED — see note below)
    # For gRPC streaming (stretch goal):
    # {:grpc, "~> 0.11.5"},
    # {:protobuf, "~> 0.14.1"},
    # For WebSocket (if needed):
    # {:websockex, "~> 0.5.1"},
  ]
end
```

- **Ed25519 signing**: Erlang's built-in `:crypto` module — no additional dependency needed.
- **BCS encoding**: Pure Elixir — no dependency needed.
- **Blake2b-256**: REQUIRES `{:blake2, "~> 1.0"}` (pure Elixir). Erlang's `:crypto.hash(:blake2b, data)`
  produces 512-bit output only. Truncating to 256 bits gives **wrong results** — BLAKE2 bakes
  the output length into initialization. Use `Blake2.hash2b(data, 32)` for correct 32-byte digests.

**BCS test vectors** for verifying our encoder: `MystenLabs/ts-sdks/packages/bcs/tests/` contains
comprehensive test cases. Key rules: all integers little-endian, ULEB128 for lengths, Options are
`0x00`/`0x01+value`, addresses are fixed 32 bytes (no length prefix), structs serialize fields
in declaration order.

---

## Competitive Landscape

### Existing Tools

**EF-Map** (ef-map.com) — The dominant tool. 55k LOC, 1,116 commits. React/Three.js/WebGL.
- 3D star map (24k+ systems), route planning (Dijkstra + A*), heat-aware routing
- Killboard with blockchain-indexed data, leaderboards across time windows
- Blueprint calculator with recursive material dependency trees
- Log parser (mining, combat, travel analytics) — runs in-browser via Web Workers
- Windows desktop overlay (DirectX 12 hook, ImGui) — real-time location sync, DPS tracking
- Real-time event streaming via Cloudflare Durable Objects + WebSockets
- Tribe Marks — shared text annotations on the map (anonymous, no auth)
- Privacy-first, no accounts required
- Stack: React/TS, Three.js, Cloudflare Pages/Workers/KV, PostgreSQL, Hetzner VPS

**EFCopilot** (efcopilot.com) — Industry-focused challenger.
- Industry calculator with byproduct optimization (differentiator vs EF-Map)
- Inventory tracking and production planning
- Deep Scan — tribe and character search/lookup
- Galaxy map + route planning, killboard
- No desktop overlay, no log parser, no real-time streaming

**EVE Datacore** (evedataco.re) — Open-source blockchain data explorer.
- Raw on-chain data browsing. Niche developer/power-user tool.
- GitHub: beaukode/eve-frontier-tools, 817 commits, CC BY-NC 4.0

### What NONE of Them Have

| Gap | Status |
|---|---|
| Tribe/organization management (roster, roles, permissions, resource tracking) | **WIDE OPEN** |
| Smart Assembly management from outside the game | **WIDE OPEN** |
| Write capability (any blockchain interaction) | **WIDE OPEN** |
| Bulk on-chain operations | **WIDE OPEN** |
| Alerts/notifications (fuel low, offline, hostile activity) | **WIDE OPEN** |
| Diplomatic standings → automated ACL configuration | **WIDE OPEN** |
| Marketplace/trading tools | **WIDE OPEN** |
| Fleet coordination | **WIDE OPEN** |
| Mobile companion | **WIDE OPEN** |

### Known Hackathon Competitor

**CradleOS** (github.com/r4wf0d0g23/CradleOS + github.com/r4wf0d0g23/Reality_Anchor_Eve_Frontier_Hackathon_2026)

**IMPORTANT: Both repos are the SAME developer/team.** CradleOS is the deployed app; Reality_Anchor is the dev monorepo. One competitor, not two.

**Current status (as of Day 5, Mar 15 2026):** Actively developing. Both repos pushed today.

**What they have deployed (React 19 + TypeScript + Vite + Three.js, GitHub Pages SPA):**
- 5 frontend tabs: Structures, Tribe Coin, Defense, Registry, Starmap
- Ship fitting calculator (14 ships, 123 modules, drag-and-drop, CPU/PG constraints)
- 3D starmap (Three.js, 24,502 solar systems, BFS route planning with heat/fuel physics)
- Structure registry with HEAVY/STANDARD/MINI tier badges, online/offline, fuel tracking
- EVE Vault (CCP's Chrome wallet extension) integration via @evefrontier/dapp-kit

**9 deployed Move contracts on Sui testnet (package `0x036c...`):**

| Module | Lines | Tests | What It Does |
|--------|-------|-------|-------------|
| `registry.move` | ~50 | 2 | Corp directory, counter |
| `corp.move` | ~180 | 6 | Member/Officer/Director roles with MemberCap capabilities |
| `treasury.move` | ~130 | 5 | SUI treasury, member deposit, director-only withdraw |
| `cradle_coin.move` | ~40 | 0 | CRADLE_COIN (CRDL) one-time-witness token |
| `tribe_vault.move` | ~200 | 0 | Infra-backed CRDL minting, tribe coin accounting, structure registration |
| `tribe_dex.move` | ~150 | 0 | Order book DEX — tribe coins vs CRDL |
| `defense_policy.move` | ~200 | 0 | Hostile/friendly per-tribe, SEC GREEN/YELLOW/RED, aggression mode, passage logging |
| `gate_control.move` | ~200 | 5 | **Real world::gate JumpPermit issuance** via typed witness, tribe membership check, blocked list |
| `character_registry.move` | ~220 | 0 | Proof-based tribe claim with epoch-priority challenge + attestor service |
| `contributions.move` | ~100 | 4 | Member contribution points, officer-only logging |

**Their documented future plans (from DESIGN.md + NAV_MISSION_BRIEF.md):**
- `corp_turret.move` — "member-exempted auto-defense" (NOT yet implemented)
- `corp_hangar.move` — "multi-asset SSU"
- `corp_beacon.move` — "member broadcast events"
- `corp_intel.move` — "aggregate + score + alert"

**Their known blockers (from DESIGN.md):**
- UID access for dynamic fields on assemblies — world package may not expose `uid_mut()`
- Both gates in a pair must use same extension type — cross-corp gate pairs are problematic
- Deposit flow requires two transactions (deposit to SSU first, then transfer to corp)

**Where CradleOS beats us:**
- 9 deployed Move contracts vs our 0. They have gate control with JumpPermit issuance working.
- Token economics (CRDL + DEX) — blockchain-native innovation we don't have planned.
- Ship fitting calculator — daily utility for players.
- Role hierarchy (Member/Officer/Director with on-chain capability objects).
- Broad feature surface — looks impressive in a demo even if tabs are partially empty.

**Where we beat CradleOS:**
- **No backend.** Static React SPA on GitHub Pages. No server, no database, no monitoring, no push updates. They cannot do background processing, alerts, or real-time multi-user sync.
- **No turret extension.** defense_policy.move is advisory only ("members are expected to apply"). No actual world::turret integration. This is our unique Move contract opportunity.
- **No assembly monitoring.** No fuel forecasting, no offline detection, no alerts.
- **No multi-user collaboration.** Browser-only means no shared real-time state between tribe officers.
- **Minimal testing.** 22 Move tests across 4 modules, 5 modules with zero tests. Single-commit big-bang push. Our 266 tests + TDD discipline is a Technical Excellence differentiator.
- **No event history.** SPA loses all state on page refresh. No audit trail, no timeline.

**Strategic positioning:**
- CradleOS targets Gameplay Impact and Creativity (token economy, role system, 3D starmap).
- Sigil targets **Technical Excellence** (pure Elixir Sui integration, OTP supervision, real-time architecture) with strong Utility angle (diplomacy-aware mapping, monitoring, alerts).
- We differentiate on **diplomacy as a first-class game system**: standings-aware route planning (no other tool does this), on-chain enforcement cascade, and the strategic map as the surface where diplomacy becomes spatially tangible.
- Additional depth: persistent monitoring, fuel forecasting, webhook alerts, multi-user collaboration — capabilities that require a real backend.
- Our turret extension (frontier_turret.move) is a unique Move contract CradleOS doesn't have.
- The "cascade" demo — change one standing, watch ALL gates deny + ALL turrets target + route map recolors in real time — is our signature moment.

**2025 hackathon context**: Only 16 participants, much smaller prizes ($3k travel vouchers).
2026 will be bigger due to 25x prize increase but still niche.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Phoenix LiveView UI                         │
│  /dashboard  /assembly/:id  /tribe/:id/diplomacy  /alerts      │
│  Real-time updates via PubSub subscription                      │
└──────────────────────┬──────────────────────────────────────────┘
                       │ PubSub
┌──────────────────────┴──────────────────────────────────────────┐
│                    Elixir Contexts (Domain Logic)                │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│  │  Accounts   │ │  Diplomacy   │ │  Assemblies (Gates/Turr) │ │
│  └─────────────┘ └──────────────┘ └──────────────────────────┘ │
│  ┌─────────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │ AssemblyMonitor  │ │ Alert Engine │ │      Tribes          │ │
│  │ (GenServer/each) │ │ (rules)      │ │                      │ │
│  └────────┬────────┘ └──────┬───────┘ └──────────────────────┘ │
│  ┌────────┴──────────────────┴──────────────────────────────────┐ │
│  │              ETS Cache (primary data layer)                   │ │
│  └──────────────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Postgres — Slice 3+: alert storage, session preferences      │ │
│  └──────────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────────────┐
│                     Sui Integration Layer                        │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────────────────┐ │
│  │ GraphQL Read  │ │ BCS Encoder  │ │ Transaction Builder     │ │
│  │ (Req + JSON)  │ │ (pure Elixir)│ │ + Ed25519 Signer        │ │
│  └──────────────┘ └──────────────┘ └─────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTPS
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Sui Blockchain (Testnet)                      │
│                                                                 │
│  EVE Frontier World Contracts        Sigil Move Package    │
│  ┌──────────────────────┐            ┌────────────────────────┐ │
│  │ gate, turret,        │            │ standings_table.move   │ │
│  │ assembly, character, │◄──reads────│ frontier_gate.move     │ │
│  │ network_node, etc.   │            │ frontier_turret.move   │ │
│  └──────────────────────┘            └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Development model**: Vertical slices. Each feature cuts through all layers — Move contract
→ Elixir backend → transaction builder → LiveView UI. After each slice, the feature is
complete and demo-able end-to-end.

### Player vs Tribe Model

Sigil works for **both solo players and tribes** — the same infrastructure serves
either use case, with tribe features only appearing when relevant.

**Solo player (no tribe):** Character.tribe_id is nil or zero. Sigil shows only
their own assets. No tribe UI surfaces. They still benefit from assembly monitoring,
fuel forecasting, and personal gate/turret extension management.

**Player in a tribe:** Character.tribe_id links them to a tribe. Sigil surfaces
the aggregate tribe view alongside their personal assets.

**Tribe formation is automatic.** When a player connects their wallet, Sigil reads
their Character's `tribe_id` from chain. If it's non-zero, they're in a tribe — no
explicit "create tribe" step. The tribe exists because the chain says it does. All
Characters sharing the same `tribe_id` are tribe members.

**Tribe member discovery uses lazy loading:**
1. First tribe member connects → Sigil queries chain for all Characters with
   matching `tribe_id` → stores address list
2. Tribe dashboard shows all known members, but assemblies are only populated for
   members who've connected to Sigil
3. Connected members get live polling (StatePoller). Non-connected members show as
   names/addresses only — no stale data pretending to be live
4. Visual distinction between "actively monitored" (connected) and "known from chain"
   (not connected)

**Assembly sharing within tribe is opt-in at the Sigil level.** All on-chain data is
public (anyone can query any address), but Sigil controls what it surfaces in the
tribe aggregate view. A player's assemblies only appear in the tribe dashboard if they've
connected their wallet. This is UX-level privacy, not cryptographic — transparent about
what it is.

**Assembly enrollment = implicit control delegation.** When a member registers their
gate/turret with the tribe's StandingsTable extension, they're subscribing that assembly
to the tribe's diplomatic policy. The StandingsTable dictates access/targeting decisions.
The member retains full ownership (OwnerCap) and can detach the extension anytime.
No separate "TribeDelegateCap" needed for MVP — the StandingsTable subscription model
covers the core story.

**On-chain roles** control who can modify the StandingsTable. The tribe leader (creator
of the StandingsTable) has full authority. Officer/member role capabilities are on-chain
objects that grant graduated permissions. This differentiates from CradleOS's role system
because our roles gate a policy that *enforces itself on-chain* across all enrolled
assemblies — not just UI-level permissions.

**One extension per assembly** (confirmed from world-contracts source): each assembly has
a single `Option<TypeName>` extension slot. Registering Sigil's extension replaces
any previous one. This means our `frontier_gate` / `frontier_turret` must be comprehensive
— all policy logic in one module per assembly type. The extension can be frozen
(`extension_freeze`) to prevent swapping. Sigil UI must clearly communicate:
"This will replace your current gate extension."

---

## Day-by-Day Plan (Vertical Slice Model)

### Schedule Overview (REVISED — reputation scoring system added)

| Days | Phase | What |
|------|-------|------|
| 1-5 (Mar 11-15) | **Core Platform** | Phoenix + Move scaffold, Sui client, BCS, signing, basic dashboard |
| 6-9 (Mar 16-19) | **Slice 1: Gate Access + 5-Tier Upgrade** | Upgrade Move contracts to 5-tier, diplomacy backend with scoring foundation, UI. **Submittable baseline.** |
| 10-12 (Mar 20-22) | **Slice 2: Turret + Storage Extensions** | Move turret extension (graduated targeting) + storage extension (tier-based access) + cascade UI. |
| 13-16 (Mar 23-26) | **Slice 3: Reputation Engine + Monitoring** | Event-driven scoring, transitive reputation, GenServer monitoring, alerts, Discord webhooks. **Primary differentiator.** |
| 17 (Mar 27) | **Slice 4 / Stretch** | Kill feed, scheduled ops, or mobile polish. Cut if behind. |
| 18-21 (Mar 28-31) | **Polish + Submit** | UX polish, demo video, documentation, deploy, submit. |

### Core Platform (Days 1-5, March 11-15)

Shared substrate that all slices build on. Not independently demo-worthy.

**Day 1 (Mar 11, Tue) — Project Bootstrap + Dual Scaffold**
- Morning: `mix phx.new sigil --live`, Postgres, git init
- Afternoon: `sui move new sigil`, study world contract extension interface,
  start `Sigil.Sui.Client` (GraphQL wrapper), join hackathon Discord for package IDs
- Deliverable: Phoenix running locally. Move project compiles. Sui client connects.

**Day 2 (Mar 12, Wed) — BCS Encoder + Data Model**
- Morning: `Sigil.Sui.BCS` — full encoder (u8-u256, ULEB128, bool, string, vector,
  option, address, struct). Test against Sui SDK reference vectors.
- Afternoon: `Sigil.Sui.Schema` — Elixir structs for Move types (Assembly, Gate,
  Turret, Character, NetworkNode). ETS cache as primary data layer.
- Deliverable: Complete BCS encoder (tested). Domain structs parse from Sui responses.

**Day 3 (Mar 13, Thu) — Transaction Building + Signing**
- Morning: `Sigil.Sui.Transaction` — PTB construction, MoveCall helpers, digest
- Afternoon: `Sigil.Sui.Signer` — Ed25519 via `:crypto`, intent signing.
  Extend Client with `execute_transaction/2`. E2E test on testnet.
- Deliverable: Can construct, sign, and submit a real Sui transaction from pure Elixir.

**Day 4 (Mar 14, Fri) — Player Dashboard + Assembly Views**
- Morning: `Sigil.Accounts` (wallet session via Sui address, cached in ETS), `/dashboard` LiveView
  (OwnerCap → assemblies, type/status/location/fuel display)
- Afternoon: `/assembly/:id` detail views per type. `GameState.Poller` GenServer
  for periodic re-fetch + PubSub broadcast. LiveViews auto-update.
- Deliverable: Enter wallet → see assemblies → drill into detail → real-time updates.

**Day 5 (Mar 15, Sat) — Core Hardening + Move Contract Design**
- Morning: Bug fixes, error handling, Sui client hardening (retry, pagination, rate limits)
- Afternoon: Design `StandingsTable` + `frontier_gate` Move contracts. Begin Move tests.
- Deliverable: Core platform solid. Move contract design documented, tests started.

### Slice 1: Diplomatic Gate Access + 5-Tier Upgrade (Days 6-9, March 16-19)

The flagship feature. Set a tribe's standing → all gates enforce on-chain. Now with
5-tier graduated standings and configurable default policy.

**Day 6 (Mar 16, Sun) — Move Contract Upgrade + Deploy** ✅ DONE
- ~~`standings_table.move`: create, set_standing, get_standing, batch_set_standings~~ DONE (3-tier)
- ~~`frontier_gate.move`: FrontierGateAuth witness, request_permit~~ DONE (3-tier)
- ~~Upgrade StandingsTable: expand to 5-tier (0-4), add `default_policy` field (NBSI/NRDS),
  add address-keyed entries for individual pilot standings, add pin/override flag~~ DONE
- ~~Upgrade frontier_gate: threshold-based allow/deny instead of hostile-only deny,
  respect default_policy for neutral tier~~ DONE
- ~~`sui client publish` to testnet~~ DONE
- Deliverable: 5-tier Move contracts deployed with configurable default policy.
- **Published Package ID (Utopia testnet):** `0x06ce9d6bed77615383575cc7eba4883d32769b30cd5df00561e38434a59611a1`
- **UpgradeCap:** `0x06aa8398840d46b7c0cf483d265c1c536c5d2924bf214f1af2e58a00d5938b90`
- **Tx Digest:** `GisQ6auY6rJigWW79DtkUvu6vv8c4qwxhhiAk4TncCYJ`

**Day 7 (Mar 17, Mon) — Diplomacy Backend + Transaction Builders**
- `Sigil.Diplomacy` context:
  - `list_standings(table_id)` — read tiers from chain via GraphQL
  - `set_standing(table_id, tribe_id, tier, signer)` — build + sign + submit tx
  - `batch_set_standings(table_id, entries, signer)` — bulk update
  - `pin_standing(table_id, tribe_id, tier)` — manual override
  - ETS cache of standings + scores, PubSub broadcast on changes
  - **Scoring foundation**: score storage (ETS), threshold config, tier computation
- `Sigil.Sui.TransactionBuilders.Diplomacy`:
  - `build_set_standing_tx/3`, `build_deploy_gate_extension_tx/2`,
    `build_authorize_extension_tx/2`
- Integration test: set standing from Elixir → verify on-chain
- Deliverable: Diplomatic standings managed from Elixir. Scoring data model in place.

**Day 8 (Mar 18, Tue) — Diplomacy + Gate Management UI**
- `/tribe/:id/diplomacy` — standings editor:
  - List tribes with 5-tier selector (Hostile/Unfriendly/Neutral/Friendly/Allied)
  - Color coding: red/orange/gray/green/blue for standing levels
  - Score display alongside tier (computed score + current tier + threshold info)
  - Pin/override toggle per tribe
  - "Save" builds + submits transaction, shows status (pending → confirmed)
- Gate pages: "Authorize Sigil extension" button, extension status, default policy toggle
- Bulk: "Apply standings to ALL gates" with progress tracking
- Deliverable: Full UI workflow. Set standings → gates enforce → on-chain.

**Day 9 (Mar 19, Wed) — Slice 1 Polish = SUBMITTABLE BASELINE**
- Error handling, transaction history, loading states, optimistic updates
- README, Fly.io deployment (first public deploy), full flow test
- Deliverable: **SUBMITTABLE.** Pure Elixir Sui integration + 5-tier on-chain diplomacy +
  real-time LiveView. Deployed to Fly.io. Eligible for: Utility, Technical, Live Frontier Integration.

### Slice 2: Turret + Storage Extensions (Days 10-12, March 20-22)

Same StandingsTable controls turrets AND storage. One diplomacy change → all infrastructure.
Now with 5-tier graduated behavior per assembly type.

**Day 10 (Mar 20, Thu) — Turret Extension + Backend**
- `frontier_turret.move`: `FrontierTurretAuth` witness, `get_target_priority_list()` that
  unpacks BCS-encoded `TargetCandidate` list, reads `character_tribe` from each candidate,
  checks StandingsTable tier, returns graduated `priority_weight`:
  - Hostile: 10000 (shoot on sight)
  - Unfriendly: 5000 if aggressor, 1000 if entering range
  - Neutral: 5000 only if aggressor
  - Friendly: 2000 only if aggressor (benefit of doubt)
  - Allied: 0 (never target)
  Must consume `OnlineReceipt` hot potato.
- `frontier_storage.move`: `FrontierStorageAuth` witness, controls deposit/withdraw:
  - Allied: full access (deposit + withdraw)
  - Friendly: deposit only
  - Others: deny
- Deploy both to testnet. Transaction builders for authorize + manage.
- Deliverable: Turret + storage extensions deployed. Full 5-tier cascade across 3 assembly types.

**Day 11 (Mar 21, Fri) — Cascade UI + Unified Management**
- Turret pages: authorize extension, bulk apply
- Storage pages: authorize extension, access policy display
- Enhanced diplomacy page: "This affects 5 gates, 3 turrets, and 2 storage units."
  "Apply to ALL infrastructure" — one click. Per-assembly-type behavior preview.
- Deliverable: One standing change → ALL gates deny + ALL turrets target + ALL storage restricts.

**Day 12 (Mar 22, Sat) — Slice 2 Polish + Buffer**
- Fix bugs, cascade UX, transaction batching. Update README + deployment.
- Deliverable: Slices 1-2 production-quality. 5-tier cascade across 3 assembly types.

### Slice 3: Reputation Engine + Monitoring (Days 13-16, March 23-26)

Elixir-native. No new Move contracts. Showcases OTP — the "why Elixir" argument.
**This slice is our primary competitive differentiator vs CradleOS.** Everything here
is impossible in a browser-only SPA. The reputation scoring engine is the crown jewel.

**Day 13 (Mar 23, Sun) — GenServer-per-Assembly Monitoring + Event Ingestion**
- `GameState.AssemblyMonitor` — GenServer per assembly:
  - Polls state from Sui on configurable interval
  - Detects changes (status, fuel, extensions), broadcasts via PubSub
  - **Fuel burn rate tracking**: accumulates historical data, computes depletion forecast
- `GameState.MonitorSupervisor` — DynamicSupervisor, one-for-one restart strategy
- **Event ingestion pipeline**: poll KillmailCreatedEvent, JumpEvent from chain.
  These feed the reputation scoring engine (Day 14).
- Deliverable: Every assembly has a dedicated GenServer. Event stream flowing.

**Day 14 (Mar 24, Mon) — Reputation Scoring Engine**
- `Sigil.Diplomacy.ReputationEngine` — the differentiator:
  - Score computation: event-driven accrual (kills=-200, gate usage=+5, trade=+10, defense=+100)
  - Time decay toward neutral (~5/day)
  - **Transitive reputation**: weighted "enemy of my friend" across tribe graph
  - Threshold mapping: score → tier, configurable per tribe leader
  - On-chain push: update StandingsTable only when tier boundary crossed
  - Manual pin/override: skip automatic scoring for pinned entries
- Wire into diplomacy UI: show computed scores alongside tiers, threshold visualization
- Deliverable: Standings evolve automatically from observed behavior. Transitive reputation live.

**Day 15 (Mar 25, Tue) — Alerts + Discord Webhooks + Ops Timeline**
- `Sigil.Alerts` + `Alerts.RuleEngine`:
  - Rules: fuel below threshold, assembly offline, extension removed/changed,
    **standing change alert** ("Tribe 7 crossed hostile threshold")
  - Triggered by AssemblyMonitor + ReputationEngine PubSub broadcasts
- `Sigil.WebhookNotifier`: Discord webhook delivery
  - **KEY DEMO MOMENT**: close browser, get Discord ping when standing changes
- `UI_OpsTimeline`: Real-time event feed (ETS-backed rolling buffer)
- Deliverable: Proactive alerts + persistent event history. Mission control feel.

**Day 16 (Mar 26, Thu) — Tribe View + Multi-User Polish**
- `/tribe/:id` overview — members, stats, standings summary with scores, monitoring status
- Reputation graph visualization: tribe relationships with score-colored edges
- Multi-user real-time demo prep: two browser windows showing synchronized state
- Deliverable: Tribe leaders see everything at a glance. Reputation graph demo-ready.

### Slice 4: Strategic Map + Diplomacy-Aware Routing (Day 17, March 27)

**The centerpiece feature.** This is where Sigil's diplomacy system becomes spatially tangible
and where we offer something no existing tool provides: **standings-aware route planning.**

- **Interactive star map** (Three.js point cloud of 24k systems from World API):
  - Minimal Three.js renderer: `BufferGeometry` point cloud from `/v2/solarsystems` coordinates,
    `OrbitControls` for pan/zoom/rotate, raycaster for click → system selection
  - LiveView JS hook bridges clicks (`pushEvent("system_selected", %{system_id: id})`) and
    receives highlights (`handleEvent("highlight_systems", ...)`) for two-way integration
  - **Why build vs embed EF-Map**: EF-Map's embed API is one-directional (no click events back).
    Our own renderer gives two-way interaction, custom overlays tied to our domain data, and no
    dependency on a closed-source external service
  - Scope: ~200-400 lines of JS (Three.js does the heavy lifting), one LiveView hook, one
    LiveView page. System coordinate data already available via StaticData module.

- **Diplomacy-aware route planning** (unique to Sigil — EF-Map does NOT do this):
  - Index ALL gates on-chain (shared objects, publicly queryable regardless of who owns them)
  - Read gate link relationships, fuel status, online/offline, installed extensions
  - For gates using Sigil's `frontier_gate` extension: read the StandingsTable to determine
    access policy for the current player's tribe
  - Route planner shows: "3 jumps from A to B. Gate 1: safe (operator Allied with you).
    Gate 2: safe (operator Friendly). Gate 3: **access unknown** (no Sigil extension)."
  - Color-coded gate corridors on the map: green=safe, red=hostile, gray=unknown policy
  - **No adoption dependency for the passive layer**: gate locations, link topology, fuel state,
    and online status are all public chain data. The map works for everyone. Diplomacy resolution
    only works for gates using Sigil's extension — "unknown" is honest and useful.

- **Tribe infrastructure overlay**:
  - Systems containing tribe assemblies glow a different color
  - Click to see "3 gates, 1 turret in this system"
  - Diplomacy overlay: link lines between allied/hostile territory color-coded by standing tier

**Why this matters for the submission:**
- Directly counters CradleOS's starmap advantage, but integrated with diplomacy data
- No other tool offers standings-aware routing — genuinely novel gameplay utility
- High visual demo impact with substantive functionality underneath

### Stretch Features (cut if behind)
- Kill feed (KillmailPoller + live feed UI)
- Scheduled operations ("bring all gates online at 18:00 UTC")
- Mobile-responsive polish (CradleOS's dense UI is unusable on phones)

### Polish + Submit (Days 18-21, March 28-31)

**Day 18 (Mar 28)** — UX polish: EVE-themed dark UI, loading/error/empty states, landing page
**Day 19 (Mar 29)** — Demo video (3-5 min), README polish, architecture diagram
  Demo script:
  1. Connect wallet, see assemblies (30s)
  2. Set diplomatic standings → cascade to gates + turrets + storage (60s)
  3. Show on-chain enforcement — deny hostile at gate, graduated turret targeting (60s)
  4. Reputation scoring: show how kills/gate usage shift scores, threshold crossing
     triggers automatic tier change + on-chain update (60s)
  5. Transitive reputation: "enemy of my ally auto-detected" (30s)
  6. Monitoring + alerts: Discord ping on standing change (30s)
  7. Strategic map: click a system, see tribe assets. Plan a route — "safe / hostile /
     unknown" per gate. Change a standing, watch route safety recolor in real time (45s)
  8. "Built in pure Elixir, no JavaScript SDK" (30s)
**Day 20 (Mar 30)** — Final Fly.io deploy, E2E testing, security review
**Day 21 (Mar 31)** — Submit by noon on GitHub + DeepSurge, fix last issues, ensure stability

---

## Known Research Gaps

### RESOLVED (pre-hackathon research)

| Gap | Resolution |
|---|---|
| **Gate/turret extension interface** | **Fully documented above.** Gate: permit-based (issue JumpPermit, NOT allow/deny). Turret: BCS-encoded I/O (TargetCandidate → ReturnTargetPriorityList), OnlineReceipt hot potato. |
| **Event types** | **27 event types catalogued** in Real-Time Data section. Key: StatusChangedEvent, FuelEvent, JumpEvent, KillmailCreatedEvent, ExtensionAuthorizedEvent. |
| **`primitives/` module** | **Location is Poseidon2 hash** (32 bytes, no raw x/y/z). Metadata = {name, description, url}. Fuel = {max_capacity, burn_rate_in_ms, quantity, is_burning, ...}. EnergySource = {max/current/reserved energy}. AssemblyStatus = enum {NULL, OFFLINE, ONLINE}. |
| **`access/` module details** | **Fully documented.** OwnerCap<T> = {id, authorized_object_id}. AdminACL = {id, authorized_sponsors: Table}. GovernorCap = {id, governor: address}. ServerAddressRegistry for location proof verification. Borrow/return pattern with hot-potato ReturnOwnerCapReceipt. |
| **Public entry functions** | **All `public fun` signatures documented** (no `entry` keyword used in this codebase). Full list available in research notes. |
| **StorageUnit full struct** | **Documented** in Queryable Objects section. Has `inventory_keys: vector<ID>` for dynamic field access, plus `extension: Option<TypeName>`. |
| **`sui` CLI availability** | **Recommended: `suiup`** version manager. Latest: testnet-v1.67.1 (March 3, 2026). Also available via cargo or pre-built binary. Prerequisites: `curl git cmake gcc libssl-dev pkg-config libclang-dev libpq-dev build-essential`. |
| **`@evefrontier/dapp-kit`** | **v0.1.5.** TypeDoc live at `http://sui-docs.evefrontier.com/`. Not on public npm — bundled in builder-scaffold. Irrelevant to us (TypeScript/React) but TypeDoc is useful as API reference. |
| **Blake2b for digests** | **RESOLVED.** Erlang `:crypto` only does Blake2b-512 (truncating gives wrong results). Use `{:blake2, "~> 1.0"}` — `Blake2.hash2b(data, 32)` for correct Blake2b-256. |

### RESOLVED (Day 1 research)

| Gap | Resolution |
|---|---|
| **Which Sui network** | Standard **Sui Testnet** (chain-id `4c78adac`). Move.toml deps reference `testnet-v1.66.2`, deployed with toolchain v1.67.1. No custom fork. |
| **Cycle 5 launch date** | **March 11, 2026** — same day as hackathon. |
| **Static data on Sui** | **NOT on-chain** (no Move modules for static data). Stillness World API still serves static reference data (types, solarsystems, tribes). Some game-state endpoints (killmails) now 404 — mid-migration. Stillness blockchain gateway fully decommissioned. |
| **Gas pool API for third parties** | Self-hosted only (no public CCP endpoint). On testnet, gas is free from faucet — non-issue for hackathon. |
| **Cycle 5 new features on-chain** | Shells/Crowns/Orbital Zones/combat are server-side, NOT on-chain. Nursery = base Assembly (no extensions). Nest = likely StorageUnit variant (has extension support). No new Move modules in public repo. |
| **New docs / World API** | Docs site (`docs.evefrontier.com`) has many 404s — being rebuilt for Sui era. Best source: GitHub `evefrontier/builder-documentation` repo (updated March 10). Key working pages: environment-setup, world-explainer, object-model, move-patterns, build guides for gate/turret/storage. TypeDoc live at `http://sui-docs.evefrontier.com/` (dapp-kit v0.1.5). |
| **`tribe_permit.move`** | **Fully analyzed.** CCP's example checks `character.tribe() == config.tribe` (single u32 match). Only supports ONE tribe per gate. Uses same typed-witness pattern (XAuth) and dynamic fields on shared ExtensionConfig. Our standings-based approach is strictly more powerful (graduated access, multi-tribe, per-tribe updates). Same architectural patterns — validates our design. Source in both `world-contracts/contracts/extension_examples/` and `builder-scaffold/move-contracts/smart_gate_extension/`. |
| **Package IDs** | **FOUND** in `Published.toml` (Sui automated address management). Four environments on testnet: Stillness (`0x28b4...`), Utopia (`0xd12a...`), Internal (`0x3539...`), Generic (`0x920e...`). Full IDs in "Deployed Package IDs" section above. |
| **Utopia server** | Utopia World API URL does NOT resolve (DNS failure). Utopia appears to be accessed via **Sui testnet RPC directly** using the Utopia package ID, not through a dedicated World API. |
| **One extension per assembly** | Each assembly supports only ONE extension at a time — attaching a new one replaces the old. Our `frontier_gate` must handle all access logic in a single extension. |
| **world-contracts v0.0.18** | Latest release (March 11). New: option to publish location on-chain (#129), freeze extension config (#132), new open inventory (#133). |
| **Local dev environment** | `efctl env up` or Docker compose in builder-scaffold gives full offline dev with seeded test data (Character, Network Node, SSU with items, two linked Gates). No game client needed. |

### ALL RESEARCH GAPS RESOLVED

No remaining open questions. All information needed for development is documented above.

## Risk Mitigation

| Risk | Mitigation |
|---|---|
| Move contract bugs | Thorough `sui move test`. Deploy to testnet first. Conservative defaults (deny-if-error). Publish with upgrade capability. |
| Extension interface complexity | Gate is permit-based (two-step: issue permit, then jump). Turret uses BCS-encoded I/O with hot potato receipt. Both patterns are well-understood from source and extension examples. |
| Package IDs not immediately available | **RESOLVED.** All four environment IDs found in `Published.toml`. Stillness: `0x28b4...`, Utopia: `0xd12a...`. |
| BCS/signing harder than expected | Full Day 2-3 dedicated. Fallback: `sui keytool sign` CLI shim. Reference vectors provide ground truth. |
| Cycle 5 data issues at launch | Package IDs known. Stillness World API serves static data. Local dev env available via `efctl env up`. Killmails endpoint 404 (mid-migration) — use Sui events instead. |
| Rate limits on public Sui endpoint | ETS caching, configurable polling intervals, Chainstack/Ankr backup. |
| Scope creep after Slice 1 | Slice 2 reuses Slice 1 infrastructure. Slice 3 has no Move dependency. Days 16-17 are explicitly cuttable. |
| Location data is hashed on-chain | On-chain: Poseidon2 hashes (no raw coords). Stillness World API provides solar system x/y/z (universe-map positions, NOT per-assembly positions). CCP intends full location obfuscation on Sui — ZK proximity proofs exist as PoC but aren't deployed yet. Build galaxy-map views from system coords; don't depend on assembly-level location data. |
| Static data (types/systems) not on Sui | **RESOLVED.** World API REST endpoints confirmed live on Stillness with all 6 data types. Ingest and cache in ETS. |
| Gas pool not available to third parties | Core features (extension config, online/offline, metadata, our own Move extensions) work without it. |
| Burnout (21 days) | Days 5, 9, 12 are buffer/catch-up. Days 18-21 are lower cognitive load (polish). |

## What "Done" Looks Like

| After | What You Can Demo | Submittable? | Prize Potential |
|---|---|---|---|
| Core (Day 5) | Pure Elixir reads/writes Sui | No | — |
| **Slice 1 (Day 9)** | **Set hostile → gates deny on-chain** | **Yes** | **Utility + Technical + Live Integration** |
| Slice 2 (Day 12) | One change cascades to ALL infrastructure | Yes | Stronger across all categories |
| Slice 3 (Day 16) | Monitoring + alerts + Discord webhooks + fuel forecasting + ops timeline | Yes | **Technical Excellence** — things CradleOS literally cannot do |
| Slice 4 (Day 17) | Strategic map + diplomacy-aware route planning | Yes | **Unique gameplay utility** — no existing tool does this |
| Polish (Day 21) | Polished, deployed, demo'd | Yes | Best shot at Overall + Technical Excellence |

---

## Emergent Systems Layer (DECIDED)

The hackathon theme says entries can range from "practical survival tools to experimental,
emergent systems." Sigil IS the emergent system — the on-chain Move extensions create
emergent behavior:

- **Diplomatic cascade**: A single standing change in Sigil propagates to all gates
  (deny access) and all turrets (target ships) simultaneously. The on-chain contracts
  enforce this without Sigil needing to be online — the enforcement IS the blockchain.
- **Infrastructure-as-code**: Players declare intent ("tribe X is hostile") and the on-chain
  system enforces it across all infrastructure. This is declarative, not imperative.
- **Composability**: Other tools can read Sigil's StandingsTable. Other extensions can
  build on it. The on-chain data becomes shared infrastructure for the ecosystem.

### Bounty System (Stretch — Slice 4 candidate)

**Feasibility: CONFIRMED.** Killmail is the verification primitive.

- Player A posts a bounty: "Kill Player B, reward X EVE tokens"
- Tokens escrowed in a Move contract
- Player C kills Player B → CCP's server creates a `Killmail` on-chain (unforgeable)
- Contract verifies: `killmail.victim_character_id == bounty.target_character_id` → pays out
- No oracle needed — Killmail is a server-created Sui object, cryptographically tied to the kill

**What's verifiable**: killer, victim, solar system, timestamp. **Not verifiable**: ship type
(not in Killmail — only `loss_type: LossType` distinguishing SHIP vs STRUCTURE).

Move contract: ~100 lines (bounty struct, escrow, killmail verification, payout).

### Intel Market — "Scouting as a Job" (Stretch — differentiator feature)

A reputation-based intelligence marketplace where tribe members can earn income by scouting.
This would be a novel game system that has never existed in EVE.

**Design (same-tribe, Sigil as intermediary):**

1. **Scout submits a report** with two fields:
   - **Public teaser** (visible to all tribe members): "Enemy mining fleet, Amarr frigates, high value"
   - **Hidden details** (encrypted, access-gated): "System 0xABC, near gate to 0xDEF, 3 barges + 1 escort"
   - Sigil encrypts the hidden field (AES), stores key in Postgres, posts ciphertext
     on-chain for tamper-proofing (hash verifiable, content can't be altered after posting)
   - Scout sets a price in EVE tokens

2. **Buyer purchases the report** — payment on-chain (EVE token transfer), Sigil
   detects payment event and reveals decrypted content in the UI

3. **Bidirectional thumbs up/down ratings:**

   | Outcome | Scout gets | Buyer gets |
   |---|---|---|
   | Buyer thumbs up | Payment + thumbs up | Thumbs up (reliable buyer) |
   | Buyer thumbs down | Payment + thumbs down | Neutral |
   | Buyer does nothing (timeout) | Payment + neutral | Thumbs down (unreliable) |

   - **Buyer never gets a refund.** Payment is final. Reputation handles quality control.
   - **Scouts can set a rep floor** for buyers — prevents troll accounts from buying + downvoting.
     High-rep buyers get access to premium scouts.

4. **Scout anonymity**: Sigil posts all reports from its platform wallet. Scout identity
   exists only in Postgres (visible to tribe leadership, not on-chain). See "Sui Address
   Transparency" section — direct on-chain posting would unmask scouts.

5. **Solo/mercenary scouts**: Tribes can post bounty boards ("We pay for intel on systems X, Y, Z").
   Freelance scouts submit reports, get paid from tribe treasury. Sigil attracts talent —
   tribes without it rely on Discord. This reinforces the tribe value proposition.

**On-chain footprint**: Token transfers + integer reputation counters. Content lives in Postgres.
Move contract: ~100 lines (report listing with hash + price, purchase via token transfer, rate).

**Killmail as optional strong review**: Buyer can submit a Killmail as proof the intel led to a
kill. Contract verifies system/target/timestamp match against the report. Strongest possible
"thumbs up" — cryptographic proof something died where the scout said to look. But it's
optional, not required — the reputation system works without it.

**Why this is compelling for judges**: It's not just a dashboard — it's an emergent economic
system that creates a new player role (professional scout) enabled by the blockchain payment
layer and Sigil's identity/anonymity infrastructure.

### Future stretch ideas (post-hackathon or if time permits):
- **Adaptive defense**: turret configs that evolve based on observed attack patterns

## Reputation Scoring System (DECIDED)

The StandingsTable stores simple tier values (u8) on-chain, but the intelligence behind
those tiers lives in Elixir. Scores are computed off-chain and pushed on-chain only when
a tier threshold changes — minimizing gas costs while enabling rich reputation dynamics.

### Five-Tier Standings (replaces 3-tier hostile/neutral/friendly)

Each assembly type interprets tiers differently:

| Tier | Value | Gate | Turret | Storage |
|------|-------|------|--------|---------|
| **Hostile** | 0 | Deny | Shoot on sight (priority 10000) | Deny all |
| **Unfriendly** | 1 | Deny | Shoot if aggressor or entering range (5000/1000) | Deny all |
| **Neutral** | 2 | *Configurable default* | Shoot only if aggressor (priority 5000) | Deny |
| **Friendly** | 3 | Allow | Don't shoot unless attacking you (priority 2000 if aggressor) | Deposit only |
| **Allied** | 4 | Allow | Never target | Full access (deposit + withdraw) |

Key behavioral distinctions:
- **Hostile vs Unfriendly**: turrets shoot hostile on sight; unfriendly only when suspicious
- **Neutral**: tribe's **default policy** (NBSI=deny, NRDS=allow) determines gate behavior.
  Turrets only engage aggressors. Most tribes will use deny-by-default for gates.
- **Friendly vs Allied**: storage is the difference. Friendly can drop off trade goods
  but not take. Allied get full warehouse access. Turrets never target allied even if
  aggressing (benefit of the doubt — could be accidental friendly fire)

### Scoring Model (off-chain, in Elixir)

**Score range:** -1000 to +1000, computed per tribe (or per individual for tribeless pilots)

**Default threshold mapping (configurable by tribe leader):**

| Score Range | Tier |
|-------------|------|
| <= -500 | Hostile (0) |
| -499 to -100 | Unfriendly (1) |
| -99 to +99 | Neutral (2) |
| +100 to +499 | Friendly (3) |
| >= +500 | Allied (4) |

Tribe leaders tune thresholds to match their playstyle. Paranoid tribes set hostile at
-200. Diplomatic tribes narrow the hostile band to -800 and below.

### Events That Feed Into Scores

**Negative (decrease score):**
- Killed one of our members (KillmailCreatedEvent): -200
- Attacked our assemblies — aggressor on grid (TargetCandidate.is_aggressor): -100
- Association with our enemies (transitive computation): -50 weighted

**Positive (increase score):**
- Used our gates peacefully (JumpEvent): +5 (small, accumulates — regular traffic builds trust)
- Traded at our storage — deposit (on-chain deposit events): +10
- Defended our assets — killed an aggressor on our grid (KillmailCreatedEvent): +100

**Passive:**
- Time decay toward neutral: drift toward 0 at ~5/day (grudges fade, friendships need maintenance)

### Transitive Reputation ("Enemy of my friend")

The primary differentiator. For each tribe pair, we compute:

    transitive_adjustment(target) = sum over known tribes T of:
        our_score(T) * T_score(target) * weight / normalization

In practice: if Tribe A is our close ally (+800) and Tribe A considers Tribe C hostile
(-900), that pulls our score for Tribe C down. Strong allies' opinions matter more
than acquaintances'. This runs periodically in Elixir (every few minutes) and pushes
one on-chain transaction when a tier boundary is crossed.

### Manual Override ("Pin")

Tribe leader can pin any tribe or pilot to a fixed tier. Pinned entries skip automatic
scoring entirely. Use cases:
- "We're at war with Tribe 7" — pin Hostile regardless of computed score
- "Tribe 42 is our founding ally" — pin Allied permanently
- "This specific pilot betrayed us" — pin individual to Hostile

### Individual Pilot Standings

**Lookup order in gate/turret/storage extensions:**
1. Check individual standing by pilot address (if entry exists)
2. Fall back to tribe standing by tribe_id
3. Fall back to default policy

**Tribeless pilots:** Individual entries computed from personal actions. Same scoring
events, keyed by address instead of tribe_id.

**Tribe members:** Inherit tribe tier. Individual overrides only via manual pin.
No automatic per-pilot scoring for tribe members at hackathon scope.

### On-Chain vs Off-Chain Split

| Layer | What Lives Here | Why |
|-------|----------------|-----|
| **Move (on-chain)** | Tier values (u8), default policy, pin flags | Enforcement. Cheap, rarely updated. |
| **Elixir (off-chain)** | Numeric scores, event history, transitive computation, decay | Intelligence. Free to compute. |

The StandingsTable contract changes are minimal: u8 tiers stay, add `default_policy`
field, expand to support address-keyed entries alongside tribe-keyed ones. Gate/turret/
storage extensions check against tiers. All the "smart" behavior lives in Elixir.

## Move Contract Security

- **Permission checks**: Only StandingsTable owner (+ role capability holders) can modify standings
- **Default behavior**: Unknown tribes = Neutral (tier 2). Gate default policy is configurable
  per-StandingsTable (NBSI=deny or NRDS=allow). Turrets only engage neutral aggressors.
- **Five-tier system**: Hostile(0)/Unfriendly(1)/Neutral(2)/Friendly(3)/Allied(4) —
  each assembly type interprets tiers with graduated behavior
- **No panic paths**: Handle edge cases gracefully, no unexpected aborts
- **Shared object contention**: Design for concurrent updates (idempotent operations)
- **Upgrade capability**: Deploy with upgrade cap so contracts can be fixed during hackathon

## Key Decisions Already Made

1. **Vertical slice model** — each feature cuts through all layers (Move → Elixir → UI)
2. **Move smart contracts are first-class** — not a stretch goal, part of Slice 1
3. **On-chain standings** — diplomacy enforced by Move contracts, not just local DB
4. **External tool + on-chain extensions** — eligible for Live Frontier Integration prize
5. **Pure Elixir** — no Node.js sidecar, all BCS/signing in Elixir
6. **Full TDD** — Elixir: ExUnit with strict phase discipline. Move: `sui move test`
7. **Fly.io deployment** — free tier, first deploy at Slice 1 completion (~Day 9)
8. **Tribe coordination** as core value prop — fills the biggest gap in the ecosystem
9. **Submittable after Slice 1 (Day 9)** — not just a read-only dashboard, a complete
   end-to-end diplomatic gate access system with on-chain enforcement
15. **Reputation scoring system** — 5-tier standings (Hostile/Unfriendly/Neutral/Friendly/
    Allied) with off-chain scoring engine. Scores computed in Elixir from chain events
    (kills, gate usage, trade). Transitive reputation ("enemy of my friend"). On-chain
    tiers updated only when thresholds crossed — minimal gas. Primary differentiator vs CradleOS.
10. **Dual-use: solo + tribe** — same platform serves both. Tribe features surface
    automatically when Character.tribe_id is present, hidden otherwise
11. **Automatic tribe formation** — no "create tribe" step. Tribe membership comes from
    chain (Character.tribe_id). Lazy discovery of other tribe members
12. **Universal Move contracts** — one published package for all tribes. Per-tribe
    StandingsTable instances are the parameterization, not separate contract deployments
13. **On-chain roles** — who can modify StandingsTable is enforced on-chain via capability
    objects, not off-chain UI permissions
14. **Enrollment as delegation** — members subscribe assemblies to tribe's StandingsTable
    extension. No TribeDelegateCap for MVP. Members retain full OwnerCap ownership
16. **Build own star map vs embed EF-Map** — EF-Map's embed API lacks click events back
    to host page (one-directional postMessage only). Building a minimal Three.js point
    cloud (~200-400 lines JS) gives two-way LiveView integration, custom tribe/diplomacy
    overlays, and no closed-source dependency. System coordinates come from World API
    via our existing StaticData module. Stretch goal (Slice 4) — cut if behind schedule.
