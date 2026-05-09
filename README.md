# SkyNova Airlines Agent

Build a ReAct agent that answers passenger service and operations
questions for SkyNova Airlines. The agent draws on three data sources
and picks the right tool for each question.

## Data sources

| Store | File | Contents |
|---|---|---|
| PDF (RAG) | `skynova_handbook.md` + `skynova_handbook.pdf` | Baggage, refund, loyalty, special assistance, boarding, in flight, and contact policies. |
| Supabase (SQL) | `supabase_seed.sql` | 25 customers, 8 airports, 5 aircraft, 15 flights, 50 bookings. Relational and joinable. |
| MongoDB | `mongodb_seed.json` | 20 support tickets, 27 flight reviews, 40 activity logs. Nested or variable shape. |

The data is internally consistent. Every `customer_id` in MongoDB
maps to a row in `customers`. Every `booking_reference` in tickets
matches `bookings.booking_reference`. Ticket events line up with
flight statuses (SN301 was cancelled and three customers filed
tickets; SN401 was delayed and five reviews mention the delay).

The fictional "today" for this dataset is **2026-05-08** so there
are completed past flights, a flight currently in the air, and
scheduled future flights.

## Example questions the agent should answer

These need two or three tools in a single ReAct loop:

1. **"Customer 3 complained about flight SN401. What is our delay
   compensation policy, what did they actually fly, and what is the
   ticket status?"**
   Tools used: RAG (handbook section 3.5) + SQL (bookings join
   flights) + Mongo (support_tickets).

2. **"For SN301, who was affected, what is our cancellation policy,
   and what did the affected passengers say?"**
   Tools used: SQL (bookings on flight 5) + RAG (handbook section
   3.4) + Mongo (tickets and reviews).

3. **"How much has Aarav Mehta spent with us this year, what tier is
   he, and what miles bonus should his Business class trip have
   earned per the program?"**
   Tools used: SQL (sum fares for customer 1) + RAG (loyalty earn
   rates) + Mongo (TCK-1007 about missing miles).

4. **"Aisha Khan has a wheelchair assistance request. What is the cut
   off for booking that, and is her flight still scheduled?"**
   Tools used: RAG (section 5.1) + SQL (flight for booking
   SKYA4H5KF) + Mongo (TCK-1009 status).

5. **"Which flights had the lowest average ratings recently, and what
   did passengers complain about?"**
   Tools used: Mongo (aggregate `flight_reviews`) + Mongo
   (`support_tickets` filtered by the same flight numbers).

Single source questions are also fair game:

- "How many Platinum customers do we have?" -> SQL only.
- "What is our pet travel policy?" -> RAG only.
- "List all open support tickets." -> Mongo only.

## How to use this repo

1. Build the handbook PDF (see below).
2. Load `supabase_seed.sql` into Supabase. See `SUPABASE_SETUP.md`.
3. Load `mongodb_seed.json` into MongoDB Atlas or local Mongo. See
   `MONGODB_SETUP.md`.
4. Read `SCHEMA.md` before wiring the agent. It documents every
   table and collection and gives you 14 worked NL to query examples
   you can drop into the agent prompt as few shot.

Then build the agent. `SCHEMA.md` section 0 walks through three ways
to ground the agent in the schema: static system prompt context, a
`describe_schema` tool the agent calls per turn, or retrieval based.

## Build the handbook PDF

```bash
uv add reportlab
uv run python build_pdf.py
# writes skynova_handbook.pdf
```

For RAG you can chunk the markdown directly (cleaner text) or load
the PDF with `pypdf`, `pdfplumber`, or `unstructured`. Both give
equivalent content.

## Sanity checks

After loading both stores, run these to confirm everything is in
place.

In Supabase SQL editor:

```sql
SELECT COUNT(*) FROM customers;       -- 25
SELECT COUNT(*) FROM bookings;        -- 50
SELECT loyalty_tier, COUNT(*) FROM customers GROUP BY 1;
SELECT status, COUNT(*) FROM flights GROUP BY 1;
```

In `mongosh`:

```js
use skynova
db.support_tickets.countDocuments()         // 20
db.flight_reviews.countDocuments()          // 27
db.user_activity_logs.countDocuments()      // 40
```
