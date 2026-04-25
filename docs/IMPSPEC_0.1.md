 

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
 

---

## 3.  Infrastructure Layer

### 3.1  Database

| Table | Columns | Notes |
|-------|---------|-------|
| `Players` | `Id`, `Username`, `PositionX`, `PositionY`, `FarmId` | |
| `Farms` | `Id`, `OwnerId` | |
| `WorldMap` | `X`, `Y`, `Ground`, `Surface`, `Lifeforms`, `Version` | Stored in SQLite; primary key `(X,Y)` |
| `Crops` | `Id`, `FarmId`, `X`, `Y`, `Type`, `Stage`, `PlantedAt`, `WateredCount` | |
| `MarketPrices` | `ItemId`, `Price`, `LastUpdated` | |
| `Transactions` | `Id`, `PlayerId`, `ItemId`, `Quantity`, `Price`, `Timestamp` | |

SQLite connection string is embedded in `appsettings.dev.json` and `appsettings.prod.json`
 
### 3.3  Timer & Bucket Engine
 
 
* `IWorldEntity` exposes `Update(long tick)`; crops implement this to advance stages.  
* The bucket key is a monotonic tick number (increments once per global tick).  
* Persistence of updated state is triggered inside `Update` via repository calls.

### 3.4  SignalR Hub

 

```csharp
[Authorize]
public class GameHub : Hub
{
 

      GameHub(  playerRepo,   farmRepo,   marketService)
    {
        _playerRepo = playerRepo;
        _farmRepo = farmRepo;
        _marketService = marketService;
    }

    oinGame()
    {
        var userId = Guid.Parse(Context.User?.Identity?.Name ?? throw new InvalidOperationException("Unauthenticated"));
        var player = await _playerRepo.GetByIdAsync(userId);
        await Groups.AddToGroupAsync(Context.ConnectionId, player.FarmId.ToString());
        await Clients.Caller.SendAsync("Initialize", player);
    }

     Move(int dx, int dy)
    {
        // Update position, clamp to world bounds, broadcast to group
    }

     PlantCrop(string type, int x, int y)
    {
        // Validate, create Crop, schedule timer, broadcast
    }

     HarvestCrop(int x, int y)
    {
        // Validate ready, remove crop, add to inventory, broadcast
    }

     SellItem(string itemId, int quantity)
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
builder.Services.AddDbContext<WorldDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("World")));

builder.Services.AddSignalR();
builder.Services.AddSingleton<WorldTimer>();
builder.Services.AddScoped<IPlayerRepository, PlayerRepository>();
builder.Services.AddScoped<IFarmRepository, FarmRepository>();
builder.Services.AddScoped<IMarketService, MarketService>();

builder.Services.AddMemoryCache(); // For tile caching
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