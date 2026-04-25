
# Understanding Web Routing

## Glossary of Technical Terms

- **Routing**: The process of determining how requests (URLs) are handled by the application, either on the server or client side.
- **Server routing**: Routing handled by the backend server, typically for API endpoints or serving the main application.
- **Client routing**: Routing handled by the frontend application, determining which components or pages to display based on the URL.
- **SPA (Single Page Application)**: A web application that loads a single HTML page and dynamically updates the content as the user interacts with it.
- **SignalR**: A library for real-time web functionality, enabling server-client communication like live updates.
- **React Router**: A library for managing client-side routing in React applications.
- **Deep links**: URLs that point to specific content or pages within a web application, even if the app is a SPA.

## Two routing situations

You’ve basically got two kinds of routing in this project, and it helps to keep them mentally separated:

1. Server routing (ASP.NET “Presentation”)
2. Client routing (your SPA: React/TS “Client”)

In a modern "MVC-ish + SPA" setup, the “pages” (`/login`, `/farm`, `/market`) are usually **client-side routes**, while the server handles **API routes** (`/api/...`) and the SignalR endpoint (`/gameHub`).

---

### 1) Server routing (ASP.NET): APIs + Hub + “serve the SPA”

What it routes

* `/api/world/tiles`, `/api/market`, etc. → Controllers
* `/gameHub` → SignalR hub endpoint
* Everything else like `/farm` or `/login` → returns your SPA `index.html` so React Router can take over


Examples:

- **Request URL**: `http://localhost:5000/api/world/tiles`
  - This is an API request handled by the server routing, mapped to a controller.

- **Deep Link**: `http://localhost:5000/farm`
  - This is a client-side route that loads the SPA and navigates to the `<Farm />` component using React Router.

### What this looks like in `Program.cs` (conceptually)

* `app.MapControllers();`
* `app.MapHub<GameHub>("/gameHub");`
* “Fallback to index.html” for non-API paths

That fallback is the key for “deep links” (typing `http://localhost:xxxx/farm` into the browser should still load the SPA).

---

## 2) Client routing (React): actual pages

### What it routes

* `/login` → `<Login />`
* `/farm` → `<Farm />`
* `/market` → `<Market />`

This is done with **React Router** (typical) and it runs entirely in the browser. The server just serves the same SPA bundle; React decides what to render.

The “best way” here is:

* Keep **pages** in the Client (React Router)
* Keep **resources / actions** as server endpoints (`/api/...`) + SignalR pushes

That’s still compatible with MVC: your “Controllers” are the “C” of server-side interactions, but your actual “Views” are React components (client-side), not Razor views.

---

## The MVC pattern interpretation in your architecture

In *your* layered model:

- **Presentation**: request routing + controllers + hub wiring (API surface)
- **Infrastructure**: persistence + SignalR hub implementation details + timers
- **Domain**: pure game rules

So “routing to different pages” is mostly a **Client concern**, while “routing to different endpoints” is a **Presentation concern**.

---

## Recommended structure for Farmia v0.1

* Server:

  * `/api/*` = REST endpoints
  * `/gameHub` = SignalR
  * `/*` (non-api) = serve SPA index.html

* Client (React Router):

  * `/login`, `/farm`, `/market`, `/profile`

This keeps the server clean and avoids mixing Razor Pages + SPA unless you *want* that.

---

If you tell me one thing—are you serving the React client **from the ASP.NET app** (same port), or running it separately on `localhost:3000` during dev?—I’ll give you the exact `Program.cs` routing snippet that matches your setup (proxy vs static files vs fallback).

