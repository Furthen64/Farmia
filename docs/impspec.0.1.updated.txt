// Farming Simulator – Implementation Specification 0.1 (Simplified)

## 1.  Project Overview
This document expands on *spec.0.1.txt* by detailing the concrete classes, interfaces, data‑structures, and runtime behaviour that will be used to build the 2‑D top‑down farming simulator.  
All code is written in **C# 12** (ASP.NET 8) for the server and **TypeScript** for the client.  
The repository is split into three logical layers:

| Layer | Responsibility | Key Artifacts |
|-------|----------------|---------------|
| **Domain** | Pure business logic – entities, value objects, aggregates | `Domain/` |
| **Infrastructure** | Persistence, SignalR, timers, caching | `Infrastructure/` |
| **Presentation** | ASP.NET 8 Web API + SignalR Hub | `Presentation/` |
| **Client** | Browser SPA (React + Redux Toolkit) | `Client/` |

---

## 2.  Domain Layer

### 2.1  Core Entities

| Entity | Key Properties | Behaviour |
|--------|----------------|-----------|
| `Player` | `Id`, `Username`, `Position (x,y)`, `Inventory`, `FarmId` | `MoveTo`, `Plant`, `Harvest`, `SellItem` |
| `Farm` | `Id`, `OwnerId`, `Tiles[800][400]` | `GetTile`, `SetTile` |
| `Tile` | `X`, `Y`, `Ground`, `Surface`, `Lifeforms`, `Version` | `IsWalkable`, `AddEntity`, `RemoveEntity` |
| `Crop` | `Id`, `Type`, `Stage`, `PlantedAt`, `WateredCount`, `WaterNeeded` | `Water`, `AdvanceStage`, `IsReady` |
| `MarketItem` | `Id`, `Name`, `Price`, `Quantity` | `UpdatePrice`, `Buy`, `Sell` |

### 2.2  Value Objects

* `Position` – immutable struct with `X`, `Y`.  
* `TileCoordinate` – same as `Position` but used for indexing.  
* `CropStage` – enum (`Seed`, `Sprout`, `Mature`, `Ready`).  

### 2.3  Aggregates & Repositories

* `PlayerAggregate` – owns `Player`, `Farm`, `Inventory`.  
* `CropAggregate` – owns `Crop` and its timers.  

Repositories are defined as interfaces:

```csharp
public interface IPlayerRepository
{
    Task<PlayerAggregate> GetByIdAsync(Guid id);
    Task SaveAsync(PlayerAggregate player);
}

public interface IFarmRepository
{
    Task<Farm> GetByIdAsync(Guid id);
    Task SaveAsync(Farm farm);
}
```

Concrete implementations use EF Core for PostgreSQL (players, inventory, market) and SQLite for the world map.

---

## 3.  Infrastructure Layer

### 3.1  Persistence

| Table | Columns | Notes |
|-------|---------|-------|
| `Players` | `Id`, `Username`, `PositionX`, `PositionY`, `FarmId` | |
| `Farms` | `Id`, `OwnerId` | |
| `WorldMap` | `X`, `Y`, `Ground`, `Surface`, `Lifeforms`, `Version` | Stored in SQLite; primary key `(X,Y)` |
| `Crops` | `Id`, `FarmId`, `X`, `Y`, `Type`, `Stage`, `PlantedAt`, `WateredCount` | |
| `MarketPrices` | `ItemId`, `Price`, `LastUpdated` | |
| `Transactions` | `Id`, `PlayerId`, `ItemId`, `Quantity`, `Price`, `Timestamp` | |

EF Core `DbContext` contains `DbSet<T>` for each table.  
SQLite connection string is embedded in `appsettings.Development.json`.  
PostgreSQL connection string is read from environment variables.

### 3.2  Tile Caching

* **Server‑side**: `MemoryCache` with a sliding expiration of 5 min.  
* **Client‑side**: IndexedDB store `tiles` keyed by `"x:y"`; fallback to in‑memory Map.

### 3.3  Timer & Bucket Engine

Implemented in `Infrastructure/Timers/WorldTimer.cs`.

```csharp
public class WorldTimer
{
    private readonly Timer _timer;
    private readonly ConcurrentDictionary<long, List<IWorldEntity>> _buckets;
    private long _currentTick;

    public WorldTimer(TimeSpan tickInterval)
    {
        _timer = new Timer(OnTick, null, tickInterval, tickInterval);
        _buckets = new ConcurrentDictionary<long, List<IWorldEntity>>();
        _currentTick = 0;
    }

    private void OnTick(object? state)
    {
        try
        {
            var tick = Interlocked.Increment(ref _currentTick);
            if (_buckets.TryRemove(tick, out var entities))
            {
                foreach (var entity in entities)
                    entity.Update(tick);
            }
        }
        catch (Exception ex)
        {
            // Log and continue; prevents silent timer shutdown
            Console.Error.WriteLine($"WorldTimer error: {ex}");
        }
    }

    public void Schedule(IWorldEntity entity, long nextTick)
    {
        _buckets.AddOrUpdate(nextTick,
            _ => new List<IWorldEntity> { entity },
            (_, list) => { list.Add(entity); return list; });
    }
}
```

* `IWorldEntity` exposes `Update(long tick)`; crops implement this to advance stages.  
* The bucket key is a monotonic tick number (increments once per global tick).  
* Persistence of updated state is triggered inside `Update` via repository calls.

### 3.4  SignalR Hub

`Infrastructure/SignalR/GameHub.cs`:

```csharp
[Authorize]
public class GameHub : Hub
{
    private readonly IPlayerRepository _playerRepo;
    private readonly IFarmRepository _farmRepo;
    private readonly IMarketService _marketService;

    public GameHub(IPlayerRepository playerRepo, IFarmRepository farmRepo, IMarketService marketService)
    {
        _playerRepo = playerRepo;
        _farmRepo = farmRepo;
        _marketService = marketService;
    }

    public async Task JoinGame()
    {
        var userId = Guid.Parse(Context.User?.Identity?.Name ?? throw new InvalidOperationException("Unauthenticated"));
        var player = await _playerRepo.GetByIdAsync(userId);
        await Groups.AddToGroupAsync(Context.ConnectionId, player.FarmId.ToString());
        await Clients.Caller.SendAsync("Initialize", player);
    }

    public async Task Move(int dx, int dy)
    {
        // Update position, clamp to world bounds, broadcast to group
    }

    public async Task PlantCrop(string type, int x, int y)
    {
        // Validate, create Crop, schedule timer, broadcast
    }

    public async Task HarvestCrop(int x, int y)
    {
        // Validate ready, remove crop, add to inventory, broadcast
    }

    public async Task SellItem(string itemId, int quantity)
    {
        // Update market, transaction log, broadcast
    }

    // Server pushes:
    // - CropGrowthUpdate
    // - MarketPriceUpdate
    // - TileChanged
}
```

All push messages are strongly typed JSON contracts defined in `Client/src/contracts.ts`.
#### 3.4.1  Snapshots vs Deltas

To keep bandwidth predictable and client state consistent, all server-to-client updates fall into two categories:

**Authoritative snapshots**
- **On join**: `Initialize` contains an authoritative snapshot of the player aggregate data needed to start the session (player, farm id, inventory summary, and the stored `player_session_position`).
- **On demand**: The client fetches an authoritative **viewport snapshot** via `GET /api/world/tiles?...` whenever it needs to fully resync the visible region.

**Incremental deltas**
- `TileChanged` – a single-tile delta (or small patch) that updates ground/surface/lifeforms and increments `Tile.Version`.
- `CropGrowthUpdate` – delta for a specific crop (stage, watered count, ready state).
- `MarketPriceUpdate` – delta for market prices.
- (Optional) `InventoryChanged` – delta for inventory after harvest/sell/buy.

**Resync rule**
- The client should re-fetch the current viewport snapshot when:
  - the SignalR connection reconnects,
  - it detects a missing update (gap in versions), or
  - it receives a server hint like `ResyncRequired`.



### 3.5  API Controllers

* `PlayerController` – CRUD, login (JWT cookie).  
* `CropController` – Plant, Harvest.  
* `MarketController` – Get prices, Sell.  
* `WorldController` – `GET /api/world/tiles` returns a DTO:

```csharp
public class TileDto
{
    public int X { get; set; }
    public int Y { get; set; }
    public string Ground { get; set; }
    public string Surface { get; set; }
    public List<CropDto> Crops { get; set; }
}
```

The controller fetches tiles from `TileCache` or DB, serialises to JSON, and streams to the client.

---

## 4.  Presentation Layer (ASP.NET 8)

### 4.1  Startup Configuration

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Postgres")));

builder.Services.AddDbContext<WorldDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("World")));

builder.Services.AddSignalR();
builder.Services.AddSingleton<WorldTimer>();
builder.Services.AddScoped<IPlayerRepository, PlayerRepository>();
builder.Services.AddScoped<IFarmRepository, FarmRepository>();
builder.Services.AddScoped<IMarketService, MarketService>();

builder.Services.AddMemoryCache(); // For tile caching
builder.Services.AddSwaggerGen(); // API contract
```

### 4.2  Middleware

* Authentication via cookie (`user=JohnnyPants`).  
* CORS policy for `http://localhost:3000`.  
* HTTPS redirection.

### 4.3  Health Checks

* `/health` returns `200 OK` if DB connections are alive.

---

## 5.  Client Layer (React + Redux Toolkit)

### 5.1  Architecture

```
src/
 ├─ api/          // Axios wrappers for REST endpoints
 ├─ contracts/    // TypeScript interfaces matching DTOs
 ├─ hooks/        // useSignalR, useViewport
 ├─ store/        // Redux slices: player, world, market
 ├─ components/   // TileGrid, CropIcon, MarketPanel
 └─ pages/        // Login, Farm, Market, Profile
```

### 5.2  Rendering

* **Canvas** (`<canvas id="farmCanvas">`) draws tiles in a 2‑D grid.  
* Each tile is 32×32 px; viewport size is computed from zoom level.  
* On `mousemove` + `drag`, the viewport center updates and a new tile request is sent.

### 5.3  Tile Request Flow

```ts
async function loadViewport(x: number, y: number, w: number, h: number) {
    const res = await api.get(`/api/world/tiles?x=${x}&y=${y}&width=${w}&height=${h}`);
    const tiles: TileDto[] = res.data;
    tiles.forEach(t => cacheTile(t));
    renderTiles(tiles);
}
```

* `cacheTile` stores in IndexedDB; if already present, skip fetch.  
* `renderTiles` draws only tiles that intersect the current viewport.

### 5.4  SignalR Integration

```ts
const connection = new HubConnectionBuilder()
    .withUrl("/gameHub")
    .build();

connection.on("CropGrowthUpdate", (crop: CropDto) => {
    store.dispatch(updateCrop(crop));
});

connection.on("MarketPriceUpdate", (prices: MarketPriceDto[]) => {
    store.dispatch(setMarketPrices(prices));
});

connection.onclose(() => {
    // Re‑load viewport on reconnect
    loadViewport(currentX, currentY, currentW, currentH);
});

connection.start();
```

### 5.5  User Actions

* **Plant** – click on a tile, open context menu, send `POST /api/crop/plant`.  
* **Harvest** – click on a ready crop, send `POST /api/crop/harvest`.  
* **Sell** – drag item to market panel, send `POST /api/market/sell`.  

All actions optimistically update Redux state and revert on error.

---

## 6.  Deployment & DevOps

* **No Docker** – run the server with `dotnet run` and the client with `npm run dev`.  
* **CI** – GitHub Actions build the server, run tests, and publish a NuGet package if needed.  
* **Monitoring** – Application Insights for ASP.NET logs, Prometheus + Grafana for timer tick rates, Sentry for client errors.

---

## 7.  Testing Strategy

| Layer | Test Type | Tool |
|-------|-----------|------|
| Domain | Unit | xUnit + Moq |
| Infrastructure | Integration | EF Core InMemory + SQLite |
| Presentation | Unit | Jest + React Testing Library |
| End‑to‑End | Cypress | Browser automation |
| Performance | Load | k6 (simulate 1000 concurrent players) |

---

## 8.  Future Enhancements (Post‑0.1)

1. **Weather System** – global weather state affecting crop growth.  
2. **NPCs & Quests** – new entity types, quest engine.  
3. **Multiplayer Co‑op** – shared farm, lock‑free tile editing.  
4. **Mobile Support** – responsive canvas, touch gestures.  
5. **Mycelium Spread** – new Z‑layer with its own bucket logic.

---

## 9.  Glossary

* **Z‑Layer** – vertical stacking of entities on a tile.  
* **Bucket** – per‑tick list of entities scheduled for update.  
* **Viewport** – subset of world tiles currently visible to the player.  
* **SignalR** – real‑time WebSocket abstraction used for push updates.  

---

**End of impspec.0.1.txt**```