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

- Presentation tier decision: Keep rendering and UI state in React; make API requests through the shared client and show loading, empty, and accessible error states.
- Logic tier decision: Keep request validation, business rules, and response shaping in Express routes/services; return deliberate 4xx responses for invalid input or missing records, and do not add authentication.
- Data tier decision: Use Mongoose schemas for required-field validation and relationships, and keep seed data idempotent without deleting unrelated records.
- Activity tracking approach: Validate that submitted user and team IDs are well-formed and reference existing records before saving; compute points and `occurredAt` in the API.
- Team management approach: Require a team name, preserve the submitted name and mascot, and support a members array that defaults to empty.
- Leaderboard approach: Calculate each team's total from its activity points, order totals highest first, and assign 1-based ranks.
- Workout suggestions approach: Find the user by email, prioritize workouts matching their fitness goal or recent activity, and include a reason; return a deliberate 4xx when no user matches.

## Test plan

Run the focused contract tests in data → API → presentation order so failures are
diagnosed at the tier that owns the behavior. Run both builds after the tests.

- Data tier — `npm run test:data --prefix octofit-tracker/backend`: checks required
  model fields and idempotent seed behavior. Four of five tests failed: required
  field validation failed for users, teams, activities, and workouts; the seed
  idempotency test passed.
- API tier — `npm run test:api --prefix octofit-tracker/backend`: checks activity
  and team creation/listing, invalid-input responses, leaderboard totals/ranks,
  and user-matched workout suggestions. All four tests failed: invalid activity
  and team requests returned `501` instead of `400`, the leaderboard was empty,
  and the first workout suggestion did not match the user's goal/activity.
- Presentation tier — `npm test --prefix octofit-tracker/frontend`: checks API
  rendering and requests, plus loading, empty, and accessible error states for
  each route. All 15 tests failed, indicating these presentation contracts are
  not currently satisfied.
- Backend build — `npm run build --prefix octofit-tracker/backend`: type-checks
  backend TypeScript. A failure indicates backend compile/type errors.
- Frontend build — `npm run build --prefix octofit-tracker/frontend`: type-checks
  frontend TypeScript and builds the Vite app. A failure indicates frontend
  compile/type errors or a Vite build error.
