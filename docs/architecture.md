# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier decision: Keep the React app as a thin, route-driven UI layer that reads data through the shared fetch client, renders loading and empty states, and surfaces HTTP failures as accessible alerts. The app should expose a simple navigation shell and let each page focus on one capability.
- Logic tier decision: Put validation and orchestration in the Express service layer rather than in the route handlers. Each route should stay thin, call a service that validates the body/params, and convert validation problems into deliberate 4xx responses while leaving auth out of scope for this exercise.
- Data tier decision: Model the domain as Mongoose schemas with required fields, object references, and stable defaults; treat seed data as idempotent demo content that updates existing records instead of dropping unrelated app data.
- Activity tracking approach: Validate `userId` and `teamId` exist, reject malformed IDs with a 4xx, and always store the normalized `user`, `team`, `type`, `minutes`, and computed `points` plus `occurredAt` on creation. The activity flow should not leak cast errors as 500s.
- Team management approach: Require `name` and `mascot`, default `members` to an empty array, and preserve member ObjectIds as the source of truth for rendering and API responses.
- Leaderboard approach: Aggregate team totals from activity points, sort descending, assign a 1-based rank, and respond with `teamName`, `totalPoints`, and `rank` in the exact order the UI expects.
- Workout suggestions approach: Resolve a user by email, choose workouts that best match the user’s recent activity or goal, include a human-readable `reason`, and return a 4xx response for unknown emails rather than a generic 500.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Current failure signal: the four schema tests fail because `User`, `Team`, `Activity`, and `Workout` have no required validation. This confirms the root cause is in the Mongoose models, not the API layer.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Current failure signal: `POST /api/activities` and `POST /api/teams` return 501 instead of a 400 validation response; `/api/leaderboard` returns an empty array; workout suggestions return the wrong first suggestion and do not include the expected reason. This tells us the route layer is wired but the service logic is still stubbed.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Current failure signal: the UI still renders placeholder “Implementation pending” content and cannot find alert states or data-driven sections. This means the presentation layer has not yet consumed the API contract correctly.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Run after the service and schema fixes; a successful build confirms TypeScript compiles cleanly with the Mongoose and Express contracts.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Run after the UI is wired to the API; a successful build confirms the React app compiles with the route, fetch, and component contracts in place.
