Yeah, that stack diagram is *close*, but a bit misleading in how it’s drawn.

> Is REST/WS API part of the asp.net communication?

Yes. In ASP.NET Core, **REST**, **WebSockets**, and **SignalR** are all just *different ways your browser talks to the same ASP.NET app* over HTTP(S).

---

### Untangling the buzzwords

Let’s name things clearly:

* **ASP.NET Core app**
  Your C# process. It hosts:

  * REST endpoints (`app.MapGet`, `app.MapPost`, controllers, etc.)
  * SignalR hubs (`app.MapHub<WorldHub>("/world")`)
  * Static files (`wwwroot/index.html`, JS, CSS)
  * Optional raw WebSocket endpoints if you want to go low-level.

* **REST API**
  Regular HTTP calls:

  * `GET /api/chunks/123`
  * `POST /api/tile`
    Your browser uses `fetch()` or `XMLHttpRequest` to hit these.

* **WS API**
  Usually means **WebSocket API**: long-lived, bidirectional connection.
  You can:

  * Use **raw WebSockets** yourself (`app.UseWebSockets()` + `HttpContext.WebSockets`).
  * Or use **SignalR**, which wraps WebSockets/long-polling for you and gives you hubs + methods.

* **SignalR**
  A higher-level real-time layer on top of WebSockets/other transports:

  * Browser calls `connection.invoke("ChangeTile", x, y, type)`
  * Server calls `Clients.All.SendAsync("TileChanged", x, y, type)`
    Under the hood it’s WebSockets (when available), but you don’t manually push bytes.

---

### A more accurate version of your diagram

Instead of:

> Client ↔ REST/WS API (ASP.NET 8) ↔ Database ↔ SignalR Hub

Think:

```text
          ┌─────────────────────────────┐
          │       ASP.NET Core          │
          │  (single C# web app)        │
          │                             │
Browser   │  • REST endpoints           │
(any)  <──┼──• SignalR hubs             │──>  SQLite / whatever
          │  • Static files (wwwroot)   │
          └─────────────────────────────┘
```

Or, labeled by behavior:

* **REST**: fetch world chunks, player profile, configs
* **SignalR**: push real-time “tile changed”, “player moved”, “chat message” events
* **DB**: persisted world state / chunks / saves

So the browser is talking **directly** to the ASP.NET Core app via:

* HTTP (REST)
* WebSocket-ish (SignalR)

SignalR is *not* “behind” the REST API; it lives next to it, in the same app.

---

### How this maps to your game

A very sane split for your tile world:

* **REST**:

  * `GET /api/chunks/{chunkId}` → returns the static snapshot of tiles
  * `GET /api/player` → player state, inventory, etc.

* **SignalR**:

  * Server → client:

    * `TileChanged(x, y, newType)`
    * `PlayerJoined(playerId, pos)`
  * Client → server:

    * `ChangeSelectedTool(toolName)`
    * `ApplyAction(x, y, toolName)`

* **DB (SQLite)**:

  * Actual storage of chunks, tiles, players etc., used by the ASP.NET app when handling those calls.

So: **REST + SignalR are both part of “ASP.NET communication”.** They’re different shapes of messages your ASP.NET Core app speaks over HTTP(S), not separate magical layers outside it.

Once you see it as “one C# process, many ways in/out”, you can mix and match them to taste for your engine.

---
Almost 🙂 — but it’s *not only* REST.

Try this mental model:

### 1. Everything goes **from Browser → ASP.NET app**

The browser never talks directly to the database, or to “SignalR as a separate thing”.
It only talks to your **ASP.NET Core app**, over HTTP(S).

From the browser’s point of view, it’s just:

```text
Browser  →  http://localhost:5000/...
                      ↑
                  ASP.NET app
```

### 2. Inside that HTTP pipe, you can have *multiple styles*:

From the browser you can:

1. **Load static files**

   ```http
   GET /           → wwwroot/index.html
   GET /js/game.js → wwwroot/js/game.js
   ```

2. **Call REST endpoints** (normal HTTP API)

   ```js
   // JS in the browser:
   const res = await fetch("/api/chunks/123");
   const chunk = await res.json();
   ```

   That hits something like:

   ```csharp
   app.MapGet("/api/chunks/{id}", (int id) => ... );
   ```

3. **Use SignalR for realtime** (still over HTTP/WebSockets)

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

All of these are **Browser → ASP.NET app**. REST is just *one* pattern on top of HTTP.

So a more precise answer to your question:

> “Ok so the communication takes place from Browser via REST endpoints to the ASP.NET app?”

* For classic request/response data (get chunk, save settings) → **yes, via REST endpoints**.
* For real-time events (tile changed, player moved) → usually **via SignalR**, which also goes Browser → ASP.NET app, but over a long-lived WebSocket-style connection instead of separate REST calls.

Same app, same ports, different shapes of messages.

