# 📐 DAX Measures Documentation

This document catalogs the DAX layer of the Global Reach Airlines project: the core business measures, the two parameter techniques used to drive interactivity, and the conventions followed throughout the model.

> **Note:** formulas marked *(representative)* reconstruct the documented business logic but should be checked against the exact expression in the `.pbix` file before publishing, in case wording differs slightly from the final implementation.

---

## 🗂️ How Measures Are Organized

The model separates DAX into **two distinct tables**, each with a single responsibility:

| Table | Purpose | Contains |
|---|---|---|
| **`_Measures`** | Core business logic — the actual numbers being analyzed | Revenue, flight, customer, and booking calculations |
| **`_Style Measures`** | Presentation logic only — never analytical content | Conditional formatting helpers (`CF ...`), YoY deltas, prior-year comparisons, dynamic color/marker logic, per-visual formatting helpers |

`_Style Measures` is further organized into **per-page folders** (`Overview_page`, `Flights_details_page`, `Customers_details_page`), each broken down into a sub-folder per visual (e.g. `Ancillary_revenue_card`, `Donut_chart`). This keeps formatting logic traceable to the exact visual it serves, without cluttering the core `_Measures` table with formatting-only calculations.

Both tables are created as empty calculated tables (not Power Query sources), so they carry no storage cost and always sort to the top of the Fields pane thanks to the leading underscore.

---

## 💰 Revenue & Financial Measures

| Measure | DAX | Description |
|---|---|---|
| **Fare Amount Revenue** | `SUM(fact_bookings[fare_amount])` | Total ticket revenue across all bookings |
| **Ancillary Revenue** | `SUM(fact_bookings[ancillary_revenue])` | Total non-ticket revenue (baggage, meals, seat selection, etc.) |
| **Total Revenue** | `[Fare Amount Revenue] + [Ancillary Revenue]` | Combined revenue — the primary financial KPI |
| **Average Fare per Seat (Booking-Level Avg)** | `AVERAGEX(fact_bookings, DIVIDE(fact_bookings[fare_amount], fact_bookings[seats_booked]))` | Averages the per-seat fare *within each booking first*, then averages across bookings — every transaction carries equal weight regardless of party size. Intentionally distinct from a seat-weighted average; the name documents which one this is |
| **Route Revenue Gap** | ```VAR RevenueAsDestination = CALCULATE([Total Revenue]) VAR RevenueAsOrigin = CALCULATE([Total Revenue], USERELATIONSHIP(fact_flights[origin_airport_id], dim_airport[airport_id])) RETURN RevenueAsDestination - RevenueAsOrigin``` | Base measure: for a single airport in context, compares revenue earned when that airport is the **destination** (active relationship) vs. the **origin** (inactive relationship, activated on demand). Positive = inbound-heavy, negative = outbound-heavy |
| **Avg Route Revenue Gap** | `AVERAGEX(FILTER(dim_airport, dim_airport[is_hub] = FALSE()), [Route Revenue Gap])` | Network-wide average of `[Route Revenue Gap]` across all non-hub airports — most meaningful when filtered to a single city, where it collapses to that city's individual gap |

---

## ✈️ Flights & Operations Measures

| Measure | DAX | Description |
|---|---|---|
| **Total Flights** | `COUNTROWS(fact_flights)` | All scheduled flights, including cancelled ones |
| **Total Flying Flights** | `CALCULATE(COUNTROWS(fact_flights), fact_flights[flight_status] <> "CANCELLED")` | Flights that actually operated (excludes cancellations) |
| **Total Flight Movements by Airport** | ```VAR OriginCount = CALCULATE(COUNTROWS(fact_flights), USERELATIONSHIP(fact_flights[origin_airport_id], dim_airport[airport_id]), fact_flights[flight_status] <> "CANCELLED") VAR DestinationCount = CALCULATE(COUNTROWS(fact_flights), fact_flights[flight_status] <> "CANCELLED") RETURN OriginCount + DestinationCount``` | Counts a flight once for departure and once for arrival at a given airport. **Only meaningful when filtered to a single airport** — used as departures + arrivals, not as a network-wide total (which would double-count) |
| **Cancel Flights** | `CALCULATE(COUNTROWS(fact_flights), fact_flights[flight_status] = "CANCELLED")` | Number of cancelled flights |
| **Flights Cancel Ratio** | `DIVIDE([Cancel Flights], COUNTROWS(fact_flights))` | Cancellation rate as a percentage of all scheduled flights |
| **Delay Avg Minutes** | `CALCULATE(AVERAGE(fact_flights[delay_minutes]), fact_flights[flight_status] = "DELAYED")` | Average delay, computed only across flights actually marked as delayed |
| **Avg Flight Time per Trip** | `CALCULATE(AVERAGEX(fact_flights, fact_flights[scheduled_duration_minutes] + fact_flights[delay_minutes]), fact_flights[flight_status] <> "CANCELLED")` | Average realized flight duration (scheduled + delay), excluding cancelled flights |
| **AVG Load Factor PCT** | `CALCULATE(AVERAGE(fact_flights[load_factor_pct]), fact_flights[flight_status] <> "CANCELLED")` | Average aircraft load factor, excluding cancelled flights |

---

## 🧑‍🤝‍🧑 Customers & Loyalty Measures

| Measure | DAX | Description |
|---|---|---|
| **Total Customers** | `DISTINCTCOUNT(fact_bookings[customer_id])` | Unique customers who have made at least one booking |
| **VIP Customers** *(representative)* | `CALCULATE([Total Customers], dim_customer[loyalty_tier_name] IN {"Silver", "Gold", "Platinum"})` | Customers enrolled in any paid loyalty tier (excludes "None") |
| **Male Customers PCT** / **Female Customers PCT** *(representative)* | `DIVIDE(CALCULATE([Total Customers], dim_customer[gender] = "Male"), [Total Customers])` | Gender split as a percentage of the customer base |
| **Loyalty Tier Customers % of Total** | `DIVIDE([Total Customers], CALCULATE([Total Customers], ALLSELECTED(dim_customer[loyalty_tier_name])))` | Each tier's share of the total customer base — `ALLSELECTED` keeps the ratio correct under any page-level filter (e.g. Year) |
| **Loyalty Tier Revenue % of Total** | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], ALLSELECTED(dim_customer[loyalty_tier_name])))` | Each tier's share of total revenue, using the same `ALLSELECTED` logic |
| **Loyalty Tier Value Index** | `DIVIDE([Loyalty Tier Revenue % of Total], [Loyalty Tier Customers % of Total])` | The core insight measure: how many times over-represented (or under-represented) a tier's revenue contribution is relative to its customer share. `> 1` = punches above its weight |

---

## 🎫 Bookings & Channels Measures

| Measure | DAX | Description |
|---|---|---|
| **Total Bookings** *(representative)* | `COUNTROWS(fact_bookings)` | Total number of booking transactions |
| **Avg seats for each Booking** *(representative)* | `AVERAGEX(fact_bookings, fact_bookings[seats_booked])` | Average party size per booking |
| **Number of Bookings per Customer** *(representative)* | `DIVIDE(COUNTROWS(fact_bookings), [Total Customers])` | Booking frequency — average transactions per unique customer |
| **Avg Ancillary Revenue for each customer** *(representative)* | `DIVIDE([Ancillary Revenue], [Total Customers])` | Ancillary spend normalized per customer |
| **Online Booking Percentage** *(representative)* | `DIVIDE(CALCULATE(COUNTROWS(fact_bookings), dim_booking_channel[channel_name] IN {"Website", "Mobile App"}), COUNTROWS(fact_bookings))` | Share of bookings made through digital channels vs. Travel Agent / Call Center |

---

## ⚙️ Parameters

Two different parameter techniques are used in this project, deliberately chosen for two different jobs: one **swaps which measure** a visual displays, the other **feeds a numeric input** into a measure's internal logic. They are not interchangeable, and using the wrong one for the wrong job is a common modeling mistake — the distinction below is why each was chosen.

### 1. Field Parameter — `Revenue Source`

**Purpose:** lets a single chart on the Overview page toggle between showing **Fare Amount Revenue** and **Ancillary Revenue** over time, without duplicating the visual or relying on show/hide bookmarks.

**Definition (DAX calculated table):**
```dax
Revenue Source = {
    ("Ancillary", NAMEOF('_Measures'[Ancillary Revenue]), 0),
    ("Fare Amount", NAMEOF('_Measures'[Fare Amount Revenue]), 1)
}
```

This generates a small table with three columns:

| Column | Role |
|---|---|
| `Revenue Source` | The display label shown to the user (bound to the toggle buttons) |
| `Revenue Fields` | Hidden — holds a *reference* to the actual measure (`NAMEOF`), not its value |
| `Revenue Order` | Hidden — controls the default display order of the two options |

**How it's wired to the visual:** `Revenue Fields` is placed in the chart's *Values* well instead of a regular measure. When the user selects "Fare Amount" or "Ancillary" (via the toggle buttons, which are just a slicer on `Revenue Source` in disguise), Power BI swaps the entire measure driving the visual — axis, tooltips, and formatting included — with no DAX branching logic (`IF`/`SWITCH`) and no duplicated visuals required.

**Why this approach over Bookmarks:** an earlier version of this toggle could have been built with two visuals and a bookmark to show/hide each one. Field Parameters collapse that into **one visual, one field, zero duplicated objects** — easier to maintain and immune to the two visuals silently drifting out of sync (e.g. one getting a formatting fix the other doesn't).

---

### 2. Disconnected What-If Parameter — `TopN Selection`

**Purpose:** powers the dynamic **5 / 10 / 15** row selector on the "Top Performing Destinations" table.

**Definition (DAX calculated table):**
```dax
TopN Selection = GENERATESERIES(5, 15, 5)
```
`GENERATESERIES(5, 15, 5)` produces a single-column table with the values `{5, 10, 15}` — the three selectable options.

**Selected value measure:**
```dax
TopN Selection Value = SELECTEDVALUE('TopN Selection'[TopN Selection], 10)
```
`SELECTEDVALUE` returns the currently selected number, or falls back to `10` as a sensible default when nothing (or more than one value) is selected.

**Used inside the ranking logic:**
```dax
Top N Revenue = 
VAR SelectedN = [TopN Selection Value]
VAR Ranked =
    RANKX(ALL(dim_airport[city]), [Total Revenue], , DESC)
RETURN
    IF(Ranked <= SelectedN, [Total Revenue])
```
*(representative — confirm the exact ranking column/logic against the model)*

**Why "disconnected":** this table has **no relationship** to any other table in the model. It exists purely to be sliced by, feeding a number into `SELECTEDVALUE()` inside a measure — it never filters `fact_flights` or `fact_bookings` directly through the relationship graph. This is the standard **What-If Parameter** pattern.

**Field Parameter vs. Disconnected Parameter — the distinction that matters:**

| | Field Parameter (`Revenue Source`) | Disconnected Parameter (`TopN Selection`) |
|---|---|---|
| **Swaps** | *Which measure* is displayed | *A number* consumed inside a measure's logic |
| **Underlying mechanism** | `NAMEOF()` reference to a field | Plain numeric value via `SELECTEDVALUE()` |
| **Typical use case** | Metric switchers, axis switchers | Top-N selectors, sensitivity/scenario sliders |

---

## 🏷️ Naming Conventions

- **Title Case** for all analytical measure names (`Total Revenue`, not `total revenue`).
- **`CF` prefix** reserved exclusively for conditional-formatting helper measures (e.g. `CF YoY_Ancillary`) — never mixed with a measure meant to be displayed directly.
- **`Last Year_...`** and **`YoY_...`** prefixes/suffixes used consistently across every KPI card that shows a year-over-year comparison, so the pattern is predictable across the model rather than reinvented per page.
- Measures explicitly named to disambiguate methodology where more than one valid calculation exists (e.g. *Booking-Level Avg* in `Average Fare per Seat`), rather than leaving the choice implicit in the code alone.
