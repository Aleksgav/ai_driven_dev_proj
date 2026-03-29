# Scrum Planning Poker — First Stage Implementation Plan

## Overview

A real-time web application for team sprint estimation. Teams join a room, vote secretly on task complexity using estimation cards, then reveal votes and calculate statistics. The app is split into two parts:

- **`backend/`** — Rails 8 API-only (PostgreSQL, ActionCable, RSpec)
- **`frontend/`** — Next.js SPA (TypeScript, TanStack Query, Zustand, WebSocket, Tailwind, Shadcn)

---

## Project Structure

```
ai_driven_dev_proj/
├── backend/                        # Rails 8 API-only
│   ├── app/
│   │   ├── channels/
│   │   ├── controllers/api/v1/
│   │   ├── models/
│   │   ├── serializers/
│   │   └── services/
│   ├── config/
│   ├── db/migrate/
│   └── spec/
│
└── frontend/                       # Next.js SPA
    └── src/
        ├── app/                    # Pages (static export, no SSR)
        ├── components/
        ├── hooks/
        ├── lib/api/ and lib/ws/
        ├── store/                  # Zustand stores
        └── types/
```

---

## Backend — Rails 8 API-Only

### Bootstrap

```bash
rails new backend --api --database=postgresql --skip-test
# Gemfile test group: rspec-rails, factory_bot_rails, faker
```

### Domain Models

#### `users`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK via pgcrypto |
| `display_name` | string | NOT NULL |
| `email` | string | UNIQUE NOT NULL |
| `password_digest` | string | `has_secure_password` |
| `token` | string | UNIQUE, auth token |
| `avatar_url` | string | |

#### `decks`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `name` | string | e.g. "Fibonacci" |
| `deck_type` | string | enum: fibonacci / modified_fibonacci / tshirt / custom |
| `is_default` | boolean | |

#### `deck_cards`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `deck_id` | uuid | FK |
| `label` | string | display value: "1", "?", "∞", "XL" |
| `value` | decimal | numeric for averaging; NULL for special cards |
| `position` | integer | ordering |

#### `rooms`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `name` | string | NOT NULL |
| `slug` | string | UNIQUE, 6-char join code |
| `owner_id` | uuid | FK → users |
| `deck_id` | uuid | FK → decks |
| `status` | string | enum: waiting / voting / revealed / finished |
| `jira_project_key` | string | optional |

#### `room_members`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `room_id` | uuid | FK |
| `user_id` | uuid | FK |
| `role` | string | enum: owner / voter / observer |
| UNIQUE | | on (room_id, user_id) |

#### `tasks`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `room_id` | uuid | FK |
| `title` | string | NOT NULL |
| `description` | text | |
| `external_id` | string | Jira key, e.g. "PROJ-123" |
| `external_url` | string | link back to Jira |
| `position` | integer | ordering |
| `status` | string | enum: pending / voting / voted / skipped |
| `final_estimate` | decimal | set by owner after reveal |

#### `votes`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `task_id` | uuid | FK |
| `user_id` | uuid | FK |
| `deck_card_id` | uuid | FK |
| UNIQUE | | on (task_id, user_id) |

#### `vote_results`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | |
| `task_id` | uuid | FK UNIQUE |
| `average_value` | decimal | |
| `min_value` | decimal | |
| `max_value` | decimal | |
| `vote_count` | integer | |
| `consensus` | boolean | true when all votes equal |
| `revealed_at` | timestamp | |

### Migrations (in order)
1. `enable_pgcrypto`
2. `create_users`
3. `create_decks`
4. `create_deck_cards`
5. `create_rooms`
6. `create_room_members`
7. `create_tasks`
8. `create_votes`
9. `create_vote_results`

### Authentication

`has_secure_password` + `users.token` (long random string via `SecureRandom`). No JWT gem needed.

- Protected routes: `Authorization: Bearer <token>` header
- ActionCable: token passed as `?token=<token>` WS query param
- `ApplicationController#authenticate_user!` sets `current_user`

### API Endpoints (`/api/v1`)

**Auth**
- `POST /auth/register` — create account, return `{ user, token }`
- `POST /auth/login` — authenticate, return `{ user, token }`
- `DELETE /auth/logout` — invalidate token
- `GET /auth/me` — current user profile

**Rooms**
- `POST /rooms` — create room
- `GET /rooms/:slug` — room state + members + tasks
- `PATCH /rooms/:slug` — update room (owner only)
- `POST /rooms/:slug/join` — join as voter
- `DELETE /rooms/:slug/leave` — leave room
- `GET /rooms/:slug/history` — all revealed tasks with results

**Tasks**
- `GET /rooms/:slug/tasks` — list tasks
- `POST /rooms/:slug/tasks` — add task manually
- `POST /rooms/:slug/tasks/import` — bulk import from Jira (owner only)
- `PATCH /rooms/:slug/tasks/:id` — update task
- `DELETE /rooms/:slug/tasks/:id` — remove task
- `POST /rooms/:slug/tasks/:id/start_voting` — owner starts voting round
- `POST /rooms/:slug/tasks/:id/reveal` — owner reveals votes → creates VoteResult
- `POST /rooms/:slug/tasks/:id/skip` — owner skips task
- `PATCH /rooms/:slug/tasks/:id/estimate` — owner saves final estimate

**Votes**
- `POST /rooms/:slug/tasks/:id/votes` — cast or update own vote
- `DELETE /rooms/:slug/tasks/:id/votes` — retract vote

**Decks**
- `GET /decks` — list available decks with cards

### ActionCable — `RoomChannel`

Subscribes to `room_<slug>` stream. All messages: `{ event: "...", payload: {...} }`.

| Event | Payload | When |
|-------|---------|------|
| `room:member_joined` | `{ member }` | User joins |
| `room:member_left` | `{ user_id }` | User leaves |
| `room:task_added` | `{ task }` | Task created |
| `room:task_updated` | `{ task }` | Task edited |
| `room:task_removed` | `{ task_id }` | Task deleted |
| `voting:started` | `{ task }` | Voting round begins |
| `voting:vote_cast` | `{ task_id, voted_count, total_members }` | Vote submitted (value hidden) |
| `voting:revealed` | `{ task, result, votes[] }` | Votes revealed with full values |
| `voting:skipped` | `{ task }` | Task skipped |
| `voting:estimate_saved` | `{ task }` | Final estimate saved |

**Secret voting**: During voting, only `voted_count`/`total_members` are broadcast. Card values are never sent until `reveal`.

### Service Objects

- `Rooms::CreateService` — creates room + owner RoomMember
- `Rooms::JoinService` — idempotent join
- `Voting::StartService` — transitions task → `voting`, broadcasts
- `Voting::CastVoteService` — upserts vote, broadcasts count
- `Voting::RevealService` — calculates avg/min/max, creates VoteResult, broadcasts

### Serializers (plain Ruby, no gems)

- `UserSerializer` — `id`, `display_name`, `avatar_url`
- `RoomSerializer` — includes nested deck, owner, members
- `RoomMemberSerializer` — includes `has_voted` bool (during voting only)
- `TaskSerializer` — `vote_result` null until revealed
- `VoteSerializer` — card value hidden until revealed
- `VoteResultSerializer` — `average_value`, `min_value`, `max_value`, `consensus`, `revealed_at`
- `DeckSerializer` — includes ordered cards

Response shape: `{ "data": {...} }` or `{ "errors": [{ "field": "...", "message": "..." }] }`

### Seeds

Default decks created in `db/seeds.rb`:
- **Fibonacci**: 0, 1, 2, 3, 5, 8, 13, 21, 34, 55, ?, ∞
- **Modified Fibonacci**: 0, 0.5, 1, 2, 3, 5, 8, 13, 20, 40, 100, ?, ∞
- **T-Shirt**: XS, S, M, L, XL, XXL, ?

---

## Frontend — Next.js SPA

### Bootstrap

```bash
npx create-next-app@latest frontend --typescript --tailwind --app
npx shadcn@latest init
```

`next.config.ts` — no SSR:
```ts
output: 'export',
trailingSlash: true,
```

### Pages

| Route | File | Description |
|-------|------|-------------|
| `/` | `app/page.tsx` | Landing |
| `/login` | `app/login/page.tsx` | Login form |
| `/register` | `app/register/page.tsx` | Register form |
| `/rooms/new` | `app/rooms/new/page.tsx` | Create room |
| `/rooms/[slug]` | `app/rooms/[slug]/page.tsx` | Main room view |
| `/rooms/[slug]/history` | `app/rooms/[slug]/history/page.tsx` | Voting history |

All pages are `'use client'`. Auth guard via `useRequireAuth` hook.

### Component Hierarchy

```
components/
├── ui/                     # Shadcn primitives
├── layout/
│   ├── Header.tsx
│   ├── PageShell.tsx
│   └── MobileNav.tsx
├── auth/
│   ├── LoginForm.tsx
│   └── RegisterForm.tsx
├── room/
│   ├── RoomLobby.tsx       # waiting state: members, deck, tasks
│   ├── RoomHeader.tsx      # name, slug/copy-link, status badge
│   ├── MemberList.tsx      # avatars + voted indicator
│   ├── MemberAvatar.tsx    # avatar with "voted" crown overlay
│   ├── CreateRoomForm.tsx
│   └── JoinRoomForm.tsx
├── task/
│   ├── TaskList.tsx
│   ├── TaskListItem.tsx
│   ├── TaskAddForm.tsx
│   ├── TaskImportDialog.tsx  # Jira import modal
│   └── ActiveTaskBanner.tsx  # currently-voted task display
├── voting/
│   ├── VotingPanel.tsx
│   ├── EstimationCard.tsx  # single card with flip animation
│   ├── CardHand.tsx        # horizontal scroll, mobile-first
│   ├── VoteStatusBar.tsx   # "X of Y voted" progress
│   ├── RevealButton.tsx    # owner-only
│   └── VoteResults.tsx     # post-reveal: face-up grid + stats
└── history/
    ├── HistoryList.tsx
    └── HistoryItem.tsx     # avg/min/max chips, consensus badge
```

### State Management (Zustand)

**`authStore`**
```ts
user: User | null
token: string | null
isAuthenticated: boolean
// actions: setAuth, clearAuth, hydrate (from localStorage)
```

**`roomStore`**
```ts
room: Room | null
members: RoomMember[]
tasks: Task[]
activeTask: Task | null
myVote: DeckCard | null
// actions: setRoom, upsertMember, removeMember, upsertTask,
//          removeTask, setMyVote, applyVoteReveal, reset
```

**`wsStore`**
```ts
connected: boolean
error: string | null
```

### TanStack Query Hooks (`src/hooks/`)

| Hook file | Exports |
|-----------|---------|
| `useAuth.ts` | `useMeQuery`, `useLoginMutation`, `useRegisterMutation`, `useLogoutMutation` |
| `useRoom.ts` | `useRoomQuery(slug)`, `useCreateRoomMutation`, `useJoinRoomMutation` |
| `useTasks.ts` | `useTasksQuery`, `useAddTaskMutation`, `useImportTasksMutation`, `useStartVotingMutation`, `useRevealMutation`, `useSaveEstimateMutation` |
| `useVotes.ts` | `useCastVoteMutation` |
| `useDecks.ts` | `useDecksQuery` |
| `useHistory.ts` | `useHistoryQuery(slug)` |

WS event handlers call `queryClient.setQueryData` for instant cache updates alongside Zustand store updates.

### WebSocket Integration

**`src/lib/ws/roomSocket.ts`** — wraps `@rails/actioncable`:

```ts
interface RoomSocketHandlers {
  onMemberJoined(member: RoomMember): void
  onMemberLeft(userId: string): void
  onTaskAdded(task: Task): void
  onTaskUpdated(task: Task): void
  onTaskRemoved(taskId: string): void
  onVotingStarted(task: Task): void
  onVoteCast(payload: { task_id: string; voted_count: number; total_members: number }): void
  onVotesRevealed(payload: { task: Task; result: VoteResult; votes: Vote[] }): void
  onVotingSkipped(task: Task): void
  onEstimateSaved(task: Task): void
  onConnected(): void
  onDisconnected(): void
}
```

**`src/hooks/useRoomSocket.ts`** — React hook that:
1. Creates ActionCable consumer at `ws://backend/cable?token=<token>`
2. Subscribes to `RoomChannel { slug }`
3. Dispatches typed events → Zustand store + `queryClient.setQueryData`
4. Returns `{ connected, error }`
5. Cleans up subscription on unmount

Used once in `/rooms/[slug]/page.tsx` for the lifetime of the room view.

---

## Voting Session Lifecycle

```
OWNER                         SERVER                         MEMBER(S)
  |                              |                               |
  | POST /rooms (create)         |                               |
  |----------------------------->|                               |
  |                              |                               |
  | WS: subscribe RoomChannel    |<-- WS: subscribe RoomChannel--|
  |----------------------------->|                               |
  |                              |-- broadcast: member_joined -->|
  |                              |                               |
  | POST /tasks (add tasks)      |                               |
  |----------------------------->|                               |
  |                              |-- broadcast: task_added ----->|
  |                              |                               |
  | POST /tasks/:id/start_voting |                               |
  |----------------------------->|                               |
  |                              |-- broadcast: voting:started ->|
  |                              |                               |
  |                              |<-- POST /votes (cast vote) ---|
  |<-- broadcast: vote_cast      |-- broadcast: vote_cast ------>|
  |    (count only, no values)   |                               |
  |                              |                               |
  | POST /tasks/:id/reveal       |                               |
  |----------------------------->|                               |
  |                              | Creates VoteResult            |
  |<-- broadcast: votes_revealed |-- broadcast: votes_revealed ->|
  |    { votes[], result }       |    { votes[], result }        |
  |                              |                               |
  | PATCH /tasks/:id/estimate    |                               |
  |----------------------------->|                               |
  |                              |-- broadcast: estimate_saved ->|
```

---

## Implementation Sequence

| Phase | Tasks |
|-------|-------|
| **1 — Foundation** | `rails new backend`, all migrations, seeds; `npx create-next-app frontend`, configure static export, init Shadcn |
| **2 — Auth** | `User` model + token auth; `AuthController`; frontend `authStore` + login/register pages |
| **3 — Rooms** | `Room`/`RoomMember` models + services; `RoomsController`; create/join room UI |
| **4 — Tasks** | `Task` model + `TasksController`; `TaskList` UI |
| **5 — Real-time** | `ApplicationCable::Connection`; `RoomChannel` + broadcasts in services; `useRoomSocket` hook |
| **6 — Voting** | `Vote`/`VoteResult` models; cast/reveal services; `VotingPanel` + `EstimationCard` UI |
| **7 — History & Polish** | History endpoint; `HistoryList` page; mobile-first layout audit |

---

## Configuration

### `backend/config/routes.rb` (outline)

```ruby
namespace :api do
  namespace :v1 do
    post   'auth/register'
    post   'auth/login'
    delete 'auth/logout'
    get    'auth/me'
    get    'decks', to: 'decks#index'

    resources :rooms, param: :slug, only: [:create, :show, :update] do
      member do
        post   :join
        delete :leave
        get    :history
      end
      resources :tasks, only: [:index, :create, :update, :destroy] do
        collection { post :import }
        member do
          post  :start_voting
          post  :reveal
          post  :skip
          patch :estimate
          get   :result
        end
        resource :vote, only: [:create, :destroy]
      end
    end
  end
end

mount ActionCable.server => '/cable'
```

### `backend/config/cable.yml`

```yaml
development:
  adapter: async
test:
  adapter: test
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
```

### `frontend/next.config.ts`

```ts
const config: NextConfig = {
  output: 'export',
  trailingSlash: true,
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_WS_URL:  process.env.NEXT_PUBLIC_WS_URL,
  },
}
```

### Environment Variables

```bash
# backend
DATABASE_URL=postgresql://localhost/planning_poker_dev
FRONTEND_URL=http://localhost:3000

# frontend
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:3001/cable
```

---

## Testing Strategy

### Backend (RSpec)
- **Model specs** — validations, associations, status enum transitions
- **Request specs** — full HTTP cycle per controller action (FactoryBot fixtures)
- **Channel specs** — `ActionCable::Channel::TestCase` for subscribe/broadcast
- **Service specs** — unit tests for `Voting::RevealService` (avg/min/max), `Rooms::JoinService` (idempotency)

Key factories: `users`, `rooms` (with `deck`), `tasks` (with `:voting`/`:voted` traits), `votes`

### Frontend (Vitest)
- **Unit** — Zustand store logic, `roomSocket.ts` event dispatch, API client error handling
- **Component** (`@testing-library/react`) — `EstimationCard` selection state, `VoteResults` reveal rendering
- **Integration** — mock WS events flow through to component re-render

---

## Critical Files

| File | Why Critical |
|------|-------------|
| `backend/config/routes.rb` | Routing contract — must be correct before any controller work |
| `backend/db/migrate/` | Immutable schema foundation — all models and tests depend on it |
| `backend/app/channels/room_channel.rb` | WS broadcast hub — defines all real-time event contracts |
| `frontend/src/lib/ws/roomSocket.ts` | Typed ActionCable adapter — all real-time frontend depends on this |
| `frontend/src/lib/api/types.ts` | Shared TypeScript interfaces — changes ripple everywhere |
