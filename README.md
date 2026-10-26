# Logistic-Operations-Analysis-and-Performance-Report


## Objective

An interactive Power BI report that shows how a trucking company performs across operations, deliveries, fleet, drivers and safety, and profitability. The aim is to give managers a clear, filterable view of the operation, so they can find where to look closer and make decisions based on data.


---

## Table of contents

- [Question](#Questions)
- [Tools](#Tools)
- [Processing](#Processing)
- [Data model](#Data-model)
- [Methodology](#Methodology)
- [Visualization](#Visualization)
- [Insights](#Insights)
- [Recommendations](#Recommendations)


### Questions 

1.  Overall Operations (How healthy and efficient is the operation?)
    - How many loads and trips are completed?
    - What is total revenue?
    - What is the on-time delivery rate?
    - How well are trucks and drivers used?
    - Are the KPIs getting better or worse over time?

2.  Delivery Performance (Where and why are deliveries delayed?)
    - What percent of deliveries are on time?
    - Which facilities have the most detention and delays?
    - Which routes do best and worst? Which customers see the most delays?
    - How much time is lost to detention?
    - Are delays bunched in certain places, days or periods?

3.  Fleet Performance (Are we getting enough value from our trucks?)
    - Which trucks earn the most revenue?
    - Which are used the most?
    - Which have too much downtime?
    - Which use the most fuel?
    - What does maintenance cost for each truck?
    - Do older trucks perform worse than newer ones?

4.  Driver Performance & Safety (Which drivers perform well, and where are the safety risks?)
     - Which drivers have the best and worst on-time rates?
     - How many trips does each driver complete?
     - How productive is each driver? Which drivers have safety incidents?
     - What percent of incidents are preventable?
     - Which drivers, trucks and routes carry the most safety risk?

5.  Profitability & Cost Efficiency (Where is revenue earned efficiently, and where do costs leak?)
     - Revenue by customer, route, truck and driver. Fuel cost and fuel efficiency. Maintenance cost. Cost per mile, revenue per mile, fuel cost per mile.
     - Which customers and routes are most valuable?
     - Which assets earn a lot but also cost a lot to run?

---

### Tools

| Tools | Purpose |
| --- | --- |
| Power BI | Data Modeling, DAX, Visualization |
| PowerPoint | Design the page background images |


---


## Data Model

 - **Dimension tables** are lists of things: drivers, trucks, customers, routes, facilities, dates.
 - **Fact tables** record events: trips, deliveries, fuel purchases, repairs, incidents.
 - All are **one-to-many**, filter direction **single** (from the list table to the event table).
 - Data consists of **14 tables**.

![v1](asset/data_modeling.png)


---

### Data Quality checks

- 85,410 loads and 85,410 trips, with exactly one trip per load.
- Every trip and every load has the status `Completed`.
- Data covers 36 months, from January 2022 to December 2024.
- Each trip has one `Pickup` stop and one `Delivery` stop (85,410 of each).
- The flag columns (`on_time_flag`, `preventable_flag`, `at_fault_flag`, `injury_flag`) hold True/False values.
- `utilization_rate` is stored as a fraction (a value of 1.0 means 100%) and ranges from 0.387 to 1.484, so some rows are above 100%.
- 150 drivers exist, 125 of them have trips. There are 170 incident rows.
- Every section file ends with one `EVALUATE ROW(...)` that returns checking numbers. Two kinds of checks were used:

- **Known totals.** Numbers that must match the data exactly (for example, completed trips must equal the number of trips in the table).
- **"Summed by" checks.** The measure is added up across every driver, truck, customer or route. The sum must equal the plain total. If it does not, a relationship is missing or wrong.
- Trip damage cost equals the damage total from Section D; allocated maintenance summed by route equals actual maintenance; Contribution Margin summed by customer, route, driver and truck equals the total.
- Truck-months 3,312; revenue, fuel, downtime and maintenance summed by truck equal the totals; maintenance cost equals labor plus parts.
- Trips summed by driver 85,410; 170 incidents summed by driver, truck and route; Risk Points equals 170 + at-fault + preventable + 3 × injury.
- Completed trips, loads and delivery stops each equal 85,410; revenue measure equals the sum of load revenue (plus surcharges after the revenue update).
- Deliveries 85,410; pickups 85,410; stops 170,820; on-time and late stops add up to total stops; deliveries and stops summed by month, customer, route and facility equal the totals.

--- 

## Visualization 
 
 - There are **6 report pages**: D has two pages, and each of A, B, C, E has one pages.
 - Why B is separate: it works on **delivery date**, while the other pages work on **dispatch date**. If both were active, their selections would overlap and the numbers would change.
 - Why E has no customer, route or driver slicers: any of those filters would flip maintenance cost from actual to estimated for every chart on the page.

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


---


### Processing


---

## Insights


**Overall Operations**
    - 8.32m current month revenue shows 3% increase vs previous month.
    - 2k current month completed trips with 2% decrease vs previous month.
    - Slight difference between 2024 and 2022 compared to 2023.
    - 43.9% on time delivery rate shows 9.1% increase vs on-time delivery target (95).
    - 2023 spent less than 2% idle time compare to 2022 & 2024
**Delivery Performance**
    - Elite Distribution Center appears to be the worst customer (73.8%)
    - Detention rate stands at 87% and most detention happens at Phoenix Distribution Center having over 10k detention hrs.
    - Houston TX to Portland becomes the best routes with rate of 42.8% success.
    - Indianapolis Warehouse tends to be the worst facility having 47% late rate.
    - Denver CO to Kansas is the worst routes with 65.1%.
    - Saturday as the worst day having 52% rate.

**Fleet Performance**
    - Trucks between 5 to 7 yrs tends to perform worse having the highest number of breakdown and cost more in maintenance,parts and labor.
    - Downtime and maintenance cost decreased significantly in 2024 compared to previous two years.
    - Inspected vehicles cost the least in maintenance cost.
    - Freightliner vehicles breakdown the most due to them being old or mostly out of service.
    - Freightliner cover over 75% of vehicles with most downtime hours (over 70k) and loss in revenue.
**Driver performance & Safety**
    - Jessica Johnson gets the most in on time delivery rate (52%), while Mary Wilson tends to top worst on time delivery rate (37.3%).
    - $2.39m is the total revenue generated per active driver.
    - Davide Miller has the highest risk driver involving in 7 incidents at 7.5 per 1m mile.
    - 37.65% of of incidents could be avoided.
    - Philadelphia is the route with highest risk having a total of 21 incidents.

**Profitability & Cost Efficiency**
    - Total revenue is $256.26m and a contribution rate of 65%.
    - More money is spent on fuel and maintenance and labor.
    - Year 2024 has an increment in fuel price per gallon ($4.2).


---
### Recommendations








