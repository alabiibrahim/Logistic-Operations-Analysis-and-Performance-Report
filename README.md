# Logistic-Operations-and-Report-Analysis

A step-by-step record of how this report was built, from raw tables to finished pages.

> **What this document covers:** the data model, the DAX measures, the visuals, the page design and how everything was checked.
> **What it does not cover (yet):** insights and recommendations. Those will be added separately.

---

## Table of contents
- [Question](#Questions)
- [Processing](#Processing)
- [Data model](#Data-model)
- [Methodology](#Methodology)
- [Visualization](#Visualization)
- [Insights](#Insights)
- [Recommendations](#Recommendations)
- [Tools](#Tools)


1. [What this project is](#1-what-this-project-is)
 
4. [Step 2: Write the DAX measures](#4-step-2-write-the-dax-measures)
5. [Step 3: Build the visuals](#5-step-3-build-the-visuals)
7. [How we checked our work](#7-how-we-checked-our-work)
8. [Problems we hit and how we fixed them](#8-problems-we-hit-and-how-we-fixed-them)
9. [Project files](#9-project-files)
10. [Rebuild checklist](#10-rebuild-checklist)

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

### Questions 

1.  Overall Operations
    - How many loads and trips are completed?
    - What is total revenue?
    - What is the on-time delivery rate?
    - How well are trucks and drivers used?
    - Are the KPIs getting better or worse over time?

2.  Delivery Performance
    - What percent of deliveries are on time?
    - Which facilities have the most detention and delays?
    - Which routes do best and worst? Which customers see the most delays?
    - How much time is lost to detention?
    - Are delays bunched in certain places, days or periods?

3.  Fleet Performance
    - Which trucks earn the most revenue?
    - Which are used the most?
    - Which have too much downtime?
    - Which use the most fuel?
    - What does maintenance cost for each truck?
    - Do older trucks perform worse than newer ones?

4.  Driver Performance & Safety
     - Which drivers have the best and worst on-time rates?
     - How many trips does each driver complete?
     - How productive is each driver? Which drivers have safety incidents?
     - What percent of incidents are preventable?
     - Which drivers, trucks and routes carry the most safety risk?

5.  Profitability & Cost Efficiency
     - Revenue by customer, route, truck and driver. Fuel cost and fuel efficiency. Maintenance cost. Cost per mile, revenue per mile, fuel cost per mile.
     - Which customers and routes are most valuable?
     - Which assets earn a lot but also cost a lot to run?

### Tools

| Tools | Purpose |
| --- | --- |
| Power BI | Data Modeling, DAX, Visualization |
| PowerPoint | Design the page background images |

---

## Build the data model

 - **Dimension tables** are lists of things: drivers, trucks, customers, routes, facilities, dates.
 - **Fact tables** record events: trips, deliveries, fuel purchases, repairs, incidents.
 - All are **one-to-many**, filter direction **single** (from the list table to the event table).
 - Data consists of **14 tables**.

![v1](asset/data_modeling.png)

### Facts about the data we confirmed with checks

- 85,410 loads and 85,410 trips, with exactly one trip per load.
- Every trip and every load has the status `Completed`.
- Data covers 36 months, from January 2022 to December 2024.
- Each trip has one `Pickup` stop and one `Delivery` stop (85,410 of each).
- The flag columns (`on_time_flag`, `preventable_flag`, `at_fault_flag`, `injury_flag`) hold True/False values.
- `utilization_rate` is stored as a fraction (a value of 1.0 means 100%) and ranges from 0.387 to 1.484, so some rows are above 100%.
- 150 drivers exist, 125 of them have trips. There are 170 incident rows.


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

---

## 5. Step 3: Build the visuals


There are **6 report pages**: D has two pages, and each of A, B, C, E has one pages.

Why B is separate: it works on **delivery date**, while the other pages work on **dispatch date**. If both were active, their selections would overlap and the numbers would change.

Why E has no customer, route or driver slicers: any of those filters would flip maintenance cost from actual to estimated for every chart on the page.

| Page | Name |
|---|---|
| A | Overall Operations |
| B1 | Delivery Performance (overview and timing) |
| C1 | Fleet Performance (revenue, utilization, downtime) |
| D1 | Driver Performance |
| D2 | Driver Safety |
| E1 | Profitability & Cost Efficiency |


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









## Question


## Processing



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








