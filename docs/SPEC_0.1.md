# Farmia - a Farming Simulator - specification v0.1


## 1. Overview 

- **Game Type**: 2D top‑down farming simulator.

- **Platforms**: Web browser (client) + ASP.NET 9 webserver (backend).

- **Core Loop**: Players plant crops, harvest, sell, upgrade tools, and expand the farm.

 
## 2. Architecture

Client (any modern browser) <-> SignalR talk to webserver <-> Webserver: [ASP.NET App with a SignalR Hub] [SQLite DB]

- **Authority**: The server is the single source of truth for all world state; the client may apply optimistic UI updates, but must reconcile with server responses and server-pushed events.



- **Server**: Handles world state, user sessions, persistence, and real‑time events via SignalR.

- **Client**: Renders the world, handles input, and communicates with the server via HTTP/WS and SignalR.



### 2.1. Viewport and Rendering

- The world is a fixed grid of **800 × 400 tiles** (each tile 32 × 32 px sprite image).  
- Each player has a **player_session_position** (`x`, `y`) stored on the server. It represents the last tile position they actively edited/worked on (a “focus point”, not necessarily an avatar location). On login, the initial viewport is centered on this position so the player returns to the area they were working on.
- When a player logs in, the server authenticates via a cookie (e.g., `user=JohnnyPants`) and retrieves the player’s last known position.  
- The client renders **only the tiles that fall within the player’s viewport**.  
- Default viewport size is **60 × 40 tiles** (≈ 1920 × 1280 px at 32 px per tile).  
- The viewport is centered on the player’s position, clamped to the world bounds.  
- The server exposes an endpoint `/api/world/tiles?x=100&y=150&width=60&height=40` that returns the tile data for the requested region.  
- The client requests the visible region on initial load and whenever the player moves or zooms.  
- Zoom levels are supported; the default zoom is 1×. Changing zoom changes the number of tiles visible (e.g., 2× zoom shows 30 × 20 tiles).  
- The client caches tiles locally (e.g., in memory or IndexedDB) to avoid re‑fetching unchanged tiles.
- **Authoritative snapshots vs incremental deltas**:
  - **Snapshots**: The viewport tile payload returned from `/api/world/tiles` is an authoritative snapshot for that region at the moment it was fetched.
  - **Deltas**: SignalR pushes incremental updates (e.g., `TileChanged`, `CropGrowthUpdate`, `MarketPriceUpdate`) that update specific tiles/entities without re-sending the full viewport.
  - **Resync rule**: On reconnect, cache/version mismatch, or if the client detects missing updates, the client re-fetches the current viewport via `/api/world/tiles`.
  
- Real‑time updates (crop growth, market changes) are pushed via SignalR and applied only to tiles currently in the viewport.



## 3. Server (C# ASP.NET 9)

### 3.1. Key Components

- **GameWorld**: In‑memory representation of all farms, crops, and entities.

- **PlayerService**: CRUD for player profiles, inventory, and progress.

- **CropService**: Planting, growth timers, harvesting logic.

- **TransactionService**: Marketplace, buying/selling items.

- **WorldMapService**: Handles persistence of the world map to SQLite.

- **SignalRHub**: Push updates to connected clients (e.g., crop growth, market changes) using SignalR.



### 3.2. API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/player/{id} | Retrieve player data |
| POST | /api/player | Create new player |
| POST | /api/crop/plant | Plant a crop |
| POST | /api/crop/harvest | Harvest a crop |
| GET | /api/market | Get current market prices |
| POST | /api/market/sell | Sell items |
| **GET** | **/api/world/tiles** | **Return tiles for a viewport** (query: `x`, `y`, `width`, `height`) |



### 3.3. Persistence

- **Entity Framework Core** with SQLite for the world map and PostgreSQL for other data (players, inventory, transactions, market prices).

- Tables: Players, Farms, Crops, Inventory, Transactions, MarketPrices, WorldMap.



## 4. Client (Browser)

### 4.1. Architecture

- **Pages**: `/login`, `/farm`, `/market`, `/profile`.

- **State Management**: Any client‑side state library (e.g., Redux, Vuex, or plain JavaScript).

- **Real‑time**: SignalR client for live updates.



### 4.2. Rendering

- Use **Canvas** or **SVG** for the 2D top‑down view.

- Tile‑based grid (e.g., 32x32 pixels per tile).

- Simple sprite sheets for crops and tools.



### 4.3. User Flow

1. **Login** → fetch player data.

2. **Farm View** → plant, water, harvest.

3. **Market** → view prices, sell items.

4. **Profile** → view stats, upgrade tools.



## 5. Game Mechanics

### 5.1. Crops

- Types: Wheat, Corn, Tomato, etc.

- Growth stages: Seed → Sprout → Mature → Ready.

- Growth time: Configurable per crop (e.g., 5 minutes in dev, 1 hour in prod).

- Watering: Each crop requires a certain number of water actions per day.



### 5.2. Z‑Layer Architecture

| Z | Layer | Current content | Future possibilities |
|---|-------|-----------------|----------------------|
| 0 | Ground | Soil tiles that host crops. | Mycelium, underground roots, etc. |
| 1 | Surface | Grass, weeds, water patches. | Saplings, vines, etc. |
| 2 | Lifeforms | Animals and other mobile entities. | Insects, birds, etc. |

*All layers are stored in the same world grid; the Z‑value simply determines which entities can occupy a tile.*



### 5.3. Timers & Buckets

- **Global tick**: A single `Timer` (or async loop) fires every **1 second**.

- **Per‑layer bucket**: For each Z‑layer we maintain a `Dictionary<int, List<Entity>>` where the key is the *tick number* (a monotonic counter that increments every global tick) at which the entity’s state should change.

- **Processing**: On each tick we:
  1. Look up the bucket for the current tick.
  2. Advance the state of every entity in that bucket (e.g., crop growth stage).
  3. If the entity is not finished, compute its next tick and re‑insert it into the bucket.
  4. Persist the updated state to the database.

- **Benefits**:  
  * O(1) lookup per tick.  
  * No per‑entity timers or threads.  
  * Easy to extend to new layers (just add another bucket dictionary).




### 5.4. Non‑Goals (v0.1)

The following are explicitly **out of scope** for version 0.1:

- No multiplayer co-op / shared editing / conflict resolution.
- No weather system or seasonal simulation.
- No NPCs, quests, or dialogue systems.
- No combat, enemies, or player damage.
- No offline progression while the server is stopped (no catch-up simulation).

### 5.5. Future Enhancements (Post‑0.1)

- Multiplayer co‑op farming.
- Weather system affecting crop growth.
- NPCs and quests.
- Mycelium spread in Z = 0 (requires a separate bucket and growth logic).



## 6. Deployment

- **Server**: Run locally with `dotnet run`. For production, publish the ASP.NET application and host it on a web server (e.g., Azure App Service, AWS Elastic Beanstalk, or any Linux/Windows server).  
- **Client**: Build with `npm run build` and serve the static assets via a CDN (Vercel, Netlify) or directly from the ASP.NET webserver.  
- **Database**: Use SQLite for the world map (embedded) and PostgreSQL for other data (Azure Database for PostgreSQL, AWS RDS, or any managed PostgreSQL service).



## 7. Testing Strategy

| Layer | Test Type | Tool |
|-------|-----------|------|
| Domain | Unit | xUnit + Moq |
| Infrastructure | Integration | EF Core InMemory + SQLite |
| Presentation | Unit | Jest + React Testing Library |
| End‑to‑End | Cypress | Browser automation |
| Performance | Load | k6 (simulate 1000 concurrent players) |

---

---

## 9. Glossary

* **Z‑Layer** – vertical stacking of entities on a tile.  
* **Bucket** – per‑tick list of entities scheduled for update.  
* **Viewport** – subset of world tiles currently visible to the player.  
* **SignalR** – real‑time WebSocket abstraction used for push updates.  

---

**End of spec.0.1.txt**```
