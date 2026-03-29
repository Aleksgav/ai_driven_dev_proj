See PROJECT.md for project description.

## Backend Stack

Ruby, Ruby on Rails 8 (API only), PostgreSQL, RSpec

### Key commands

- `bin/setup` — bootstrap
- `bin/rails s` — run server
- `bundle exec rspec` — run tests
- `bin/rails db:migrate` — migrate

### Conventions

- Rails API only,
- RSpec for tests, FactoryBot and Faker for fixtures
- No new gems without explicit request

### Constraints

- Don't touch existing migrations

## Frontend Stack

Typescript, NextJs (SPA), react, TanStack Query (backend interations), Zustand (state management), Websocket, Vitest, Tailwind, Shadcn

### Key commands

- npm run dev
- npm run build
- npm start
- npm test

### Conventions

- NextJs SPA
- Vitest for tests
- Shadcn for components
- Tailwind for styles
- Mobile first

### Constraints

- No SSR
