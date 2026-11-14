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

Record your plan here before editing code.

- Presentation tier decision: Build a small route-driven React app that reads/writes through the shared API client, renders accessible loading/empty/error/success states, and keeps each capability in its own screen with Bootstrap styling.
- Logic tier decision: Keep Express handlers thin and move validation/business logic into TypeScript services under `src/services`, with explicit 4xx errors for invalid IDs, missing team/user references, and unknown email lookups.
- Data tier decision: Use Mongoose schemas as the source of validation and defaults, and seed stable demo users, teams, activities, and workouts with upserts while preserving prior membership links when a seeded user is reassigned.
- Activity tracking approach: Require `user`, `team`, `type`, `minutes`, and `occurredAt` on the model; compute `points` from the submitted minutes/type, and validate foreign references so invalid casts never leak as 500s.
- Team management approach: Team names and mascots are required; `members` defaults to an empty array and stores user IDs in a normalized list, with list/create routes exposing the joined member IDs.
- Leaderboard approach: Aggregate activities by team, sum `points`, order descending by total points, and assign a 1-based rank after sorting so the API returns the ranking contract exactly.
- Workout suggestions approach: Match a user by email, inspect their recent activity patterns or goal, and return workout suggestions ordered by the strongest match with a reason string explaining the recommendation.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Failing data model validation indicates the schema contracts are incomplete; fix required fields/defaults before API work.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Failures here show the Express routes/services are not yet implementing the `/api/*` contract or are returning 501/empty data instead of deliberate 4xx and sorted results.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Failing UI tests reveal the app is still a placeholder and not rendering actual data or accessible states from the API client.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Confirms TypeScript compiles cleanly after the data and API contract work is implemented.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Confirms the React app compiles and the API-driven pages render without TypeScript/module issues.
