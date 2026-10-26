# Logistics Operations Dashboard (Power BI)

A step-by-step record of how this report was built, from raw tables to finished pages.

> **What this document covers:** the data model, the DAX measures, the visuals, the page design and how everything was checked.
> **What it does not cover (yet):** insights and recommendations. Those will be added separately.

---

## Table of contents

1. [What this project is](#1-what-this-project-is)
2. [The rules we followed](#2-the-rules-we-followed)
3. [Step 1: Build the data model](#3-step-1-build-the-data-model)
4. [Step 2: Write the DAX measures](#4-step-2-write-the-dax-measures)
5. [Step 3: Build the visuals](#5-step-3-build-the-visuals)
6. [Page design and background images](#6-page-design-and-background-images)
7. [How we checked our work](#7-how-we-checked-our-work)
8. [Problems we hit and how we fixed them](#8-problems-we-hit-and-how-we-fixed-them)
9. [Project files](#9-project-files)
10. [Rebuild checklist](#10-rebuild-checklist)
11. [Glossary](#11-glossary)

---

## 1. What this project is

A Power BI report for a trucking (logistics) company. It answers five groups of questions. Each group is one **section** of the report.

| Section | Name | The big question |
|---|---|---|
| **A** | Overall Operations | How healthy and efficient is the operation? |
| **B** | Delivery Performance | Where and why are deliveries delayed? |
| **C** | Fleet Performance | Are we getting enough value from our trucks? |
| **D** | Driver Performance & Safety | Which drivers perform well, and where are the safety risks? |
| **E** | Profitability & Cost Efficiency | Where is revenue earned efficiently, and where do costs leak? |

### Questions each section must answer

**A. Overall Operations:** How many loads and trips are completed? What is total revenue? What is the on-time delivery rate? How well are trucks and drivers used? Are the KPIs getting better or worse over time?

**B. Delivery Performance:** What percent of deliveries are on time? Which facilities have the most detention and delays? Which routes do best and worst? Which customers see the most delays? How much time is lost to detention? Are delays bunched in certain places, days or periods?

**C. Fleet Performance:** Which trucks earn the most revenue? Which are used the most? Which have too much downtime? Which use the most fuel? What does maintenance cost for each truck? Do older trucks perform worse than newer ones?

**D. Driver Performance & Safety:** Which drivers have the best and worst on-time rates? How many trips does each driver complete? How productive is each driver? Which drivers have safety incidents? What percent of incidents are preventable? Which drivers, trucks and routes carry the most safety risk?

**E. Profitability & Cost Efficiency:** Revenue by customer, route, truck and driver. Fuel cost and fuel efficiency. Maintenance cost. Cost per mile, revenue per mile, fuel cost per mile. Which customers and routes are most valuable? Which assets earn a lot but also cost a lot to run?

### Tools used

- **Power BI Desktop** (Model view, Report view and DAX query view)
- **PowerPoint** (to design the page background images)

### The data (14 tables)

| Table | What it holds |
|---|---|
| `trips` | One row per trip: driver, truck, trailer, miles, hours, fuel used, idle time, status |
| `loads` | One row per load: customer, route, revenue, fuel surcharge, extra charges |
| `delivery_events` | Pickup and delivery stops: scheduled vs actual time, on-time flag, detention minutes |
| `fuel_purchases` | Fuel buys: gallons, price, cost, truck, driver, trip |
| `maintenance_records` | Repair jobs: cost, labor, parts, downtime hours, truck |
| `safety_incidents` | Accidents and claims: damage costs, flags (preventable, at fault, injury), driver, truck, trip |
| `truck_utilization_metrics` | One row per truck per month: utilization rate, miles, downtime |
| `driver_monthly_metrics` | One row per driver per month: trips, miles, revenue, on-time rate |
| `drivers`, `trucks`, `trailers` | Lists of people and equipment |
| `customers`, `routes`, `facilities` | Lists of who we haul for, where we drive, and where we stop |

### Facts about the data we confirmed with checks

- 85,410 loads and 85,410 trips, with exactly one trip per load.
- Every trip and every load has the status `Completed`.
- Data covers 36 months, from January 2022 to December 2024.
- Each trip has one `Pickup` stop and one `Delivery` stop (85,410 of each).
- The flag columns (`on_time_flag`, `preventable_flag`, `at_fault_flag`, `injury_flag`) hold True/False values.
- `utilization_rate` is stored as a fraction (a value of 1.0 means 100%) and ranges from 0.387 to 1.484, so some rows are above 100%.
- 150 drivers exist, 125 of them have trips. There are 170 incident rows.

---

## 2. The rules we followed

1. **Three steps, in order:** (1) data model, (2) DAX measures, (3) visuals.
2. **One section at a time.** Finish A completely (measures, checks, visuals) before starting B.
3. **One clean DAX file per section.** One `DEFINE` block, one `EVALUATE` at the end. All measures are added in one go, never pasted one by one.
4. **Every measure has a comment** that explains *why* it is written that way.
5. **Check the numbers** before moving on. Each file ends with a "verification row" with known answers.
6. **The report must be interactive.** Slicers and clicks change every chart, so measures are written to react to filters.

---

## 3. Step 1: Build the data model

### 3.1 The simple idea

Tables come in two kinds:

- **Dimension tables** are lists of things: drivers, trucks, customers, routes, facilities, dates.
- **Fact tables** record events: trips, deliveries, fuel purchases, repairs, incidents.

A **relationship** joins a dimension to a fact table by an ID column. When you click one driver, the relationship lets Power BI filter every fact table to just that driver.

### 3.2 Table roles

| Role | Tables |
|---|---|
| Dimensions | `customers`, `routes`, `drivers`, `trucks`, `trailers`, `facilities`, `dim_date`, `dim_delivery_date` |
| Hub fact (the center) | `trips` |
| Parent of the hub | `loads` (holds revenue, customer and route) |
| Event facts | `delivery_events`, `fuel_purchases`, `maintenance_records`, `safety_incidents` |
| Monthly summaries | `driver_monthly_metrics`, `truck_utilization_metrics` |

### 3.3 The shape of the model

```text
customers ─┐
routes ────┴─► loads ─► trips ◄─ drivers
                         │  ▲ ◄─ trucks
                         │  └─── trailers
                         │       dim_date (dispatch date)
                         ▼
                  delivery_events ◄─ facilities
                         ▲
                  dim_delivery_date (scheduled date)

trucks / drivers / dim_date ─► fuel_purchases
trucks / dim_date ─────────► maintenance_records
trucks / drivers / routes / dim_date ─► safety_incidents
trucks / dim_date ─────────► truck_utilization_metrics
drivers / dim_date ────────► driver_monthly_metrics
```

### 3.4 Relationships

All are **one-to-many**, filter direction **single** (from the list table to the event table).

| From (one) | To (many) | Key | Active? |
|---|---|---|---|
| `customers` | `loads` | customer_id | Yes |
| `routes` | `loads` | route_id | Yes |
| `loads` | `trips` | load_id | Yes |
| `drivers` | `trips` | driver_id | Yes |
| `trucks` | `trips` | truck_id | Yes |
| `trailers` | `trips` | trailer_id | Yes |
| `dim_date` | `trips` | dispatch_date | Yes |
| `trips` | `delivery_events` | trip_id | Yes |
| `facilities` | `delivery_events` | facility_id | Yes |
| `dim_delivery_date` | `delivery_events` | scheduled_date | Yes |
| `dim_delivery_date` | `delivery_events` | actual_date | **No** |
| `trucks` | `fuel_purchases` | truck_id | Yes |
| `drivers` | `fuel_purchases` | driver_id | Yes |
| `trips` | `fuel_purchases` | trip_id | **No** |
| `dim_date` | `fuel_purchases` | purchase_date | Yes |
| `trucks` | `maintenance_records` | truck_id | Yes |
| `dim_date` | `maintenance_records` | maintenance_date | Yes |
| `trucks` | `safety_incidents` | truck_id | Yes |
| `drivers` | `safety_incidents` | driver_id | Yes |
| `trips` | `safety_incidents` | trip_id | **No** |
| `dim_date` | `safety_incidents` | incident_date | Yes |
| `routes` | `safety_incidents` | route_id (a column we add, see 3.6) | Yes |
| `trucks` | `truck_utilization_metrics` | truck_id | Yes |
| `dim_date` | `truck_utilization_metrics` | month | Yes |
| `drivers` | `driver_monthly_metrics` | driver_id | Yes |
| `dim_date` | `driver_monthly_metrics` | month | Yes |
| `dim_date` | `loads` | load_date | **No** |

Power BI shows inactive relationships as dotted lines.

### 3.5 Why it is built this way

- **`trips` is the center.** Customer and route filters reach `trips` through `loads`, so customer and route results work with normal slicers.
- **Revenue sits on `loads`, but truck and driver slicers filter `trips`.** A filter on `trips` cannot flow back up to `loads`. So we add a column on `trips` that copies the load's revenue (see 3.6).
- **Only one active date path per chain.** If `dim_date` connected to both `loads` and `trips`, Power BI would find two paths to `trips` and refuse. So `dispatch_date` is the active one and `load_date` is inactive.
- **A second date table for deliveries.** Section B needs the *delivery* date, not the dispatch date. `dim_delivery_date` is a copy of `dim_date` used only for `delivery_events`.
- **Fuel and incidents connect by truck and driver**, not only by trip, because some rows might have no trip. The trip links are kept inactive and used only inside special calculations.
- **The monthly summary tables are used lightly.** They only supply the utilization rate. Everything else is calculated from the base tables, so nothing is counted twice.
- **The date table has a fixed range.** The automatic calendar function would also read birth dates and hire dates and stretch back decades, so we set the range from the real operating dates.

### 3.6 Extra tables and columns we created


**Empty measures table:** Home > Enter data, create a table called `_Measures`, then hide its blank column. All measures live here.

---

## 4. Step 2: Write the DAX measures

### 4.1 What is a measure?

A **measure** is a calculation that gives a different answer depending on what is selected. "Total Revenue" gives the company total. Click one truck, and the same measure gives that truck's revenue. That is how the report stays interactive.

**DAX** is the formula language Power BI uses for measures.

### 4.2 How measures were added

1. Open **DAX query view** in Power BI Desktop.
2. Open a new query tab and paste one whole section file (for example `section_a_measures.dax`). Paste it alone, never mixed with older snippets.
3. Click **Update model** above the `DEFINE` line. This writes all the measures into the model at once.
4. Press **Run**. The file ends with one `EVALUATE ROW(...)` that shows the verification numbers.

Every file looks like this:


### 4.3 Settings (config measures)

Business rules live in small "config" measures at the top of each file, so one edit changes every measure that uses them.

| Config measure | Value | Used for |
|---|---|---|
| `_Cfg Completed Status` | `"Completed"` | Which trips count as finished |
| `_Cfg Delivery Event Type` | `"Delivery"` | Which stops are deliveries |
| `_Cfg Pickup Event Type` | `"Pickup"` | Which stops are pickups |
| `_Cfg On-Time Target` | `0.95` | The on-time goal (a placeholder) |
| `_Cfg Min Volume` | `30` | Smallest number of trips, deliveries or stops allowed in a ranking |
| `_Cfg Min Miles` | `10000` | Smallest mileage allowed in a safety-risk ranking |
| `_Cfg Excess Downtime Factor` | `1.25` | A truck has "excess" downtime if it is 25% above the fleet average |
| `_Cfg Flag Yes Value` | `TRUE ()` | What "yes" looks like in the flag columns |
| `_Cfg Pts At Fault`, `_Cfg Pts Preventable`, `_Cfg Pts Injury` | `1`, `1`, `3` | Extra points in the safety Risk Points score |
| `_Cfg Include Incident Cost` | `1` | `1` counts incident damage as a running cost, `0` leaves it out |

### 4.4 What each section's file contains

| Section | File | Measures | Main groups |
|---|---|---|---|
| A | `section_a_measures.dax` | 49 | Volume, revenue, on-time delivery, utilization and efficiency, month-over-month trends |
| B | `section_b_measures.dax` | 34 | Late rate, pickups, stops, detention, concentration shares, callout cards |
| C | `section_c_measures.dax` | 34 | Revenue per truck, utilization, downtime, fuel, maintenance, callout cards |
| D | `section_d_measures.dax` | 41 | Driver on-time, productivity, safety counts, incident cost, risk per million miles, callout cards |
| E | `section_e_measures.dax` | 33 | Fuel cost, maintenance (actual and estimated), damage, margin, cost per mile, truck quadrants, waterfall, callout cards |

**Total: 191 measures.**

### 4.5 Key definitions

- **Completed Trips / Revenue / Miles:** counted only for trips with status `Completed`. Revenue is the load revenue plus fuel surcharge plus accessorial charges, placed on the trip (`trip_revenue`).
- **On-Time Delivery %:** on-time delivery stops divided by all delivery stops. It is always a ratio of counts, never an average of averages.
- **Stop:** any row in `delivery_events` (a pickup or a delivery). Facilities are measured on stops, because a facility handles both.
- **Detention:** extra waiting minutes recorded at a stop. We report minutes, hours, how many stops had detention, and the average per stop.
- **Truck Utilization %:** the average of the monthly utilization rates in `truck_utilization_metrics`. Rates above 100% are kept, not capped.
- **Fleet MPG:** total miles divided by total gallons (not an average of trip MPG values).
- **Month-over-month (MoM):** this month compared with last month. It only returns a value when exactly one month is selected.
- **Rolling 3M:** the last three whole calendar months, ending in the selected month.
- **Safety risk:** incidents **per 1,000,000 miles**, so busy drivers, trucks and routes are compared fairly. **Risk Points** adds severity: every incident scores 1, plus the config points for at-fault, preventable and injury.
- **Claim amount:** in this data, the claim amount equals vehicle damage plus cargo damage. We never add claims on top of damage, to avoid counting a loss twice.
- **Contribution Margin** = Revenue − Fuel − Maintenance − Incident damage. **It is not net profit.** The data has no driver pay, insurance, depreciation or overhead.
- **Maintenance in Section E:** maintenance is recorded per truck, not per customer, route or driver. So:
  - truck, fleet and date views use the **actual** maintenance cost;
  - when a customer, route, driver or trailer filter is active, the measure switches to an **estimate** = that slice's miles × the network's maintenance cost per mile. The estimates add back to the actual total.
- **Callout cards:** text measures such as "Worst Day" or "Most Valuable Customer". They update with the slicers and ignore groups below the minimum volume.

### 4.6 Number formats

| Format | Used for |
|---|---|
| `#,0` | Counts (trips, stops, incidents) |
| `$#,0` | Money totals |
| `$0.00` | Money per mile or per gallon |
| `0.0%` | Rates and shares |
| `+0.0%;-0.0%` | Gaps and point changes |
| `#,0.0` or `0.0` | Averages, hours, rates per million miles |

---

## 5. Step 3: Build the visuals

### 5.1 Pages

There are **nine report pages**: A has one page, and each of B, C, D and E has two.

| Page | Name |
|---|---|
| A | Overall Operations |
| B1 | Delivery Performance (overview and timing) |
| B2 | Delivery Hot Spots (facilities, routes, customers) |
| C1 | Fleet Performance (revenue, utilization, downtime) |
| C2 | Fleet Costs & Age (fuel, maintenance, age) |
| D1 | Driver Performance |
| D2 | Driver Safety |
| E1 | Profitability & Cost Efficiency (overview) |
| E2 | Customers, Routes & Assets |

### 5.2 Slicers and syncing

Open **View > Sync slicers**. For each slicer, tick **Sync** on the pages where it should filter.

| Slicer | Pages |
|---|---|
| `dim_date[Year]`, `dim_date[Quarter]` | A, C, D, E (not B) |
| `dim_delivery_date[Year]`, `[Quarter]` | B only |
| `trucks[make]`, `trucks[home_terminal]` | C and E |
| `customers[customer_name]` | A and B only |
| `facilities[facility_type]` | B only |
| `drivers[employment_status]`, `drivers[home_terminal]` | D only |

Why B is separate: it works on **delivery date**, while the other pages work on **dispatch date**. If both were active, their selections would overlap and the numbers would change.

Why E has no customer, route or driver slicers: any of those filters would flip maintenance cost from actual to estimated for every chart on the page.

### 5.3 Section A: Overall Operations (1 page)

| Visual | Type | Fields |
|---|---|---|
| KPI cards (5) | Card | `[Completed Trips]`, `[Total Revenue]`, `[On-Time Delivery %]`, `[Truck Utilization %]`, `[Revenue per Mile]`. First four use the matching Trend Label and Trend Color measures and a "Top 1" filter on `dim_date[Month Start]` by `[Latest Month With Data]` |
| Revenue growth vs service quality | Line and clustered column | X `dim_date[Month Start]`; columns `[Total Revenue]`; line `[On-Time Delivery %]` |
| On-time vs target | Line | `[On-Time Delivery %]`, `[On-Time % Rolling 3M]`, `[On-Time Target Line]` |
| Volume vs yield | Line and clustered column | Columns `[Completed Trips]`; line `[Revenue per Mile]` |
| Utilization vs idle waste | Line | `[Truck Utilization %]`, `[Idle Time %]` |
| Asset productivity | Clustered column | `[Trips per Active Truck]`, `[Trips per Active Driver]` |
| KPI scorecard | Matrix | Rows `Month Start` (newest first); revenue, trips, on-time and utilization with their MoM measures |

### 5.4 Section B: Delivery Performance (2 pages)

**B1**

| Visual | Type | Fields |
|---|---|---|
| KPI cards (4) | Card | `[On-Time Delivery %]`, `[Late Deliveries]`, `[Detention Hours]`, `[Detention Rate %]` |
| Callouts (5) | Card | `[Worst Day (Late %)]`, `[Worst Facility (Late %)]`, `[Worst Customer (Late %)]`, `[Worst Route (Late %)]`, `[Best Route (Late %)]` |
| Delay trend | Line and clustered column | X `dim_delivery_date[Month Start]`; columns `[Late Deliveries]`; line `[On-Time Delivery %]` |
| Pickup vs delivery | Clustered column | `[Pickup On-Time %]`, `[On-Time Delivery %]` |
| Detention trend | Line and clustered column | Columns `[Detention Hours]`; line `[Avg Detention per Stop (min)]` |
| Weekday pattern | Column | X `dim_delivery_date[Day Name]`; Y `[On-Time Delivery %]`; constant line from `[On-Time % (All Days)]` |
| Seasonal heat map | Matrix | Rows `Day Name`; columns `Month`; values `[Late Delivery %]`; color gradient |

**B2**

| Visual | Type | Fields |
|---|---|---|
| Worst facilities | Bar | Y `facilities[facility_name]`; X `[Stop On-Time %]`; Bottom 10; filter `[Meets Min Stops]` = 1; color `[Stop vs Network Color]` |
| Detention hot spots | Bar | Y `facility_name`; X `[Detention Hours]`; Top 10 |
| Facility map | Bubble map | Latitude and longitude from `facilities`; size `[Late Stops]` |
| Worst routes | Bar | Y `routes[route_label]`; X `[On-Time Delivery %]`; Bottom 10; filter `[Meets Min Deliveries]` = 1; color `[Delivery vs Network Color]` |
| Best routes | Bar | Same fields; Top 10 |
| Customers most affected | Bar | Y `customers[customer_name]`; X `[Late Deliveries]`; Top 10 |
| Detention vs lateness | Scatter | X `[Avg Detention per Stop (min)]`; Y `[Stop On-Time %]`; size `[Total Stops]`; details `facility_name` |
| Does detention cause lateness? | Clustered column | `[Stop Late % (Detained)]`, `[Stop Late % (No Detention)]` |

### 5.5 Section C: Fleet Performance (2 pages)

**C1**

| Visual | Type | Fields |
|---|---|---|
| KPI cards (7) | Card | `[Active Trucks]`, `[Revenue per Active Truck]`, `[Truck Utilization %]`, `[Over-Utilized %]`, `[Downtime Hours]`, `[Maintenance Cost]`, `[Fleet MPG]` |
| Callouts (5) | Card | `[Top Revenue Truck]`, `[Highest Utilization Truck]`, `[Most Downtime Truck]`, `[Highest Fuel Use Truck]`, `[Highest Maintenance Cost Truck]` |
| Top revenue trucks | Bar | Y `trucks[truck_label]`; X `[Total Revenue]`; Top 10 |
| Highest utilization | Bar | X `[Truck Utilization %]`; Top 10; color `[Utilization vs Fleet Color]` |
| Lowest utilization | Bar | Same fields; Bottom 10 |
| Excessive downtime | Bar | X `[Downtime Hours]`; Top 10; color `[Downtime Color]` |
| Revenue vs downtime | Scatter | X `[Downtime Hours]`; Y `[Total Revenue]`; size `[Total Miles]`; details `truck_label` |

**C2**

| Visual | Type | Fields |
|---|---|---|
| Fuel hogs | Bar | X `[Total Fuel Gallons]`; Top 10 |
| Least efficient | Bar | X `[Fleet MPG]`; Bottom 10; color `[MPG vs Fleet Color]` |
| Maintenance cost by truck | Stacked bar | `[Labor Cost]`, `[Parts Cost]`; Top 10 by `[Maintenance Cost]` |
| Maintenance by type | Bar | Y `maintenance_records[maintenance_type]`; X `[Maintenance Cost]` |
| Old vs new trucks | Matrix | Rows `trucks[truck_age_band]`; active trucks, revenue per truck, utilization, MPG, maintenance cost per mile, downtime per 1,000 miles |
| Age curve | Line | X `trucks[truck_age_years]`; Y `[Maintenance Cost per Mile]` |
| Cost and downtime trend | Line and clustered column | Columns `[Maintenance Cost]`; line `[Downtime Hours]` |
| Truck leaderboard | Matrix | Rows `truck_label`; revenue, miles, utilization, downtime, fuel, MPG, maintenance cost |

### 5.6 Section D: Driver Performance & Safety (2 pages)

**D1: performance**

| Visual | Type | Fields |
|---|---|---|
| KPI cards (4) | Card | `[Active Drivers]`, `[Trips per Active Driver]`, `[Revenue per Active Driver]`, `[On-Time Delivery %]` |
| Callouts (7) | Card | `[Best On-Time Driver]`, `[Worst On-Time Driver]`, `[Most Productive Driver]`, `[Most Incidents Driver]`, `[Highest Risk Driver]`, `[Highest Risk Truck]`, `[Highest Risk Route]` |
| Best on-time drivers | Bar | Y `drivers[driver_label]`; X `[On-Time Delivery %]`; Top 10; filter `[Meets Min Deliveries]` = 1 |
| Worst on-time drivers | Bar | Same fields; Bottom 10; color `[Driver vs All Color]` |
| Productivity vs punctuality | Scatter | X `[Trips per Active Month]`; Y `[On-Time Delivery %]`; size `[Total Revenue]` |
| Driver leaderboard | Matrix | Rows `driver_label`; trips, trips per active month, miles, revenue, on-time %, idle %, MPG, incidents |

**D2: safety**

| Visual | Type | Fields |
|---|---|---|
| KPI cards (4) | Card | `[Total Incidents]`, `[Preventable %]`, `[Incidents per 1M Miles]`, `[Incident-Free Driver %]` |
| Incident trend | Line and clustered column | Columns `[Total Incidents]`; line `[Incidents per 1M Miles]` |
| Incident types | Bar | Y `safety_incidents[incident_type]`; X `[Total Incidents]` |
| Preventable share | Donut | Legend `safety_incidents[preventable_flag]`; values `[Total Incidents]` |
| Highest-risk drivers | Bar | X `[Incidents per 1M Miles]`; Top 5; filter `[Meets Min Miles]` = 1; color `[Risk vs Network Color]` |
| Highest-risk trucks | Bar | Same, by `trucks[truck_label]` |
| Highest-risk routes | Bar | Same, by `routes[route_label]` |
| Does experience matter? | Matrix | Rows `drivers[experience_band]`; on-time %, trips per month, MPG, incidents per 1M miles, preventable % |

### 5.7 Section E: Profitability & Cost Efficiency (2 pages)

**E1: overview**

| Visual | Type | Fields |
|---|---|---|
| KPI cards (8) | Card | `[Total Revenue]`, `[Operating Cost]`, `[Contribution Margin]`, `[Contribution Margin %]`, `[Revenue per Mile]`, `[Cost per Mile]`, `[Fuel Cost per Mile]`, `[Fleet MPG]` |
| Callouts (7) | Card | `[Most Valuable Customer]`, `[Lowest-Margin Customer]`, `[Most Valuable Route]`, `[Lowest-Margin Route]`, `[Most Valuable Driver]`, `[Most Profitable Truck]`, `[Costliest Truck per Mile]` |
| Where each revenue dollar goes | Waterfall | Category `'Margin Bridge'[Step]`; Y `[Margin Bridge Value]` |
| Revenue vs cost trend | Line and clustered column | Columns `[Total Revenue]`; lines `[Operating Cost]`, `[Contribution Margin]` |
| Unit economics | Line | `[Revenue per Mile]`, `[Cost per Mile]`, `[Fuel Cost per Mile]` |
| Cost mix | Stacked column | `[Fuel Cost]`, `[Maintenance Cost (Used)]`, `[Trip Damage Cost]` |
| Fuel price vs efficiency | Line | `[Fuel Price per Gallon]`; second axis `[Fleet MPG]` |

**E2: customers, routes, assets**

| Visual | Type | Fields |
|---|---|---|
| Most valuable customers | Bar | Y `customers[customer_name]`; X `[Contribution Margin]`; Top 10; color `[Margin vs Network Color]` |
| Customer size vs margin | Scatter | X `[Total Revenue]`; Y `[Contribution Margin %]`; size `[Total Miles]`; filter `[Meets Min Trips]` = 1 |
| Lowest-margin routes | Bar | Y `routes[route_label]`; X `[Contribution Margin %]`; Bottom 10; filter `[Meets Min Trips]` = 1 |
| Asset quadrant | Scatter | X `[Total Revenue]`; Y `[Cost per Mile]`; size `[Total Miles]`; details `truck_label`; data color `[Asset Quadrant Color]` |
| Truck leaderboard | Matrix | Rows `truck_label`; revenue, cost, margin, margin %, cost per mile, fuel cost per mile, maintenance cost per mile, quadrant |

The asset quadrant sorts each truck into four boxes: revenue at or above the fleet average or below it, and cost per mile above the fleet's or not.

---

## 6. Page design and background images

### 6.1 Page size and grid

- **Page size:** custom **1920 × 1080 px** (16:9) on every page. Set it in Format page > Canvas settings > Custom.
- **Why not the default 1280 × 720:** 1920 × 1080 fills a normal monitor at 100%, so text stays readable. Keeping every page the same size also lets one design be reused.
- **Grid:** 32 px side margins, 16 px gaps, 12 columns of 140 px. A span of 3 columns is 452 px, 4 is 608, 5 is 764, 6 is 920, 7 is 1,076 and 12 is 1,856.
- **Fixed header on every page:** title on the left, page-navigator panel at X 812 and Y 24 (1,076 × 72), four slicer panels at Y 112 (452 × 56 each), content starting at Y 184.

Section A as an example:

| Element | X | Y | Width × Height |
|---|---|---|---|
| Title | 32 | 24 | 764 × 72 |
| Page navigator | 812 | 24 | 1,076 × 72 |
| 4 slicers | 32, 500, 968, 1,436 | 112 | 452 × 56 each |
| 5 KPI cards | 32, 406, 780, 1,154, 1,528 | 184 | 358 × 120 (last one 360 wide) |
| Revenue growth vs service quality | 32 | 320 | 1,076 × 250 |
| On-time vs target | 1,124 | 320 | 764 × 250 |
| Volume vs yield, Utilization vs idle, Asset productivity | 32, 656, 1,280 | 586 | 608 × 250 each |
| KPI scorecard | 32 | 852 | 1,856 × 204 |

The exact X, Y, width and height of **every** panel and visual on all nine pages are in `visual_placement.csv`.

### 6.2 Colors and fonts

| Use | Color |
|---|---|
| Page background | `#EEF1F4` |
| Panels | White, 1 px outline `#E3E8ED`, 8 px corners |
| Main text | `#1B2A3A` |
| Muted text | `#4B5866` |
| Accent colors | Teal `#0F4C5C` and `#0F766E`, amber `#B45309`, red `#B3261E`, green `#2E7D32` |
| Font | Segoe UI |

### 6.3 Making the background image in PowerPoint

1. Set the slide size to **20 in × 11.25 in** (Design > Slide Size > Custom). PowerPoint exports at 96 pixels per inch, so this gives exactly 1920 × 1080.
2. Divide pixels by 96 to get inches. Example: 1,076 × 250 px at (32, 320) is 11.21 × 2.60 in at (0.33, 3.33).
3. Draw only the parts that never change: the background color, the white rounded panels, and each panel's static title and subtitle.
4. Do **not** draw data, charts or tabs.
5. Export: File > Export > Change File Type > PNG > **Just This One**. Check the size is 1920 × 1080.

### 6.4 Loading it into Power BI

1. Format page > Canvas settings: Custom, 1920 × 1080.
2. Format page > Canvas background > Image > Add image. Set Image fit to **Fit** and Transparency to **0%**.
3. Set **Wallpaper** to `#EEF1F4`.
4. For each visual, turn **Effects > Background** off so the panel shows through.
5. Type each visual's X, Y, Width and Height from `visual_placement.csv` (Format visual > General > Properties).
6. Use Insert > Buttons > Navigator > Page navigator for the A to E tabs.

### 6.5 Where visuals sit inside a panel

- **Tiles (charts and tables):** the visual sits inside the panel with an 8 px inset on the sides and bottom and 56 px at the top, which leaves room for the static title and subtitle. Visual size = panel width − 16, panel height − 64.
- **KPI cards, callouts, slicers and the navigator:** the visual fills the whole panel (same X, Y, width and height).
- Turn **off** each visual's own title, so the background title is not shown twice.

---

## 7. How we checked our work

Every section file ends with one `EVALUATE ROW(...)` that returns checking numbers. Two kinds of checks were used:

- **Known totals.** Numbers that must match the data exactly (for example, completed trips must equal the number of trips in the table).
- **"Summed by" checks.** The measure is added up across every driver, truck, customer or route. The sum must equal the plain total. If it does not, a relationship is missing or wrong.

| Section | Main checks |
|---|---|
| A | Completed trips, loads and delivery stops each equal 85,410; revenue measure equals the sum of load revenue (plus surcharges after the revenue update) |
| B | Deliveries 85,410; pickups 85,410; stops 170,820; on-time and late stops add up to total stops; deliveries and stops summed by month, customer, route and facility equal the totals |
| C | Truck-months 3,312; revenue, fuel, downtime and maintenance summed by truck equal the totals; maintenance cost equals labor plus parts |
| D | Trips summed by driver 85,410; 170 incidents summed by driver, truck and route; Risk Points equals 170 + at-fault + preventable + 3 × injury |
| E | Trip damage cost equals the damage total from Section D; allocated maintenance summed by route equals actual maintenance; Contribution Margin summed by customer, route, driver and truck equals the total |

The monthly summary tables (`driver_monthly_metrics`, `truck_utilization_metrics`) were compared with the base tables for information only. They were not used as the source for totals.

---

## 8. Problems we hit and how we fixed them

| Problem | Cause | Fix |
|---|---|---|
| Red underlines after pasting DAX | A patch was pasted into the middle of the original file, leaving a stray `EVALUATE` and duplicate measure names | One clean file per section, with one `DEFINE` and one `EVALUATE`. Paste the whole file into a new tab and do not edit it |
| "Name is misspelt" error | A table or column name in the query did not match the model exactly (most often the measures table name `_Measures`) | Run `EVALUATE INFO.VIEW.TABLES()` to see the real table names, then fix the name |
| Cleared the validation query by mistake | Nothing to recover | The validation queries only read data. The measures live in the model, not in the query tab |
| Rolling 3-month measures counted extra days | The built-in period function counts back from the last day of the month, which pulls in extra days in short months | Use a window from the first day three months back to the end of the selected month |
| Utilization showed the wrong value for averages above 1 | An early rule divided by 100 whenever the average was above 1 | The data is stored as a fraction, so the measure is a plain average with no scaling |
| Circular dependency error on `trip_damage_cost` | Two columns both used `CALCULATE` on `trips`, so each depended on the other | Write `trip_damage_cost` with `SUMX` and `FILTER` instead of `CALCULATE` |
| Same maintenance cost on every route | Maintenance is not linked to routes, customers or drivers, so those filters were ignored | Switch to the estimated maintenance when one of those filters is active (see 4.5) |
| Page title centered in PowerPoint | The title placeholder defaulted to center alignment | Set the placeholder alignment to left |

---

## 9. Project files

Suggested repository layout:

```text
.
├── README.md                          (this file)
├── dax/
│   ├── section_a_measures.dax
│   ├── section_b_measures.dax
│   ├── section_c_measures.dax
│   ├── section_d_measures.dax
│   └── section_e_measures.dax
├── backgrounds/
│   ├── A_Overall_Operations.png
│   ├── B1_Delivery_Overview.png
│   ├── B2_Delivery_Hot_Spots.png
│   ├── C1_Fleet_Revenue_Utilization.png
│   ├── C2_Fleet_Fuel_Maintenance_Age.png
│   ├── D1_Driver_Performance.png
│   ├── D2_Driver_Safety.png
│   ├── E1_Profitability_Overview.png
│   ├── E2_Customers_Routes_Assets.png
│   ├── Dashboard_Backgrounds_A_D_E.pptx
│   ├── Dashboard_Backgrounds_B_C.pptx
│   └── visual_placement.csv
└── report/
    └── logistics_dashboard.pbix
```

---

## 10. Rebuild checklist

**Model**
- [ ] Load the 14 tables
- [ ] Create `dim_date` and `dim_delivery_date`, mark both as date tables, set the sort-by columns
- [ ] Create the relationships in section 3.4 and set the four inactive ones
- [ ] Create the calculated columns and the `Margin Bridge` table in section 3.6
- [ ] Create the empty `_Measures` table and hide its blank column

**Measures** (repeat for A, then B, C, D, E, finishing each before the next)
- [ ] New DAX query tab, paste the whole section file, click Update model, run it
- [ ] Compare the verification row with the known values
- [ ] Set number formats

**Visuals**
- [ ] Set every page to Custom 1920 × 1080
- [ ] Add the page's background PNG, set Wallpaper to `#EEF1F4`
- [ ] Place each visual using `visual_placement.csv`
- [ ] Turn off visual backgrounds and visual titles
- [ ] Set up the slicers and sync them as in section 5.2
- [ ] Add the page navigator

---

## 11. Glossary

| Word | Simple meaning |
|---|---|
| **DAX** | The formula language of Power BI |
| **Measure** | A calculation that changes when you click or filter |
| **Calculated column** | A new column added to a table, worked out row by row |
| **Calculated table** | A whole new table made from a formula |
| **Relationship** | A link between two tables through a shared ID column |
| **Dimension table** | A list of things (drivers, trucks, dates) |
| **Fact table** | A table of events (trips, fuel buys, incidents) |
| **Slicer** | A filter box on the report page |
| **KPI** | A key number that shows how things are going |
| **On time** | A stop marked on time in `on_time_flag` |
| **Detention** | Extra waiting time at a stop, recorded in minutes |
| **Utilization** | How much of a truck's available capacity was used |
| **MPG** | Miles per gallon of fuel |
| **MoM** | Month over month: this month compared with last month |
| **Rolling 3M** | The last three whole months, ending now |
| **Per 1M miles** | A rate worked out per one million miles driven |
| **Contribution margin** | Revenue minus fuel, maintenance and incident damage. Not net profit |
| **DEFINE / EVALUATE** | The two parts of a DAX query: DEFINE creates measures, EVALUATE shows a result |



# Logistic-Operations-and-Report-Analysis


- [Question](#Questions)
- [Processing](#Processing)
- [Methodology](#Methodology)
- [Visualization](#Visualization)
- [Insights](#Insights)
- [Recommendations](#Recommendations)
- [Tools](#Tools)


## Question


## Processing


## Methodology
![v1](asset/data_modeling.png)

## Visualization

**Overall Operations**
![v1](asset/01.png)


**Delivery Performance**
![v1](asset/02.png)


**Fleet Performance**
![v1](asset/03.png)



**Driver Performance**
![v1](asset/04.png)


**Driver Safety**
![v1](asset/05.png)



**Profitability & Cost Efficiency**
![v1](asset/06.png)

## Insights


## Recommendations


## Tools


| Tools | Purpose |
| --- | --- |
| Power BI | Data Modeling, DAX measures, Visualization |





