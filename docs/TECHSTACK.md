# Thinking out loud

> Is `REST`/`WS API` part of the asp.net communication?

Yes. In ASP.NET Core, `REST`, `WebSockets`, and `SignalR` are all just *different ways your browser talks to the same ASP.NET app* over HTTP(S).

## Untangling the buzzwords

Let’s name things clearly:

* `ASP.NET Core app`
  Your C# process. It hosts:

  * REST endpoints (`app.MapGet`, `app.MapPost`, controllers, etc.)
  * SignalR hubs (`app.MapHub<WorldHub>("/world")`)
  * Static files (`wwwroot/index.html`, JS, CSS)
  * Optional raw WebSocket endpoints if you want to go low-level.

* `REST API`
  Regular HTTP calls:

  * `GET /api/chunks/123`
  * `POST /api/tile`
    Your browser uses `fetch()` or `XMLHttpRequest` to hit these.

* `WS API`
  Usually means `WebSocket API`: long-lived, bidirectional connection.
  You can:

  * Use `raw WebSockets` yourself (`app.UseWebSockets()` + `HttpContext.WebSockets`).
  * Or use `SignalR`, which wraps WebSockets/long-polling for you and gives you hubs + methods.

* `SignalR`
  A higher-level real-time layer on top of WebSockets/other transports:

  * Browser calls `connection.invoke("ChangeTile", x, y, type)`
  * Server calls `Clients.All.SendAsync("TileChanged", x, y, type)`
    Under the hood it’s WebSockets (when available), but you don’t manually push bytes.

---

### A more accurate version of your diagram

Instead of:

> Client ↔ `REST`/`WS API` (`ASP.NET 8`) ↔ Database ↔ `SignalR Hub`

Think:
```
          ┌─────────────────────────────┐
          │       ASP.NET Core          │
          │  (single C# web app)        │
          │                             │
Browser   │  • REST endpoints           │
(any)  <──┼──• SignalR hubs             │──>  SQLite 
          │  • Static files (wwwroot)   │
          └─────────────────────────────┘
```

So the browser is talking `directly` to the ASP.NET Core app via:

* HTTP (`REST`)
* WebSocket-ish (`SignalR`)

---

### How this maps to your game

A very sane split for your tile world:

* `REST`:

  * `GET /api/chunks/{chunkId}` → returns the static snapshot of tiles
  * `GET /api/player` → player state, inventory, etc.

* `SignalR`:

  * Server → client:

    * `TileChanged(x, y, newType)`
    * `PlayerJoined(playerId, pos)`
  * Client → server:

    * `ChangeSelectedTool(toolName)`
    * `ApplyAction(x, y, toolName)`

* `DB (SQLite)`:

  * Actual storage of chunks, tiles, players etc., used by the ASP.NET app when handling those calls.


---

### Inside that HTTP pipe, you can have *multiple styles*

From the browser you can:

1. `Load static files`

   ```http
   GET /           → wwwroot/index.html
   GET /js/game.js → wwwroot/js/game.js
   ```

2. `Call REST endpoints` (normal HTTP API)

   ```js
   // JS in the browser:
   const res = await fetch("/api/chunks/123");
   const chunk = await res.json();
   ```

   That hits something like:

   ```csharp
   app.MapGet("/api/chunks/{id}", (int id) => ... );
   ```

3. `Use SignalR for realtime` (still over HTTP/WebSockets)

   ```js
   const connection = new signalR.HubConnectionBuilder()
       .withUrl("/worldHub")
       .build();

   await connection.start();
   connection.on("TileChanged", (x, y, type) => { ... });
   ```

   That hits:

   ```csharp
   app.MapHub<WorldHub>("/worldHub");
   ```

> “Ok so the communication takes place from Browser via REST endpoints to the ASP.NET app?”

* For classic request/response data (get chunk, save settings) → `yes, via REST endpoints`.
* For real-time events (tile changed, player moved) → usually `via SignalR`, which also goes `Browser → ASP.NET app`, but over a long-lived WebSocket-style connection instead of separate `REST` calls.

Same app, same ports, different shapes of messages.
