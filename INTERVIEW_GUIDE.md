# SeatFlow Interview Guide

## 30-second introduction

SeatFlow is a full-stack event and movie ticket-booking MVP. Users can discover events, select a show, choose seats, complete a simulated payment, and view their bookings. The central backend problem is preventing double booking, so seat holds and final availability checks are handled by the Express API rather than trusted to the React client.

## Stack

- React and Vite for the responsive client
- Express and Node.js for REST APIs
- JWT authentication
- bcrypt password hashing
- Lucide icons and custom CSS for the product UI
- In-memory seeded data for zero-setup demonstration
- Planned production replacements: PostgreSQL and Redis

## Architecture explanation

```text
Browser
  |
  v
React + Vite
  |
  v
Express REST API
  |-- JWT auth middleware
  |-- Admin role middleware
  |-- Event/show APIs
  |-- Seat lock service
  `-- Booking service
```

The client handles presentation and navigation. The server owns identity, role checks, seat availability, pricing, and booking creation.

## Main user journey

1. The home page loads seeded events from `GET /api/events`.
2. The user opens an event and the client loads shows from `GET /api/shows/:eventId`.
3. The seat screen loads availability from `GET /api/shows/:showId/seats`.
4. The user signs in if needed. The API returns a JWT after bcrypt verifies the password.
5. The client requests a temporary hold through `POST /api/seats/lock`.
6. Checkout calls `POST /api/bookings`.
7. The server verifies every hold, calculates the total, marks seats booked, removes locks, and returns a booking id.
8. The client shows confirmation and later loads the user's bookings through `GET /api/bookings`.

## The concurrency story

A seat has three important states:

- Available: no booking and no active hold
- Held: temporarily reserved for one user for five minutes
- Booked: permanently assigned to a confirmed booking

If User A holds A4, User B cannot hold A4 because the server sees the existing lock. The lock includes the show id, seat id, user id, and expiry time. Expired locks are removed when seat availability or lock creation is checked.

The final booking request does not trust the frontend selection. It verifies that every requested seat still has a lock owned by the current user. This prevents a user from changing the request body or trying to book after the hold expires.

## Authentication and authorization

Passwords are hashed with bcrypt and never returned by the API. On login, the API signs a JWT with the user id and role. Protected routes verify the token with middleware. Admin routes run an additional role check. The frontend hides admin navigation for normal users, but backend authorization is still enforced if someone manually calls the endpoint.

## Why the MVP uses memory

The project had a short build and demonstration window. In-memory collections remove database credentials and setup failures, so the complete journey runs immediately. The tradeoff is that data resets on server restart and separate API instances do not share locks.

For production:

- Store users, shows, seats, and bookings in PostgreSQL.
- Use a database transaction when converting a hold into a booking.
- Use Redis `SET key value NX EX 300` for distributed seat locks.
- Add a payment provider with server-side webhooks.
- Use secure cookies or short-lived access tokens with refresh tokens.
- Add request validation, rate limiting, Helmet, audit logs, and monitoring.

## Five-minute presentation

### Minute 1: Product

Show the home page, search, category filters, event cards, and the original SeatFlow visual identity.

### Minute 2: Browse to seats

Open an event, choose a venue/show, and explain that show data comes from the API. Open the seat map and point out available, booked, held, and selected states.

### Minute 3: Authentication and locking

Sign in with `user@seatflow.com` / `User@123`. Select two seats. Explain that the selection is held on the server for five minutes, not merely colored blue in the browser.

### Minute 4: Checkout

Show the server-calculated ticket total and simulated payment. Complete the booking and point out the generated booking id and seat list.

### Minute 5: Admin and scaling

Sign out, sign in with `admin@seatflow.com` / `Admin@123`, show the dashboard and system status, then explain that Redis and PostgreSQL are the production replacements for the local lock map and seeded collections.

## Strong closing statement

"I optimized this version for a reliable end-to-end demonstration. The important business rule is enforced at the API boundary: a booking is only created when the server confirms ownership of every active seat hold. The next production step would be replacing the in-memory state with Redis and transactional PostgreSQL while keeping the client contract unchanged."
