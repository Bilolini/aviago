# Airline Operations Console

**A full reservation workflow, from recurring flight schedules to seat holds, ticketing, cancellations, and revenue reporting.**

![React](https://img.shields.io/badge/React-18-149ECA?logo=react&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5-646CFF?logo=vite&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-ES_Modules-F7DF1E?logo=javascript&logoColor=black)
![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3FCF8E?logo=supabase&logoColor=white)

Built for CSE 305, this project connects an interactive React interface to a relational airline database in Supabase. Customers can compare direct and connecting routes, select seats for each flight, complete a timed checkout, and manage their bookings. Staff can create recurring schedules, generate flights with priced seat inventory, and inspect financial trends.

The central engineering problem is **coordinating changing seat inventory across search, checkout, payment, and cancellation**. The application uses PostgreSQL RPC functions for reservation operations and React for the customer and staff workflows.

[Features](#features) · [Architecture](#architecture) · [Engineering Decisions](#engineering-decisions) · [Run Locally](#run-locally) · [Demo Scope](#demo-scope-and-next-steps)

## Features

### Route search and comparison

Search by departure airport, arrival airport, and departure date using airport data loaded from the database.

- Filter by **Economy, Business, or First Class**.
- Set the number of adults and children.
- Include direct flights or routes with **one, two, or three stops**.
- Compare airline names, departure and arrival times, total journey duration, connection airports, and itinerary prices.
- See full airport names on hover and flight-by-flight layover details during checkout.
- Show a route only when every leg has enough available seats in the selected cabin for the requested party.

Results support four rankings:

| Ranking | Selection logic |
| --- | --- |
| Cheapest | Total cabin fare, then duration and stop count |
| Fastest | Total journey duration, then price and stop count |
| Fewest stops | Stop count, then duration and price |
| Best comfort | Weighted stop count, waiting time when supplied by the RPC, and journey duration; price breaks ties |

The comfort score is a deterministic heuristic: `stops × 100000 + waitMinutes × 100 + durationMinutes`. It strongly favors fewer transfers; it does not compute a Pareto frontier.

### Three-step checkout

**1. Passenger information**

Existing customers can log in or reuse their saved customer session. New passengers enter their first name, last name, email, and Korean-format phone number (`+82`). Saving information is optional: providing a password creates a reusable login. Guest checkout still creates the account record required by the database, without saving a customer session.

**2. Seat selection**

A horizontal cabin diagram displays selectable seats and disabled unavailable positions. The layout is inferred from seat labels rather than a manufacturer-certified cabin map. Connecting itineraries support up to four flight segments, with a separate seat selection for each flight and traveler. The next-flight control activates once enough seats are selected for the current segment.

Proceeding from this step creates database booking holds for the selected seats.

**3. Payment and ticketing**

A **15-minute countdown** displays the booking hold window. Users choose credit card, mobile pay, bank transfer, or PayPal, then confirm payment. The database records payment, confirms the booking, marks the seat booked, and issues a ticket. These are simulated payment methods; no external payment gateway is connected.

Customers who logged in or saved their information are taken to their Account after successful payment.

### Customer account and trip management

- Shared customer state across Account and Checkout avoids repeated login prompts.
- Separate **Pending, Current, and Past** booking views keep the account easy to scan.
- Pending bookings can be paid from the account.
- Expanded bookings reuse the checkout itinerary components.
- Related connecting-flight bookings are grouped into a trip in the UI.
- A single **Cancel Trip** action previews aggregated fees and refunds before cancelling eligible segments.
- Partially cancelled trips identify the cancelled flight in the itinerary.
- The customer summary displays accumulated refund-record amounts.
- Logout removes the saved customer session from localStorage.

### Staff schedule and inventory management

A lightweight staff-ID entry reveals the Staff and Revenue sections.

Staff first creates a recurring schedule by selecting an airline, flight number, existing airport pair, departure/arrival times, validity dates, and one operating weekday. They then choose that schedule, an aircraft from the airline, a generation date range, and fares for each cabin class.

The database generates individual flights and their seat inventory. A results table shows generated flight IDs, schedule IDs, aircraft, timestamps, and statuses.

### Revenue dashboard

Select a date range and aggregate by **day, month, quarter, year, or overall**.

The dashboard displays booking, payment, cancellation, and refund counts alongside gross revenue, refunds, net revenue, cancellation rate, and refund rate. Custom SVG line charts show bookings and cancellations; a revenue-flow visualization compares net revenue and refunds against gross revenue.

## Architecture

```text
React SPA
  Search | Checkout | Account | Staff | Revenue
                         |
               src/lib/airlineApi.js
             Queries and RPC adapters
                         |
                 Supabase JS client
                         |
           Supabase PostgreSQL database
       Tables + relationships + views + functions
                         |
       Route discovery, inventory, reservations,
         ticketing, refunds, and reporting
```

| Layer | Responsibility |
| --- | --- |
| React 18 | Components, form state, navigation, checkout steps, seat selection, and reporting UI |
| Vite 5 | Local development server and production bundling |
| JavaScript + CSS | Route ranking, itinerary presentation, responsive layouts, and custom charts |
| Lucide React | Consistent interface icons |
| Supabase JS | Relational queries and PostgreSQL RPC calls |
| PostgreSQL | Persistent data, recursive search, row locking, reservation transitions, and reporting views |

There is no separate Express or FastAPI service. The API adapter keeps Supabase calls outside the UI components, while database functions perform the reservation mutations. Customer and staff session data are persisted separately in localStorage; route-search results stay in memory.

### Relational model

| Area | Main entities | Design purpose |
| --- | --- | --- |
| Network | `airport`, `route`, `airline` | Valid airport pairs, airport timezones, and operating airlines |
| Scheduling | `flight_schedule`, `flight_schedule_day`, `flight` | Separate recurring plans from dated flight instances |
| Inventory | `aircraft`, `aircraft_model_seat`, `seat_class`, `flight_seat` | Reuse seat templates while tracking availability and price per flight |
| Reservations | `account`, `customer`, `booking`, `payment`, `ticket` | Connect passenger identity, a selected seat, payment, and ticket issuance |
| Cancellation | `cancellation_policy`, `refund` | Calculate fees and retain refund records |
| Reporting | `v_revenue_booking_fact` | Provide a reporting basis for financial aggregation |

### Database operation contracts

The connected Supabase project supplies these functions:

| Function | Responsibility |
| --- | --- |
| `find_available_routes` | Discover direct and connecting itineraries through recursive SQL |
| `get_route_details` | Add per-leg cabin availability, seat lists, and route price summaries |
| `generate_flight_from_flight_schedule_date_range` | Generate dated flights and priced seats from a recurring schedule |
| `create_booking` | Lock an available seat and create a pending booking with a 15-minute hold |
| `pay_booking` | Record payment, confirm a held booking, book its seat, and issue a ticket |
| `calculate_cancellation_fee` | Calculate the applicable fee and refundable amount before confirmation |
| `cancel_booking` | Cancel a booking, release the seat, void the ticket, and create a refund request |
| `expire_booking_holds` | Expire unpaid holds and release held inventory |
| `get_revenue_summary` | Aggregate booking activity and financial totals by reporting period |

The API adapter also contains a `recommend_route` wrapper; the current search interface uses the four local ranking controls.

## Engineering Decisions

### Bound recursive route discovery

The documented search function limits journeys to **three stops**, permits connections of **at least one hour and less than 24 hours**, prevents repeated airports and flights, and returns at most **200 candidates**. The first departure is matched against the departure airport's local date. Each leg must have available inventory and a `SCHEDULED` or `BOARDING` status.

These constraints bound recursion and exclude cyclic or impractical connections. The frontend then filters the returned candidates for cabin and party size.

### Reserve inventory with database row locks

A search result is a snapshot, not a seat reservation. The documented booking function uses `FOR UPDATE` to lock the selected seat before validating availability and moving it into a held state. Competing requests are resolved at the database boundary.

```text
Seat:     AVAILABLE -> HELD -> BOOKED
                       |         |
                  hold expires   cancellation
                       |         |
                       +----> AVAILABLE

Booking:  PENDING_PAYMENT -> CONFIRMED -> CANCELLED
                 |
              EXPIRED
```

### Separate the hold from payment

A temporary hold gives the customer time to finish checkout. `create_booking` reserves inventory; `pay_booking` performs payment recording, booking confirmation, and ticket issuance within one database call.

**Transaction boundary:** each booking RPC is atomic. Multi-seat and connecting-trip operations currently call these functions sequentially from the frontend, so the entire trip is not one all-or-nothing transaction.

### Reuse templates, preserve per-flight inventory

`aircraft_model_seat` provides reusable seat definitions. Generation creates `flight_seat` instances with independent prices and statuses, so booking one flight does not consume the corresponding seat on another flight using the same aircraft model.

### Preserve seat-level bookings, present trip-level journeys

Each booking refers to one selected flight seat. The account UI groups related rows using booking timestamps, airport continuity, and connection timing. This makes individual seat and ticket states explicit while giving customers a whole-trip view. A persistent itinerary ID would make grouping more robust as the system grows.

## Run Locally

### Prerequisites

- Node.js and npm.
- A Supabase project with the airline schema, relationships, RPC functions, reporting view, and reference data already provisioned.
- Database permissions and RLS policies appropriate to the configured client.

**Repository scope:** this repository contains the frontend and API adapter. The SQL migrations, function definitions, and seed data are not currently versioned here. A fresh Supabase project will need those database objects before the full workflows can run.

### Installation

```bash
git clone https://github.com/Bilolini/cse305-airline-console.git
cd cse305-airline-console
npm ci
```

Create `.env.local` in the project root:

```dotenv
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-anon-key
```

Use a browser-safe anon key; keep service-role credentials out of the frontend. Vite environment variables are included in the client bundle.

Start development:

```bash
npm run dev
```

Open the localhost URL printed by Vite. Reference data loads from Supabase at startup.

| Command | Purpose |
| --- | --- |
| `npm ci` | Install dependencies from the lockfile |
| `npm run dev` | Run the local Vite server |
| `npm run build` | Build production assets into `dist/` |
| `npm run preview` | Serve the production build locally |

### Suggested walkthrough

1. Enter a demo Staff ID, create a schedule for an existing route, and generate flights on a matching weekday.
2. Search the generated departure date and compare cabin, stops, and ranking modes.
3. Enter passenger information or log in, select seats on each leg, and proceed to the timed payment step.
4. Confirm simulated payment and inspect the trip in Account.
5. Preview a cancellation, confirm it, and inspect the booking and refund records.
6. Open Revenue and compare booking activity, cancellations, and financial totals for the period.

## Code Map

```text
src/
  App.jsx              Application shell, customer/staff workflows,
                       itinerary ranking, seat layouts, and charts
  main.jsx             React entry point
  styles.css           Shared UI styles and responsive layouts
  lib/
    airlineApi.js      Supabase table queries and RPC adapters
    supabase.js        Client initialization and environment checks
    format.js          Date, currency, and percentage formatting
```

Useful review entry points: [API integration](src/lib/airlineApi.js), [application workflows](src/App.jsx), and [shared styling](src/styles.css).

## Demo Scope And Next Steps

This is an academic reservation-system prototype with explicit paths toward production hardening:

- **Identity and permissions:** customer login currently checks the account table in the browser, and staff entry accepts any nonempty ID. Saved localStorage profiles and hidden tabs provide UI convenience, not authenticated authorization. A production version needs managed authentication, verified staff roles, and database-enforced access policies.
- **Payments and refunds:** payment methods simulate database operations. Cancellation creates refund requests; displaying refund-record totals does not imply money has been transferred. A gateway integration would require verified webhooks and idempotent processing.
- **Trip transactions:** move multi-leg booking, payment, and cancellation into coordinated backend operations, with persistent itinerary identifiers and recovery for partial failures.
- **Timezones:** timestamps currently display in the browser timezone. The documented generator interprets both schedule times in the departure airport timezone; airport-local arrival times need explicit handling.
- **Reproducibility and verification:** version database migrations and seed data, then add integration tests for competing seat reservations, expired holds, failed payments, and cancellation rollback.

The project brings together relational modeling, recursive SQL, transactional inventory handling, React workflow design, and financial reporting in one connected application.
