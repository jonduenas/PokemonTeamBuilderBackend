# Pokémon Champions Team Builder: Backend Learning Project Plan

Nov 11, 2026 · Jon

## Goal and non-goals

The goal is to learn backend fundamentals, authentication above all, by building a Pokémon Champions team builder that is useful to its author and presentable as a portfolio piece.

**Goals**

- Build and deploy a real service with relational data, accounts, authentication, automated tests and CI.
- Make something the author will use for their own VGC team building.
- Leave a clear story for interviews: what was built, what was decided, and why.

**Non-goals for v1**

- Legality enforcement for species, moves and items per regulation.
- Match logging, analytics and strategy suggestions.
- Sharing and import/export.
- Standing out in a crowded market. Several Champions team builders already exist, and that is acceptable for a learning project.

## v1 scope

v1 is accounts plus multiple saved teams over a synced copy of Pokémon reference data, with no legality checks.

**In v1**

- Reference data (species including forms, moves, abilities, items, natures) synced from PokeAPI into the project's own database and read from there at runtime. PokeAPI asks consumers to cache what they fetch.
- Email-and-password accounts with self-built authentication.
- Create, read, update and delete multiple teams per user, up to six members each.
- Structural validation only: at most six members per team and four moves per member.

**Not in v1**

- Regulation or learnset legality, sharing, import/export, match logging.

**Known inaccuracy.** Users can pick any move for any species in v1. PokeAPI reflects the main-series games, not Champions, so some choices will not exist in the real game (for example, Incineroar reportedly lost Knock Off in Champions). This is accepted for v1 and is the first thing to fix later.

## Data model

The model has reference tables, accounts and teams, and teams point at reference rows by ID so legality rules can be added later without reshaping them.

| Entity | Key fields | Notes |
| --- | --- | --- |
| User | id, email, password hash, created at | Credentials only; no other personal data |
| RefreshToken | id, user, hashed token, expires at, revoked or rotated | Supports logout and revocation (see API and authentication) |
| Species | id, name, form, PokeAPI reference, synced at | Includes alternate forms; base stats and types optional in v1 |
| Move, Ability, Item, Nature | id, name, PokeAPI reference, synced at | Read-only; each row records its source and sync time |
| Team | id, user, name, notes, created at, updated at | Many teams per user |
| TeamMember | id, team, slot (1 to 6), species, item, ability, nature, up to four moves, stat points per stat | Level is fixed at 50 and IVs are all maximum in Champions, so neither is stored |

**To confirm:** the exact stat point limits per stat and in total, since Champions uses stat points instead of EVs. Gender and nicknames are left out unless wanted.

**Reserved for later.** Regulations, per-species learnsets and per-regulation legality become new tables, versioned by date because legality can change mid-regulation.

## API and authentication

The API is a versioned REST service (`/v1/...`), and authentication is built by hand with vetted libraries because that is the largest gap in the author's experience.

| Group | Examples | Access |
| --- | --- | --- |
| Auth | register, log in, refresh, log out | Public, rate-limited |
| Reference data | species, moves, abilities, items, natures; paginated and searchable | Read-only; cacheable |
| Teams | list and create; read, update and delete one team | Signed-in user only |
| Team members | edit one slot, for example `PUT /teams/{id}/members/{slot}` | Signed-in user only |

**Ownership.** Every team request is scoped to the signed-in user. Asking for another user's team returns "not found", not "forbidden", so it does not reveal that the team exists.

**Authentication design**

- Email and password. Passwords are hashed with a well-vetted library (argon2 or bcrypt), never a custom scheme.
- Login issues a short-lived access token and a longer-lived refresh token. Refresh tokens are stored hashed in the database so sessions can be revoked.
- Refresh tokens rotate on use, and reuse of an old token is detected and treated as a possible theft.
- Rate limiting on login. HTTPS only.

**Reference data sync.** A command-line job pulls from PokeAPI and upserts into the tables. It is idempotent, so running it twice creates no duplicates. It runs manually at first and on a schedule later.

## Stack, structure and testing

The proposed stack is TypeScript on Node.js with Fastify and Postgres, chosen because TypeScript also covers a web frontend later, which keeps full-stack roles in reach, and its static typing is close to Swift. Python with FastAPI was the earlier proposal and remains a reasonable alternative, since the concepts transfer.

- **Framework and data:** Fastify with schemas for requests and responses, Drizzle ORM for the database layer, drizzle-kit for migrations, Postgres. Drizzle stays close to SQL, which suits the aim of learning SQL properly.
- **Language and tooling:** Node.js (current LTS) with TypeScript in strict mode, run in development with tsx.
- **Local development:** Docker Compose runs the API and a Postgres database, so setup is one command.
- **Testing:** Vitest against a real test database, calling routes with Fastify's inject so no port is opened. The first priorities are the authentication flows (register, log in, refresh, rotation, log out) and team ownership checks, since mistakes there are the costly ones.
- **Auth libraries:** the argon2 package for password hashing and a vetted JWT library such as `@fastify/jwt` for access tokens; no hand-written cryptography.
- **Quality gates:** continuous integration runs type checking, tests and linting on every push.
- **Errors and validation:** consistent error responses and validation at the API boundary.

## Deployment

The first deployment goes to a simple managed host, with AWS as an optional later step.

- **Host:** Fly.io, Railway or Render are the candidates; the choice is open.
- **Configuration:** secrets in environment variables, never in the repository.
- **Operations basics:** a health endpoint and structured logs.
- **Transport:** HTTPS only.
- **Outcome:** a live URL that can be linked from a resume and a README that explains the decisions.

## Suggested milestones

The build runs in five phases, with authentication early because it is the main learning goal, and each phase ends with a check that can be run.

| Phase | What gets built | Done when |
| --- | --- | --- |
| 1. Foundations | Repo, Docker Compose and Postgres, migrations, health endpoint | One command starts the stack and CI runs the tests |
| 2. Accounts and auth | Register, log in, refresh with rotation, log out, login rate limit | Auth tests pass, including reuse of an old refresh token |
| 3. Reference data | PokeAPI sync job and read-only reference endpoints | Running the sync twice creates no duplicates |
| 4. Teams | Team and member CRUD with ownership scoping and structural checks | Another user's team returns not found, covered by tests |
| 5. Deploy | Managed host, secrets, health check, README with decisions | A live URL works end to end. This is v1. |

Phases run top to bottom; each one's check must pass before the next begins.

## Learning plan

The milestones above say what to build. This section says what to learn in each phase, where to learn it, and how to keep Claude from doing the learning for you. Each phase follows the same loop, and the sources are official docs or standards documents wherever one exists.

**The loop for every phase**

1. **Read first.** Work through the listed sources before writing the phase's code. Take notes in your own words.
2. **Predict.** Before running anything new, write down what you expect to happen (a response code, a table, a failing test). Surprises are where the learning is.
3. **Build it yourself.** Write the phase's code from the docs. Get stuck for a real stretch before asking for help.
4. **Explain it back.** Answer the phase's check questions in writing without looking, then compare with the sources.
5. **Get reviewed.** Only now ask Claude to review your code, point out what you missed and challenge your answers.

**Rules for using Claude**

- Allowed any time: explaining a concept you have already read about, decoding an error message, suggesting what to search for, quizzing you.
- Allowed after you have tried: a hint (not the solution), a code review, a critique of your design, a check of your answers to the questions.
- Not allowed: asking for the working implementation of a phase's core piece (password hashing and verification, token issue and rotation, the ownership check, the sync upsert) before you have written your own version. Boilerplate such as config files and Dockerfiles is fine to generate, but read every line.
- Ask Claude to teach with questions when you are unsure: "quiz me on refresh token rotation" works better than "write refresh token rotation".
- Keep a short decisions log in the repository as you go (what you chose, what you rejected, why). It is both your study record and your interview material.

**Pace.** Reading and building are interleaved, so do not read everything up front. Finish a phase's check before starting the next.

### Phase 1 learning: Foundations

Goal: understand what a request does from the HTTP message to the database row, and have a project that runs and tests with one command.

**Read** (the Fastify and TypeScript Handbook links come from the sites' documented structure and were not opened while planning, so check that each loads)

- [Overview of HTTP, MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview): requests, responses, methods, headers, status codes.
- [PostgreSQL tutorial, official docs](https://www.postgresql.org/docs/current/tutorial.html): tables, queries, joins, aggregates, updates and deletes, in psql. Skip what you already know about SQL, but do the joins section by hand.
- [The TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html): read the sections on everyday types, narrowing, generics and modules. Coming from Swift, pay attention to union types and structural typing.
- [Fastify: Getting Started](https://fastify.dev/docs/latest/Guides/Getting-Started/) and [Validation and Serialization](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/): routes, plugins and how request schemas are validated.
- [Docker: using Compose, official getting-started](https://docs.docker.com/get-started/workshop/08_using_compose/): defining an app and a database as services in one file.
- [Vitest: Getting Started](https://vitest.dev/guide/) and [Fastify's testing guide](https://fastify.dev/docs/latest/Guides/Testing/): then read how to set up and tear down a test database, since that will be a setup file or hook in your tests.

**Build:** the repository, a Compose file running the API and Postgres, a health endpoint, one migration-free table, and a first passing test in CI.

**Check yourself, in writing**

- What is the difference between a 401 and a 403, and between a 404 and a 400?
- Why does the API container reach the database at a hostname like `db` rather than `localhost`?
- Why does the test suite use a real Postgres database instead of mocks, and how does each test start from a clean state?

**Claude's role:** quiz you on HTTP semantics after you read; review your Compose file and test setup once they work.

### Phase 2 learning: Accounts and auth

Goal: be able to explain, without notes, why each part of the auth design exists and what attack it answers. This is the phase to slow down on.

**Read**

- [Password Storage Cheat Sheet, OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html): why fast hashes are unsuitable, why Argon2id is the first recommendation, and its minimum settings. Check the current numbers on the page rather than trusting a blog's copy.
- [Authentication Cheat Sheet, OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html): password policy, safe comparison, TLS, error messages that do not leak whether an account exists.
- [Session Management Cheat Sheet, OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) and MDN's [session management overview](https://developer.mozilla.org/en-US/docs/Web/Security/Authentication/Session_management): server-side sessions versus token-based approaches.
- [RFC 9700, OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700): you are not building OAuth, but section 4.14 (Refresh Token Protection) is the authoritative reasoning behind refresh token rotation and reuse detection. Read that section slowly; skim the rest.
- Fastify's [Getting Started](https://fastify.dev/docs/latest/Guides/Getting-Started/) and [Hooks](https://fastify.dev/docs/latest/Reference/Hooks/) docs show where a check can run in the request lifecycle, which is where your token verification belongs, before the route handler. The [argon2 package](https://github.com/ranisalt/node-argon2) covers hashing and verifying and defaults to Argon2id; check its options against the OWASP guidance above before copying any pattern, and read [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106) if you want the algorithm itself.

**Build in this order, each step with tests:** register with hashed password, log in, a protected endpoint, short-lived access token expiry, refresh token stored hashed, rotation on refresh, reuse detection, logout and revocation, login rate limiting.

**Check yourself, in writing**

- Why is a salted SHA-256 hash not acceptable for passwords, and what does Argon2id's memory cost add?
- Why store the refresh token hashed, the same way you would a password?
- Walk through a stolen refresh token being used by an attacker first, then by the real user. What does your system do at each step, and why does the whole token family get revoked?
- Why return the same error for "unknown email" and "wrong password"?
- What does your access token prove, and what can you not revoke before it expires?

**Claude's role:** after you attempt each question, act as a skeptical security reviewer of your code and your written answers. Ask Claude to list attacks against your design, then try to write the failing test for each one yourself.

### Phase 3 learning: Reference data

Goal: model relational data with Drizzle, change the schema safely with migrations, and write a sync job that is safe to run twice.

**Read**

- [Drizzle ORM documentation](https://orm.drizzle.team): declare tables in TypeScript, then read the sections on queries, filters, joins and relations. Because it stays close to SQL, compare each query you write with the SQL you would write by hand.
- [Drizzle migrations](https://orm.drizzle.team/docs/migrations) and the [generate](https://orm.drizzle.team/docs/drizzle-kit-generate) and [migrate](https://orm.drizzle.team/docs/drizzle-kit-migrate) commands: drizzle-kit writes SQL migration files from changes to your schema, and migrate applies the pending ones. Treat generated SQL as a draft you must read and correct, and do not use the shortcut that pushes the schema straight to the database once real data exists.
- [PokeAPI documentation](https://pokeapi.co/docs/v2): resources, links between them, and the fair use policy, which asks you to cache. Its data is read-only and does not change often, so syncing into your own tables is the intended pattern.
- PostgreSQL docs on `INSERT ... ON CONFLICT` for the upsert, found from the [official documentation](https://www.postgresql.org/docs/current/) (look it up in the SQL commands reference).

**Build:** models for the reference tables, an initial migration you have read line by line, a sync command that upserts species, moves, abilities, items and natures, and read-only paginated endpoints.

**Check yourself, in writing**

- What problem does a migration file solve that pushing your schema straight to the database does not?
- What makes your sync idempotent, and what would break if two syncs ran at once?
- Why do reference rows keep a PokeAPI reference and a synced-at time?
- What does the ORM issue as SQL for your list endpoint, and is it one query or many? (Turn on SQL logging and look.)

**Claude's role:** explain a Drizzle concept after you have read the matching tutorial section; review your migration and your sync for failure cases; do not write the upsert for you.

### Phase 4 learning: Teams

Goal: design a resource API where a user can only ever touch their own data, and prove it with tests.

**Read**

- Revisit Fastify's [Hooks](https://fastify.dev/docs/latest/Reference/Hooks/) and [Validation and Serialization](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/) docs: your "current user" check should be a hook or decorator that every team route goes through, and nested request bodies need schemas.
- Revisit the HTTP methods and status codes in the [MDN HTTP guides](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview): which method fits "replace one slot", what each create, update and delete should return.
- The [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) and [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) again, this time asking what an authenticated user is allowed to do rather than how they sign in.
- [Fastify's testing guide](https://fastify.dev/docs/latest/Guides/Testing/) and the [Vitest guide](https://vitest.dev/guide/), for building two users and running the same request as each.

**Build:** team and member tables with a migration, CRUD endpoints scoped to the current user, structural validation (six members, four moves, slots one to six), and tests with two users.

**Check yourself, in writing**

- Where in your code is ownership enforced, and what happens if you forget it on one new endpoint? How could the design make forgetting harder?
- Why return 404 rather than 403 for someone else's team?
- What belongs in request validation, what in the database as constraints, and what in business logic?

**Claude's role:** play a second user trying to read, edit and delete your teams, and list the requests you should test; then review your tests for gaps. Write the tests yourself first.

### Phase 5 learning: Deploy

Goal: get the service running somewhere real, with secrets handled properly, and be able to explain what happens between a push and a live URL.

**Read**

- [Fly.io's JavaScript docs](https://fly.io/docs/languages-and-frameworks/node), [Dockerfiles for Node apps](https://www.fly.io/docs/js/the-basics/dockerfiles/) and the [Fly Launch overview](https://fly.io/docs/reference/fly-launch): what `fly launch` and `fly deploy` do for you, and how a Dockerfile, written or generated, builds your app. If you choose Railway or Render instead, read their equivalent Node or Docker quickstart; the concepts are the same.
- Your host's docs on secrets, so you know how environment variables reach the running app.
- The [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) section on transmitting credentials only over TLS, as a final checklist.

**Build:** a deployed API with a managed Postgres, secrets set through the host, migrations run on deploy, a health check, structured logs, and a README that records your decisions.

**Check yourself, in writing**

- Which values are secrets, where do they live in development, in CI and in production, and what would you do if one leaked?
- What runs the migrations in production, and what happens if a migration fails halfway?
- How would you find out the service is down, and where do you look first?

**Claude's role:** review the README as a stranger would read it, and walk through a failure drill with you (a bad deploy, a leaked secret). You do the deploy.

## Later increments and open questions

Legality is the first thing to add after v1, and its data is the hardest open problem.

**Later increments, roughly in order**

1. Species legality for the current regulation, with one server-side validation rule.
2. Champions-accurate learnsets per species, imported once with provenance recorded on every row.
3. Showdown-format import and export of teams.
4. Sharing teams by link, which exercises permissions.
5. A thin iOS client with proper token handling: Keychain storage and a refresh flow that copes with concurrent requests.
6. Match logging tied to teams, recording the opponent's team and result but not the opponent's trainer name, to avoid holding other people's personal data.
7. Passkeys or Sign in with Apple.

**Open questions**

- Does PokeAPI cover any Champions-specific data? Not confirmed.
- Where do Champions learnsets come from? Bulbapedia has a Champions learnset page per Pokémon, and PokeTools shows Champions movepools. Check each source's license and terms before importing.
- The community Champions data repo (CC BY 4.0) bases its learnsets on Showdown's main-series data, and Champions corrections are still pending, so it cannot be treated as ground truth.
- Regulations rotate roughly every few months, and sources show Regulation M-C as current from September 9, 2026. Legality must be versioned by regulation and by date.
- Exact stat point limits and whether to store gender or nicknames.
- Which host to use for the first deployment.

## Sources

These came from research during planning. Several are community sites, and the Bulbapedia and PokeAPI items were seen in search results, so verify licenses, terms and current data before relying on any of them.

- [Regulation Set M-B, Bulbapedia](https://bulbapedia.bulbagarden.net/wiki/Regulation_Set_M-B): rules and eligible Pokémon per regulation
- [Incineroar Champions learnset, Bulbapedia](<https://bulbapedia.bulbagarden.net/wiki/Incineroar_(Pok%C3%A9mon)/Champions_learnset>): example of a per-Pokémon Champions learnset page
- [Reg M-B snapshot, MetaVGC](https://metavgc.com/guides/pokemon-champions-regulation-m-b-legal-pokemon-items-moves): community list of legal Pokémon, items and moves
- [pokemon-champions-data, GitHub](https://github.com/otterlyclueless/pokemon-champions-data): CC BY 4.0 repo whose learnsets are based on main-series data
- [PokeAPI overview, APIs.io](https://apis.io/providers/pokeapi/): open-source API; encourages client-side caching
- [Silph Scope](https://silph-scope.com): fan-made Champions ladder and tools site; its privacy policy says its team builder runs entirely in the browser
- [MunchStats FAQ](https://munchstats.com/about/): describes capturing in-game Champions data by automation
- [Incineroar build guide, GamingProMax](https://gamingpromax.com/pokemon-champions-best-builds-incineroar/): reports Incineroar lost Knock Off in Champions
