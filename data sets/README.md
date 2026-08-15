# Global Reach Airlines — Raw Data Sources

## Overview

This folder contains the **raw, source-system-level extracts** used as the input layer for the Gulf Falcon Airways Power BI project. The dataset is entirely **synthetic** — it was generated to simulate the operations of a fictional mid-sized Gulf airline (2022–2025), and does not represent any real airline, customer, or flight.

Unlike a typical "clean" portfolio dataset, these files are intentionally structured to reflect how data actually arrives in a real airline's analytics environment: **fragmented across multiple operational systems**, each with its own naming conventions, formats, and quality issues. The files here are the *inputs* to the Power Query transformation layer, not the final analytical model. The reconciled, modeled output (star schema) lives in the `.pbix` file itself.

The purpose of documenting the raw layer separately is to make the ETL logic auditable: anyone reviewing this repository can trace exactly which quality issue in the source maps to which transformation step in Power Query.

---

## Source System Map

The 12 files in this folder originate from **six simulated source systems**. Two of them (aircraft, airport, cabin class, booking channel, date, loyalty tier) are reference/lookup data and required no transformation; the remaining four systems each export data in a different shape, requiring merge, append, and standardization logic before they can be loaded into the star schema.

| # | Simulated System | Files | Update Pattern | Key Identifier |
|---|---|---|---|---|
| 1 | CRM (Customer Master) | `customer_profiles.csv` | Slowly changing, one row per customer | `customer_id` |
| 2 | Loyalty Platform | `loyalty_members.csv` | Independent membership export | `CustomerID` |
| 3 | Flight Scheduling System | `flights_schedule.csv` | Published ahead of operations | `flight_id` |
| 4 | Ops Monitoring System | `flights_ops_performance.csv` | Populated after flight completion | `FlightID` |
| 5 | Online Booking Engine (Website + Mobile App) | `bookings_online.csv` | Real-time transactional | `booking_id` |
| 6 | Offline Booking Channels (Travel Agents + Call Center) | `bookings_offline.csv` | Batch export, manual entry origin | `BookingID` |
| — | Reference / Lookup Data (unchanged) | `dim_aircraft.csv`, `dim_airport.csv`, `dim_booking_channel.csv`, `dim_cabin_class.csv`, `dim_date.csv`, `dim_loyalty_tier.csv` | Static / rarely changing | Respective `*_id` |

**Why this matters:** systems 1–2, 3–4, and 5–6 are each *pairs* that describe the same real-world entity (a customer, a flight, a booking) but were extracted from different systems at different times, with different owners. This is the source of every overlap and inconsistency documented below.

---

## File-by-File Reference

### 1. `customer_profiles.csv` — CRM Export
299,992 rows · one row per customer

| Column | Type | Description |
|---|---|---|
| `customer_id` | Integer (PK) | Unique customer identifier |
| `full_name` | Text | Customer name (synthetic) |
| `gender` | Text | Male / Female |
| `date_of_birth` | Date (YYYY-MM-DD) | Used to derive age segments |
| `nationality` | Text | Customer nationality |

**Known issues:** None — this file reflects a well-governed CRM export.

---

### 2. `loyalty_members.csv` — Loyalty Platform Export
300,032 rows

| Column | Type | Description |
|---|---|---|
| `CustomerID` | Integer | Foreign key to `customer_profiles.customer_id` |
| `LoyaltyTierID` | Integer | Foreign key to `dim_loyalty_tier.tier_id` |
| `JoinDate` | Text, `dd/mm/yyyy` | Date the customer joined the loyalty program |

**Known issues:**
- **Header casing mismatch:** PascalCase (`CustomerID`) instead of the CRM's snake_case (`customer_id`) — reflects a different vendor/system exporting the file.
- **Date format mismatch:** `dd/mm/yyyy` instead of ISO `yyyy-mm-dd` used elsewhere in the project — requires explicit locale handling on import, not a plain type-cast.
- **~40 exact duplicate rows** — a common artifact of loyalty platform batch exports (e.g., re-sent records after a failed sync).

**Overlap with `customer_profiles.csv`:** Both files describe the *same customer population* and share a common key (`customer_id` / `CustomerID`), but neither file alone is a complete customer record — one holds demographic attributes, the other holds loyalty attributes. They must be merged (not appended) into a single customer dimension.

---

### 3. `flights_schedule.csv` — Scheduling System Export
72,624 rows · one row per scheduled flight

| Column | Type | Description |
|---|---|---|
| `flight_id` | Integer (PK) | Unique flight identifier |
| `flight_number` | Text | Commercial flight number (`R` suffix = return leg) |
| `date_id` | Integer | Foreign key to `dim_date.date_id` |
| `origin_airport_id` / `destination_airport_id` | Integer | Foreign keys to `dim_airport.airport_id` |
| `aircraft_id` | Integer | Foreign key to `dim_aircraft.aircraft_id` |
| `scheduled_departure` / `scheduled_arrival` | Datetime | Planned times |
| `scheduled_duration_minutes` | Integer | Planned flight duration |
| `direction` | Text | `Outbound` (from hub) / `Inbound` (to hub) |
| `origin_city`, `origin_country`, `destination_city`, `destination_country` | Text | **Denormalized** copies of `dim_airport` attributes |

**Known issues:**
- **Denormalization:** the four `*_city` / `*_country` columns duplicate data already available in `dim_airport`. This mirrors a common real-world pattern where a scheduling system snapshots reference data at export time rather than referencing it live — the values should be validated against `dim_airport` and dropped from the final model rather than trusted as a second source of truth.

**Overlap with `flights_ops_performance.csv`:** Both files describe the *same flight population* (`flight_id` = `FlightID`) but split it by lifecycle stage: this file exists before departure, the ops file is only populated after the flight is flown.

---

### 4. `flights_ops_performance.csv` — Ops Monitoring Export
72,624 rows

| Column | Type | Description |
|---|---|---|
| `FlightID` | Integer | Foreign key to `flights_schedule.flight_id` |
| `DelayMinutes` | Integer | Minutes of delay (0 if on time) |
| `FlightStatus` | Text | `ON TIME` / `Delayed` / `CANCELLED` |
| `SeatsSoldEconomy`, `SeatsSoldBusiness`, `SeatsSoldFirst` | Integer | Seats sold per cabin, per the Ops system |
| `LoadFactorPct` | Decimal | Aircraft load factor (%) |

**Known issues:**
- **Inconsistent categorical casing:** `FlightStatus` mixes full-caps (`ON TIME`, `CANCELLED`) with title case (`Delayed`) — must be standardized before it can be used in any filter, slicer, or measure logic; an un-normalized string comparison will silently return blank results rather than an error.
- **PascalCase headers**, again reflecting a different exporting system than the scheduling file.

**Important operational note:** the seat counts in this file come from the **Ops system**, while seat counts implied by `bookings_online.csv` / `bookings_offline.csv` come from the **Booking system**. These two figures are not guaranteed to reconcile exactly — this is intentional and mirrors how real airlines run operational and commercial systems independently. The gap between them is itself a valid analytical insight, not a data error to be silently corrected.

---

### 5. `bookings_online.csv` — Online Booking Engine (Website + Mobile App)
620,917 rows

| Column | Type | Description |
|---|---|---|
| `booking_id` | Integer (PK) | Unique booking transaction |
| `flight_id` | Integer | Foreign key to `flights_schedule.flight_id` |
| `customer_id` | Integer | Foreign key to the customer dimension |
| `booking_date_id` | Integer | Foreign key to `dim_date.date_id` |
| `cabin_id` | Integer | Foreign key to `dim_cabin_class.cabin_id` |
| `channel_name` | Text | `Website` or `Mobile App` (already resolved to text) |
| `seats_booked` | Integer | Seats in this transaction (1–4) |
| `fare_amount` | Decimal | Ticket revenue for the transaction |
| `ancillary_revenue` | Decimal | Extra revenue (baggage, meals, seat selection) |

**Known issues:** None structural — this is the cleanest of the four transactional exports, consistent with an online system where data entry is automated end-to-end.

---

### 6. `bookings_offline.csv` — Travel Agent + Call Center Export
207,060 rows

| Column | Type | Description |
|---|---|---|
| `BookingID` | Integer | Unique booking transaction |
| `FlightID` | Integer | Foreign key to `flights_schedule.flight_id` |
| `CustomerID` | Integer | Foreign key to the customer dimension |
| `BookingDate` | Text, `dd/mm/yyyy` | Booking date **as a real date**, not a surrogate key |
| `CabinClass` | Text | `Economy` / `Business` / `First` (as a label, not an ID) |
| `ChannelName` | Text | `Travel Agent` / `Call Center` |
| `SeatsBooked` | Integer | Seats in this transaction |
| `FareAmount` | Text, e.g. `"$585.25"` | **Currency-formatted string**, not a numeric type |
| `AncillaryRevenue` | Decimal, ~1% null | Missing for a small share of records |

**Known issues:**
- **`FareAmount` is stored as text** with a `$` symbol — this is common when the source is a manual-entry system (call center agents) or a spreadsheet export that applied currency number formatting, which flattens into a display string on export. Requires explicit text-to-number cleanup.
- **`BookingDate` is a literal date string**, not the `booking_date_id` surrogate key used elsewhere — this file was clearly extracted independently of the internal date dimension and needs to be resolved against `dim_date.full_date` before it can join to the rest of the model.
- **`CabinClass` and `ChannelName` are text labels**, not the `cabin_id` / `channel_id` surrogate keys used in `bookings_online.csv` — must be resolved against `dim_cabin_class` and `dim_booking_channel` respectively.
- **~1% missing `AncillaryRevenue`** — plausible for a call center workflow where an agent completes the fare booking but does not always log ancillary add-ons.

**Overlap with `bookings_online.csv`:** both files describe the *same booking transaction population*, split by channel of origin. They are conceptually one fact table (`fact_bookings`) that was exported from two different points of sale with two different schemas, and must be **standardized to a common column set and appended**, not merged on a shared key.

---

## Cross-System Overlaps at a Glance

| Real-world entity | Appears in | Shared key | Reconciliation needed |
|---|---|---|---|
| Customer | `customer_profiles.csv` + `loyalty_members.csv` | `customer_id` / `CustomerID` | Merge, dedupe, standardize date format |
| Flight | `flights_schedule.csv` + `flights_ops_performance.csv` | `flight_id` / `FlightID` | Merge, standardize status casing |
| Booking | `bookings_online.csv` + `bookings_offline.csv` | *(no shared key — same grain, different channel)* | Standardize column names/types, append |
| Seats sold | `flights_ops_performance.csv` (Ops) vs. bookings files (Booking system) | `flight_id` | **Not reconciled by design** — the two figures are expected to diverge slightly, reflecting two independently operated systems |
| Airport attributes | `dim_airport.csv` (source of truth) vs. `flights_schedule.csv` (denormalized copy) | `airport_id` | Validate and drop duplicated columns from the schedule export |

---

## What This Data Is *Not*

To avoid any ambiguity for anyone reviewing this repository:

- This is **not** a data quality bug list to be silently patched — every inconsistency above was deliberately introduced to simulate a realistic multi-source ingestion scenario for a Power Query / data modeling exercise.
- The reconciled output of this raw layer is the star schema (`dim_customer`, `dim_aircraft`, `dim_airport`, `dim_cabin_class`, `dim_booking_channel`, `dim_date`, `fact_flights`, `fact_bookings`) documented separately and implemented in `pbix/analysis.pbix`.
- No real airline, individual, or booking is represented anywhere in this dataset.
