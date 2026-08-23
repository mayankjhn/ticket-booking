# Ticket Booking System 🎫

A comprehensive full-stack ticket booking platform for movies and concerts featuring real-time interactive seat maps, automated seat holds, waitlist auto-assignment, and QR code ticketing.

## Features

- **Interactive Seat Map**: Visual grid showing real-time seat availability (Available, Held, Booked).
- **Automated Seat Holds (TTL)**: Seats are temporarily held when selected, preventing concurrent bookings. Abandoned checkouts auto-release the seat after a configurable TTL (e.g., 10 minutes).
- **Concurrency Protection**: Robust checks ensure simultaneous booking attempts for the same seat cannot succeed.
- **Smart Waitlist System**: Sold-out events offer waitlists. Cancellations trigger an automatic, time-limited offer to the next user in the queue.
- **QR Code Ticketing**: Confirmed bookings generate and email a unique QR code ticket to the user.
- **Role-Based Access Control**: Tailored portals for Customers, Organisers (event creation, analytics), and Admins (venue and category management).

## Tech Stack

- **Frontend & API**: [Next.js](https://nextjs.org/) (App Router), React, TypeScript, Tailwind CSS
- **Database**: PostgreSQL with [Prisma ORM](https://www.prisma.io/)
- **Caching & Caches**: Redis (via `ioredis`) for seat holds and queues
- **Authentication**: NextAuth.js (JWT)
- **Email Delivery**: Nodemailer / Resend

## Setup Guide

### 1. Prerequisites
- Node.js (v18+)
- PostgreSQL database
- Redis instance

### 2. Installation

Clone the repository and install dependencies:
```bash
git clone https://github.com/mayankjhn/ticket-booking.git
cd ticket-booking
npm install
```

### 3. Environment Variables
Create a `.env` file in the root directory and add the following variables based on the provided `.env.example`:

```bash
cp .env.example .env
```
Fill in your Postgres connection URL, Redis URL, NextAuth secret, and Email SMTP credentials.

### 4. Database Setup
Push the schema to your PostgreSQL database and seed the initial data:
```bash
npx prisma db push
npx prisma db seed
```

### 5. Running the Application
Start the development server:
```bash
npm run dev
```
Open [http://localhost:3000](http://localhost:3000) to view the application.

---

## DB Schema (Prisma)

Here is a high-level overview of our core data models:

- **User**: Stores Customer, Organiser, and Admin profiles with RBAC.
- **Venue**: Represents physical locations (has layout maps).
- **Event**: Specific shows happening at a Venue (movies, concerts).
- **SeatCategory**: Pricing tiers (e.g., VIP, Standard) mapped to an Event.
- **Seat**: Represents physical seats in a Venue.
- **Booking**: Links a User to an Event and specific Seats (Status: Pending, Confirmed, Cancelled).
- **Waitlist**: Queues Users for specific Seat Categories when sold out.

---

## Architecture & Logic Explanations

### Seat Hold TTL & Concurrency Protection

To solve high-demand concurrency, this platform relies heavily on **Redis** for state management:
1. **Hold Initiation**: When a user clicks a seat, the backend attempts to set a key in Redis: `seat:{eventId}:{seatId}` using the `NX` (Not Exists) argument alongside an `EX` (Expire) of 600 seconds (10 mins).
2. **Concurrency**: The `NX` flag guarantees that if two users click simultaneously, only the first request successfully writes the key and acquires the "hold". The second request fails instantly.
3. **Auto-Release**: If the user abandons the checkout, the Redis key automatically expires after 10 minutes, instantly reverting the seat's status on the frontend to "Available" via WebSockets/Polling.
4. **Completion**: If checkout is successful, the booking is committed to PostgreSQL, and the Redis hold key is explicitly deleted.

### Waitlist Auto-Assignment & Time-Limited Offers

1. **Queueing**: If a category is sold out, users can opt into a waitlist. Their `userId` is pushed into a Redis List (or a Postgres Waitlist table sorted by timestamp).
2. **Trigger**: When a customer cancels a confirmed booking, the backend queries the waitlist for that specific event and category.
3. **Offer Creation**: The system dequeues the first user, generates a secure, time-limited JWT offer token (valid for e.g., 15 mins), and sends an email notification.
4. **Resolution**: 
   - If the user buys the ticket, the booking completes.
   - If the JWT expires before purchase, a cron job or lazy evaluation passes the seat to the *next* user in the queue.

---

## API Documentation

- `POST /api/auth/*` - NextAuth routes for login/registration
- `GET /api/events` - Fetch available events with filtering
- `GET /api/events/:id/seats` - Fetch real-time seat availability grid
- `POST /api/bookings/hold` - Initiate a Redis seat hold
- `POST /api/bookings/confirm` - Finalize payment and commit booking
- `POST /api/waitlist/join` - Join the waitlist for a category

*Note: For the full system design document as requested in the deliverables, please see the PDF or additional `SYSTEM_DESIGN.md`.*
