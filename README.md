<div align="center">

# ✈️ Global Reach Airlines — Performance & Operational Analytics

### An end-to-end Power BI project: from messy multi-source raw data to a fully modeled, interactive analytics dashboard

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Advanced-1B4965?style=for-the-badge)
![Power Query](https://img.shields.io/badge/Power%20Query-ETL-5FA8D3?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-Synthetic-8A94A6?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

<br>

![Home Page](assets/screenshots/home.png)

</div>

---

## 📌 About This Project

**Global Reach Airlines** is a fictional mid-sized airline used as the subject of this project. The dataset is **entirely synthetic** — generated to simulate four years (2022–2025) of operations across a 27-destination network, a fleet of 34 aircraft, and roughly 300,000 customers. No real airline, individual, or transaction is represented anywhere in this repository.

The goal of this project was not just to build a dashboard, but to demonstrate a **realistic, end-to-end BI workflow**: raw data deliberately fragmented across multiple simulated source systems (CRM, loyalty platform, scheduling system, ops monitoring, online and offline booking channels), reconciled and reshaped in Power Query, modeled into a proper analytical schema, and analyzed through advanced DAX and interactive visuals — the same kind of workflow a BI analyst would face with real operational data.

> 📄 The raw source data and every data-quality issue it contains are documented in detail in [`data/raw/README.md`](data/raw/README.md).

---

## 📑 Table of Contents

- [Key Features](#-key-features)
- [Data Model](#-data-model)
- [Dashboard Walkthrough](#-dashboard-walkthrough)
- [Advanced Techniques](#-advanced-techniques)
- [Tech Stack & Skills Demonstrated](#-tech-stack--skills-demonstrated)
- [Repository Structure](#-repository-structure)
- [Key Business Questions Answered](#-key-business-questions-answered)
- [Limitations & Assumptions](#-limitations--assumptions)
- [How to Explore This Project](#-how-to-explore-this-project)

---

## ✨ Key Features

- 🗂️ **Multi-source data reconciliation** — 6 simulated source systems merged and appended into a clean star schema using Power Query
- 🌌 **Galaxy schema modeling** — two fact tables (`fact_flights`, `fact_bookings`) sharing conformed dimensions, with dual active/inactive relationships correctly resolved in DAX
- 🧮 **Advanced DAX layer** — `USERELATIONSHIP`, `LOOKUPVALUE`, Field Parameters, dynamic Top-N parameters, and weighted vs. simple average patterns
- 📊 **4 interactive report pages** — Home, Overview, Flights Performance, and Bookings & Customers Performance
- 🎨 **Accessible, branded theme** — consistent visual identity with a custom Power BI theme
- 🧭 **Fully custom navigation** — bookmark and button-driven sidebar navigation, independent of the default page tabs

---

## 🧩 Data Model

The model follows a **Galaxy (Fact Constellation) Schema**: two fact tables — `fact_flights` and `fact_bookings` — share conformed dimensions rather than referencing each other directly, keeping each fact table in a clean star shape of its own.

<div align="center">

![Data Model](docs/data_model.png)

</div>

**Modeling highlights:**
- `dim_date` and `dim_airport` each relate to `fact_flights` **twice** (flight date vs. booking date; origin vs. destination airport) — one relationship active, one inactive per Power BI's single-active-path rule, resolved explicitly with `USERELATIONSHIP()` in DAX where needed.
- All dimensions are fully denormalized (no snowflaking) — e.g., `dim_customer` includes loyalty tier name and join date directly, with no separate lookup table required at query time.
- A `route_airport` pattern (via `LOOKUPVALUE`, independent of the model's active relationship state) resolves "the other end of the route" regardless of whether a flight is outbound or inbound from the hub — enabling accurate route-level analysis without doubling the fact table's grain.

---

## 📊 Dashboard Walkthrough

### 🏠 Home
Landing page with brand identity and button-driven navigation to every report page.

![Home Page](assets/screenshots/home.png)

### 📈 Overview — Performance & KPIs
High-level KPIs (Total Revenue, Fare vs. Ancillary Revenue, Total Flights, Total Customers) with YoY comparisons, monthly revenue trend, loyalty tier revenue contribution, and booking channel mix.

![Overview Page](assets/screenshots/overview.png)

### ✈️ Flights — Performance Details
Operational deep-dive: fleet-level revenue and flight volume, cabin class revenue contribution, regional performance, and a fare-vs-duration correlation analysis, plus a custom **Average Route Revenue Gap** KPI comparing outbound and inbound profitability per destination.

![Flights Page](assets/screenshots/flights.png)

### 🧑‍🤝‍🧑 Bookings & Customers — Performance Details
Customer-centric view: loyalty tier segmentation, booking behavior by tier, nationality mix, and a **Customer Value Concentration** scatter plot showing which loyalty tiers contribute disproportionately more revenue than their share of the customer base.

![Bookings Page](assets/screenshots/bookings.png)

---

## 🚀 Advanced Techniques

A few techniques in this project go beyond a typical portfolio dashboard and are worth calling out directly:

| Technique | Where it's used | Why it matters |
|---|---|---|
| **`USERELATIONSHIP()` for dual-role dimensions** | `Route Revenue Gap` measure | Correctly separates a single airport's revenue when it's the origin vs. when it's the destination, using the model's inactive relationship on demand |
| **`LOOKUPVALUE()` independent of model relationships** | `route_airport` calculated column | Avoids a subtle bug where `RELATED()` silently follows only the active relationship, producing correct-looking but wrong results |
| **Field Parameters** | `Revenue Source` table, Overview page toggle | Lets a single visual switch between Fare and Ancillary revenue without duplicating visuals or relying on bookmarks |
| **Disconnected parameter table (`GENERATESERIES`)** | `TopN Selection`, Top Performing Destinations table | Implements a dynamic Top-N selector (5/10/15) — a classic What-If Parameter pattern |
| **Staging query pattern in Power Query** | `Source_Data` folder (load disabled) | Keeps raw, un-modeled extracts out of the in-memory model entirely, reducing file size and separating transformation logic from the final schema |
| **Weighted vs. simple average, explicitly separated** | `Average Fare per Seat` naming | Two legitimate but different answers to "what's the average fare per seat?" — the measure name documents which one is which, avoiding silent ambiguity |

---

## 🛠️ Tech Stack & Skills Demonstrated

| Area | Tools / Techniques |
|---|---|
| **Data Cleaning & ETL** | Power Query — merge, append, locale-aware date parsing, currency string cleanup, categorical value standardization, staging query pattern |
| **Data Modeling** | Star & Galaxy schema design, conformed dimensions, active/inactive relationships, degenerate dimensions |
| **DAX** | `CALCULATE`, `USERELATIONSHIP`, `LOOKUPVALUE`, `AVERAGEX`, `FILTER`, Field Parameters, disconnected parameter tables, time intelligence |
| **Visualization** | Custom theme design, bookmark-driven navigation, KPI cards with sparklines, scatter plots, treemaps, dynamic Top-N tables |
| **Documentation** | Data dictionaries, ERD diagrams, source-system-level data quality documentation |

---

## 🗂️ Repository Structure

```
global-reach-airlines-analytics/
│
├── README.md                        ← you are here
├── LICENSE
│
├── data/
│   └── raw/
│       ├── README.md                ← raw data dictionary & source-system notes
│       ├── customer_profiles.csv
│       ├── loyalty_members.csv
│       ├── flights_schedule.csv
│       ├── flights_ops_performance.csv
│       ├── bookings_online.csv
│       ├── bookings_offline.csv
│       └── dim_*.csv                ← reference/lookup tables
│
├── pbix/
│   └── Global_Reach_Analytics.pbix
│
├── docs/
│   ├── data_model.png
│   └── measures_documentation.md
│
└── assets/
    └── screenshots/
        ├── home.png
        ├── overview.png
        ├── flights.png
        └── bookings.png
```

---

## ❓ Key Business Questions Answered

- Which loyalty tiers generate revenue disproportionate to their share of the customer base?
- Is there a systematic revenue imbalance between outbound and inbound legs on the same route?
- Which aircraft models and cabin classes contribute the most to total revenue and flight volume?
- How does booking channel mix (Website, App, Travel Agent, Call Center) vary, and what's the online vs. offline split?
- What is the relationship between flight duration and average fare across destinations?

---

## ⚠️ Limitations & Assumptions

- **Revenue, not profit** — the dataset contains ticket and ancillary revenue only; no cost data (fuel, crew, maintenance) exists, so all financial KPIs are explicitly framed as *revenue*, not *profit*.
- **Synthetic data** — all figures, names, and patterns are artificially generated and do not reflect any real airline's performance.
- **Operational vs. commercial system divergence is intentional** — seat counts from the Ops system and the Booking system are not expected to reconcile exactly; this mirrors how real airlines run these systems independently, and the gap itself is treated as an analytical signal rather than an error to hide.

---

## 🔍 How to Explore This Project

1. Clone the repository and open `pbix/Global_Reach_Analytics.pbix` in Power BI Desktop.
2. Review [`data/raw/README.md`](data/raw/README.md) to understand the source-system structure before diving into the Power Query steps.
3. Check `docs/measures_documentation.md` for a full list of DAX measures with explanations.
4. Explore the report starting from the **Home** page navigation.

---

<div align="center">

**Built as a portfolio project to demonstrate end-to-end Power BI capability — from raw, multi-source data to a polished analytical product.**

</div>
