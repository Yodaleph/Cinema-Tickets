# Cinema Ticket Booking

A real-time cinema seat booking app built with **Go** and **Redis**. When many people try to book the same seat at the same moment, exactly one of them gets it. There are no double bookings and no manual locking code.

<!-- Add a screenshot here once you have one: -->
<!-- ![Seat selection screen](docs/screenshot.png) -->

## Why this project

Booking systems have a classic concurrency and parallelism problem: booking tickets. Two people click the same seat at the same instant. This project solves it with a **temporary seat hold**:

1. Clicking a seat reserves it for **2 minutes**.
2. The user can **confirm** (the booking becomes permanent) or **release** it.
3. If they do nothing, the hold **expires on its own** and the seat becomes available again.

The hold uses Redis `SET ... NX` with a TTL. `NX` means "only set if this key doesn't exist". Redis runs the command as a single atomic operation, so concurrent requests can't both succeed.

## Features

- Seat map with live status: available, held by you, held by someone else, confirmed
- Atomic seat holds with automatic 2-minute expiry
- Confirm or release a hold
- The page polls every 2 seconds so you see other people's holds appear
- A test that fires 100,000 concurrent booking attempts at one seat and checks that exactly one succeeds
- Plain HTML and JavaScript frontend, served by the Go server (no build step)

## Tech stack

| Part | Choice |
|---|---|
| Language | Go 1.25 (standard library `net/http` routing, no web framework) |
| Data store | Redis 7 |
| Frontend | Vanilla HTML, CSS and JavaScript |
| Local infrastructure | Docker Compose (Redis plus a Redis web viewer) |

## Architecture

```
Browser (static/index.html)
   | HTTP + JSON
handler.go        parses requests, writes responses
   |
service.go        business logic layer
   |
BookingStore      Go interface: Book, ListBookings, Confirm, Release
   |
redis_store.go    Redis implementation
```

The storage layer is hidden behind the `BookingStore` interface, so the Redis implementation can be swapped without changing the handlers. The repo also contains simpler in-memory stores that show how the design developed.

```
cmd/main.go                  server entry point and routes
internal/booking/            domain types, handlers, service, stores, tests
internal/adapters/redis/     Redis client setup
internal/utils/              JSON helpers
static/index.html            the frontend
docker-compose.yaml          Redis and Redis Commander for local use
```

## Run it locally

**You need:** [Go 1.25+](https://go.dev/dl/) and [Docker Desktop](https://www.docker.com/products/docker-desktop/).

```bash
User A ──► read seat A1 → "free" ──► write booking ──► success
User B ──► read seat A1 → "free" ──► write booking ──► ???
```
# 1. Clone the repo
git clone https://github.com/YOUR-USERNAME/cinema-ticket-booking.git
cd cinema-ticket-booking

# 2. Start Redis
docker compose up -d

# 3. Run the app
go run ./cmd/main.go
```

Open **http://localhost:8080**. A Redis web viewer is at http://localhost:8081, where you can watch seat keys appear and expire.

Run the tests (Redis must be running):

```bash
go test ./internal/booking/
```

## What I learned and what's next

Working through this project taught me about atomic operations, TTL-based state, interface-driven design in Go, and how a frontend and a Go JSON API fit together.

**Planned features**

- [ ] Admin sign-in for people who sell tickets
- [ ] A page where an admin registers a movie to sell tickets for
- [ ] A shareable link per movie that opens the seat-booking page for buyers
- [ ] Proper error responses (`409 Conflict` when a seat is taken)
- [ ] Verify that the user confirming or releasing a seat is the one who held it
- [ ] Redis address from an environment variable, plus a Dockerfile for deployment
