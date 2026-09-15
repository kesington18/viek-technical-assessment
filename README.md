# VIEK Client Management — Debugging Assessment

This document details the bugs found in the VIEK Client Management application (React/Vite frontend + Express backend), their root causes, the fixes applied, how the app was tested afterward, and security considerations identified during the review.

## Bugs Identified, Root Causes & Solutions

### 1. `React is not defined` (App.jsx)
**Root cause:** The JSX in `App.jsx` compiles down to `React.createElement(...)` calls, which require `React` to be in scope. The file only imported `useEffect` and `useState` from `react`, with no default `React` import, and no automatic JSX runtime was picking up the slack.
**Solution:** Added `import React from "react";` to the top of `App.jsx`.

### 2. `Cannot read properties of undefined (reading 'map')` — projects list
**Root cause:** `const [projects, setProjects] = useState();` initialized `projects` as `undefined`. Since data is fetched asynchronously inside `useEffect` (which runs *after* the first render), the very first render tried to call `.map()` on `projects` before any data had arrived, and `undefined.map` throws.
**Solution:** Changed the initial state to an empty array: `useState([])`, matching the pattern already used for `clients`. This lets the first render safely map over an empty list until real data replaces it.

### 3. `Cannot read properties of undefined (reading 'length')` — clients list
**Root cause:** The `GET /api/clients` backend route responds with `{ data: clients }`, but `loadClients()` on the frontend read `result.clients`, a key that doesn't exist on that response. This meant `setClients(undefined)` was called, overwriting the safe initial `[]` value.
**Solution:** Changed the frontend to read `result.data` instead of `result.clients`.

### 4. `POST /api/clients` returns 400 "Name and email are required" despite valid input
**Root cause:** The `addClient` request was missing a `"Content-Type": "application/json"` header. Without it, Express's `express.json()` middleware doesn't know to parse the request body, so `req.body` came through empty, making `name` and `email` appear missing on the server even though they were filled in on the form.
**Solution:** Added `"Content-Type": "application/json"` to the request headers, alongside the existing `Authorization` header.

### 5. `DELETE /api/clients/:id` returns 404 for a client that exists
**Root cause:** Express route parameters (`req.params.id`) always arrive as strings, while `client.id` in the in-memory array is a number. The delete route used strict inequality (`client.id !== id`), which was `true` for every client since a number never strictly equals a string, so nothing was ever filtered out, triggering the "not found" branch.
**Solution:** Converted the incoming ID to a number before filtering: `const id = Number(req.params.id);`. (A loose equality (`!=`) fix was also considered, but rejected. It works, but silently relies on type coercion rather than making the type expectation explicit, which is a worse practice to carry into production code.)

### 6. Deleting a client leaves their projects behind
**Root cause:** The deleteClient route only filtered the `clients` array. It never touched `projects`, so any projects belonging to a deleted client became orphaned, still present and still displayed, referencing a `clientId` that no longer existed.
**Solution:** Added a filter on `projects` inside the same delete route (`projects = projects.filter((project) => project.clientId !== id);`), and changed the `projects` declaration from `const` to `let`, since it now needed to be reassigned.

### 7. Projects list doesn't refresh after a client is deleted
**Root cause:** `deleteClient()` only called `loadClients()` on success never `loadProjects()`. So even though the backend correctly removed the orphaned projects, the frontend kept displaying stale project data until a full page reload.
**Solution:** Added a `loadProjects()` call alongside `loadClients()` in the success branch of `deleteClient()`.

### 8. New client IDs can collide with existing IDs after a deletion
**Root cause:** New client IDs were generated as `clients.length + 1`. This breaks once a deletion has happened: e.g., starting with clients `[1, 2, 3]`, deleting client `1` leaves `[2, 3]` (length 2), and the next new client would be assigned `id: 3` colliding with the existing client `3`.
**Solution:** Refactored ID generation to `Math.max(...clients.map(c => c.id), 0) + 1`, which finds the actual highest existing ID and increments it, regardless of array length or ordering.

## Testing

Testing was done manually against the running app, using the browser DevTools Console and Network tabs as the primary tools:

- Walked through the full user flow after each fix: login → view clients → add a client → delete a client → filter projects by client.
- Used the Network tab to inspect actual request/response payloads and status codes for each API call, to confirm fixes matched what the backend was really sending.
- Used `console.log` inside the backend route to compare the types of `req.params.id` and `client.id` directly, confirming the string/number mismatch before applying the fix.
- Reproduced the client-ID collision bug deliberately (delete a client, then add a new one) to confirm both the bug and the fix.
- Verified the error-handling middleware by manually sending a request with deliberately malformed JSON via `fetch()` in the console, confirming the server returned a clean JSON `500` error instead of crashing or hanging.

## Security Considerations

**Plaintext password storage.** User passwords are stored as plain, readable text in the `users` array, and login compares them directly. In production, passwords should be hashed with a purpose built algorithm like `bcrypt`, the stored value should never be the raw password, and login should compare hashes, not plaintext.

**Weak, static authentication token.** Every successful login receives the same hardcoded string, `"demo-token"`, and the `authenticate` middleware just checks for that exact string. This is trivially guessable by anyone who knows the string can authenticate as any user without logging in. Production should issue a unique, cryptographically random token per session, or a signed token format like JWT.

**No token expiry.** The token never expires, so a stolen or leaked token would grant indefinite access. Production tokens should have a limited lifetime, with a refresh mechanism for renewing access.

**Unrestricted CORS policy.** `app.use(cors())` is called with no configuration, allowing requests from any origin. Production should restrict this to an explicit allowlist of trusted frontend origins.

**Pre-filled login credentials.** The login form initializes with real, working credentials already filled in (`admin@viek.test` / `password123`). This is convenient for testing but should never ship to production. Login forms should start empty.

**Token stored in `localStorage`.** The auth token is stored in `localStorage`, which is readable by any JavaScript on the page including malicious scripts from an Cross Site Scripting vulnerability. A safer alternative is an `httpOnly` cookie, which client-side JavaScript cannot read at all.

## Reflection

**Issue that required the most investigation:** The `DELETE /api/clients/:id` returning 404 for a client that clearly existed. It wasn't obvious from reading the code alone  the filter logic *looked* correct at a glance. Confirming it required actually logging both values and their types (`typeof`) inside the route to see the string vs number mismatch directly, rather than trying to spot it by inspection.

**General debugging approach:** Reproduce the issue first, then read the error message/status code carefully rather than guessing, then trace the data backward from where it broke (the crash site) to where it originated (state initialization, an API response shape, a type mismatch) checking assumptions with `console.log` or the Network tab rather than assuming code was correct just because it looked reasonable.

**Unresolved issues:** None outstanding all identified bugs were reproduced, root-caused, and fixed. The error-handling middleware was reviewed and confirmed to work correctly for parser-level errors (via a deliberate malformed-JSON test); and tested using the console in the devtools