# Architecture

## Domain

What belongs here:

* Game rules and invariants
  * “A crop can only advance from Sprout → Mature if watered enough”
  * “You can only plant on soil”

* Entity behavior **expressed as pure logic**
  * `Crop.AdvanceGrowth(state)`
  * `Farm.CanPlaceTile(x, y)`

* State transitions that are deterministic and testable

What does **not** belong here:

* Timers
* Threads
* SignalR
* Databases
* “Every second do X”

Think of Domain as:

> *If the universe froze and you handed me a snapshot of the world, I could tell you what the next valid states are — but I wouldn’t advance time myself.*

CPU-heavy **can** happen here, but it’s not the defining feature.

---

## Infrastructure

Me: "long term storage of gameworld, database CRUD, timers for what?, communication via SignalR hub, hosting the SignalR hub here”

You’re circling the bullseye.

**Infrastructure = how the world touches reality.**

Correct responsibilities:

* Persistence
 
  * SQLite 
* Time & scheduling

  * Global tick loop
  * Bucket/timer engine
* Communication plumbing

  * SignalR Hub *implementation*
  * Redis later, message buses, etc.

Clarification:

* **The timer doesn’t decide *what* happens.**
  It decides *when* something should happen, then asks Domain logic what that means.

So:

* Infrastructure says: “It’s tick 12345, this crop is due”
* Domain says: “Given its state, here’s the new state”

Hosting the SignalR hub here is reasonable and common, because:

* It’s transport + I/O
* It’s not business logic

---

## Presentation

> *“ASP.NET stuff (I dont know what an aspnet entry point is), oh I dont know any of this. The html+css+js?”*

This confusion is **very normal**, and the fix is simple.

**Presentation = how the outside world talks to your app.**

In your case:

What belongs here:

* `Program.cs`

  * The **ASP.NET entry point**
  * This is literally: “start the web server, set up DI, routing, middleware”
* Controllers (`[HttpGet]`, `[HttpPost]`)
* SignalR endpoint wiring (`app.MapHub<GameHub>("/gameHub")`)
* Auth, CORS, middleware, Swagger
* DTOs / API contracts

What does **not** belong here:

* Game rules
* Tick logic
* EF entities
* Long-running loops

Important correction:

* **HTML/CSS/JS is NOT Presentation (in this solution)**
  That’s the **Client**, which just happens to be served *by* Presentation or separately.

Think of Presentation as:

> *The receptionist, not the factory.*

It translates HTTP / WebSocket messages into method calls and sends results back.

---

## Clean mental model (keep this one)

### Domain — “What is allowed to happen?”

* Pure rules
* Deterministic state changes
* No time, no I/O, no frameworks
* Easy to unit test

### Infrastructure — “How does it actually happen?”

* Databases
* Timers / tick engine
* SignalR hub implementation
* File systems, caches, OS, network
* Calls Domain logic to decide outcomes

### Presentation — “How do users talk to the system?”

* ASP.NET startup (`Program.cs`)
* Controllers & endpoints
* SignalR endpoint wiring
* Auth, CORS, middleware
* Converts HTTP/WS → method calls

### Client (separate, but important)

* HTML / CSS / JS
* Canvas rendering
* Input handling
* Calls APIs + listens to SignalR

---

“EF Core maps C# entity classes to tables, translates LINQ queries to SQL, tracks changes to entities, and applies those changes + schema migrations to the database.”

"20251220000000_AddSharedKeyToPlayers.cs" is a perfectly reasonable migration name.

## Routing 

Please see ROUTING.md

