# UBER Trip Data Analysis - using SQLite & Power BI

## NOW: I performed data analysis using Python, Pandas, NumPy, Matplotlib, and Seaborn.

> **Project summary:** Interactive Power BI solution analyzing 103,728 Uber trips (one main Trip Details table) with a Location table (256 rows), a Calendar table (30 rows) and a Dynamic Measure table. This repository/dashboard demonstrates data modelling (star schema), DAX measures, KPIs, time & location analysis, and a 3‑page report (Overview, Time Analysis, Details Analysis).

---
### Overview page

<img src="https://github.com/omkarshinde25/Uber-Trip-Analysis/blob/main/Dashboards/Uber%20Trip%20Analysis%20Overview.png" width="800"> <br>


## Table of contents

1. Project overview
2. Files & images included
3. Data model (star schema)
4. Data preparation & transformations (step‑by‑step)
5. DAX measures (full code & explanation)
6. Dashboard pages & visualisations (detailed point‑by‑point)
7. Navigation, bookmarks & buttons (how to add PNG as page navigator)
8. Performance recommendations
9. How to reproduce / deliverables
10. Drill-Down and Interactivity

---

## 1. Project overview

This project analyses a single large trips table (103,728 rows) and two small lookup tables (Location and Calendar). Business goals:

* Monitor total bookings, revenue (fare + surge), distance and time KPIs.
* Discover busiest pickup/dropoff locations and farthest trips.
* Analyse temporal patterns (hour/day heatmap, hourly totals, weekday behaviour).
* Provide drillable dashboards with navigation and bookmarks for storytelling.

Audience: product/operations analysts, city managers, BI reviewers.

---

## 2. Files & images included

* `Trip Details` table — main fact table (103,728 rows)
* `Location Table` — 256 rows (LocationID, Location, City)
* `Calendar` table — 30 rows (Date, DayName, DayNum)
* `Dynamic Measure` — small DAX table to switch metrics in visuals

Dashboard images (embedded below for quick preview):


### Time Analysis page

<img src="https://github.com/omkarshinde25/Uber-Trip-Analysis/blob/main/Dashboards/Uber%20Trip%20Time%20Analysis.png" width="800"> <br>

### Details Analysis page

<img src="https://github.com/omkarshinde25/Uber-Trip-Analysis/blob/main/Dashboards/Uber%20Trip%20Detail%20Analysis.png" width="800"> <br>


---

## 3. Data model (Star Schema)

<img src="https://github.com/omkarshinde25/Uber-Trip-Analysis/blob/main/Dashboards/Uber%20Trip%20Analysis%20Star%20Schema.png" width="800"> <br>

**Structure**

* Fact: `Trip Details` — contains transactional fields: TripID, Pickup Time, Drop Off Time, PULocationID, DOLocationID, trip_distance, fare_amount, Surge Fee, passenger_count, vehicle type, payment_type, etc.
* Dimension: `Location Table` — LocationID, Location, City, (other location metadata)
* Dimension: `Calendar` — Date, DayName, DayNum, Month, Year, WeekOfYear, IsWeekend
* Utility: `Dynamic Measure` — table with names and references to measures for UI switching.

**Relationships**

* `Trip Details[PULocationID]` -> `Location Table[LocationID]` (active)
* `Trip Details[DOLocationID]` -> `Location Table[LocationID]` (inactive; use USERELATIONSHIP when needed)
* `Trip Details[Pick Up Date]` -> `Calendar[Date]` (active)

---

## 4. Data preparation & transformations (step‑by‑step)

1. **Import** the `Trip Details`, `Location Table` and `Calendar` into Power BI Desktop (Get Data → Excel).
2. **Power Query clean-up:**

   * Ensure `Pickup Time` and `Drop Off Time` are DateTime type.
   * Trim spaces on string columns (payment_type, vehicle, etc.).
   * Replace nulls: For numeric nulls use 0 (fare_amount, surge_fee) or appropriate sentinel values.
   * Create `Pick Up Date` column:

     ```powerquery
     Date.From([Pickup Time])
     ```
   * Remove duplicate location rows in `Location Table`.
3. **Calendar table:** Use your 30‑row calendar or create a full date table if you need longer date ranges. Include DayName and DayNum columns.
4. **Load** cleaned tables to the model.
5. **Set relationships** in Model view (see section 3). Mark `Calendar` as date table if relevant (`Mark as Date Table`).
6. **Create indexes** (if necessary) or reduce cardinality: convert high-cardinality text to keys where possible.

---

## 5. DAX measures 

### Dynamic Measure (table code)

```dax
Dynamic Measure = {
    ("Total Bookings", NAMEOF('Trip Details'[Total Bookings]), 0),
    ("Total Booking Value", NAMEOF('Trip Details'[Total Booking Value]), 1),
    ("Total Trip Distance", NAMEOF('Trip Details'[Total Trip Distance Measure1]), 2)
}
```

### Key KPI measures

```dax
-- 1) Total Bookings
Total Bookings = COUNT('Trip Details'[Trip ID])

-- 2) Total Booking Value
Total Booking Value = SUM('Trip Details'[fare_amount]) + SUM('Trip Details'[Surge Fee])

-- 3) Avg Booking Value
Avg Booking Value = DIVIDE([Total Booking Value], [Total Bookings], BLANK())

-- 4) Total Trip Distance
Total Trip Distance Measure1 = SUM('Trip Details'[trip_distance])

-- 5) Avg Trip Distance (formatted text)
Avg Trip Distance =
VAR AvgMiles = AVERAGE('Trip Details'[trip_distance])
RETURN
CONCATENATE(FORMAT(AvgMiles, "0"), " miles")

-- 6) Avg Trip Time (minutes)
Avg Trip Time =
VAR AvgTrip = AVERAGEX('Trip Details', DATEDIFF('Trip Details'[Pickup Time], 'Trip Details'[Drop Off Time], MINUTE))
RETURN
CONCATENATE(FORMAT(AvgTrip, "0"), " Min")
```

### Location measures

```dax
-- Most frequent Dropoff point (uses inactive relationship)
Most frequent Dropoff point =
VAR DropOffCounts =
    ADDCOLUMNS(
        SUMMARIZE('Trip Details','Location Table'[Location]),
        "DropOffCount",
            CALCULATE(
                COUNT('Trip Details'[Trip ID]),
                USERELATIONSHIP('Trip Details'[DOLocationID], 'Location Table'[LocationID])
            )
    )
VAR RankDropOffs = ADDCOLUMNS(DropOffCounts, "Rank", RANKX(DropOffCounts, [DropOffCount], , DESC, DENSE))
VAR TopDropOff = FILTER(RankDropOffs, [Rank] = 1)
RETURN
CONCATENATEX(TopDropOff, 'Location Table'[Location], ",")

-- Most frequent pickup point (TOPN implementation)
Most frequent pickup point =
VAR PickPoint = TOPN(
    1,
    SUMMARIZE('Trip Details', 'Location Table'[Location], "Pickup Point", COUNT('Trip Details'[Trip ID])),
    [Pickup Point], DESC
)
RETURN
CONCATENATEX(PickPoint, 'Location Table'[Location], ",")

-- Farthest Trip (pickup->dropoff for max distance)
Farthest Trip =
VAR MaxDistance = MAX('Trip Details'[trip_distance])
VAR PickupLocation = LOOKUPVALUE(
    'Location Table'[Location], 'Location Table'[LocationID],
    CALCULATE(SELECTEDVALUE('Trip Details'[PULocationID]), 'Trip Details'[trip_distance] = MaxDistance)
)
VAR DropoffLocation = LOOKUPVALUE(
    'Location Table'[Location], 'Location Table'[LocationID],
    CALCULATE(SELECTEDVALUE('Trip Details'[DOLocationID]), 'Trip Details'[trip_distance] = MaxDistance)
)
RETURN
"Pickup:" & PickupLocation & " -> Drop-off: " & DropoffLocation & " (" & FORMAT(MaxDistance, "0.0") & " miles)"
```

### Day / Night flag

```dax
Day/Night =
VAR HourValue = HOUR('Trip Details'[Pickup Time])
RETURN
IF(HourValue >= 6 && HourValue <= 19, "Day", "Night")
```

### Additional helper columns used in Time Analysis

```dax
Hour(HH:MM:SS) = HOUR('Trip Details'[Pickup Time])
Pick Up Date = DATE(YEAR('Trip Details'[Pickup Time]), MONTH('Trip Details'[Pickup Time]), DAY('Trip Details'[Pickup Time]))
```

---

## 6. Dashboard pages & visualisations (detailed)

### Overview page (first image)

* **Top KPI cards (6):** Total Bookings, Total Booking Value, Avg Booking Value, Total Trip Distance, Avg Trip Distance, Avg Trip Time.

  * These are formatted as big tiles and use the DAX measures above. Keep labels short and add small icons.
* **Metric selector (Dynamic Measure):** A segmented control (slicer) driven by `Dynamic Measure` to switch between Total Bookings / Booking Value / Trip Distance on charts.
* **Two donut charts:** Payment type distribution & Day vs Night booking distribution. Show percent and absolute numbers.
* **Location Analysis box:**

  * Most frequent pickup point (text card using measure)
  * Most frequent dropoff point (text card using measure)
  * Farthest trip (text concatenated measure)
* **Vehicle Type Analysis table:** Small table visual with vehicle icons, total bookings, booking value, avg value and total trip distance. Use conditional formatting to highlight top rows.
* **Bar charts:** Booking value by location and count by vehicle type.

> Visual tips: keep consistent font, align tiles in a grid, and use subtle borders/shadows for depth.

### Time Analysis page (second image)

* **Line / area charts:** Total booking values by pickup time (hour bucket) and by day name.
* **Heatmap (matrix visual):** Total Booking Values by Hour & Day — use a matrix visual with conditional formatting to create a heatmap look. Right side: stacked bar of hourly totals.
* **Use the Dynamic Measure slicer** at top to switch metric displayed on the heatmap.

Explanation for viewers: The heatmap identifies peak demand windows (hours x weekdays). Recommend using this for surge planning and driver allocation.

### Details Analysis page (third image)

* **Detailed table (table/matrix):** Show raw transactional rows with columns: Trip ID, Pick Up Date, Vehicle, Payment_type, Passenger Count, Total Trip Distance, Total Booking Value, Pickup LocationID, Pickup Hour, Total Bookings (1 per row). Enable vertical scroll and totals at bottom.
* **Usage:** Enables audit & row-level validation. Keep a page with export enabled so analysts can export filtered rows to CSV.

---

## 7. Navigation, bookmarks & page navigator PNG

**Add a PNG icon as a page navigator / bookmark button**

1. Import your PNG (e.g. small home icon) into Power BI (Insert → Image).
2. With the image selected, create a bookmark that captures the current page or a filtered state.
3. Go to `View` → `Bookmarks Pane`. Create a bookmark (e.g. `Overview_State`).
4. While the image is selected, go to `Action` in the Visualizations pane, turn `Action` on, set `Type` to `Bookmark` and pick the bookmark you created.
5. Repeat for other page images (Time, Details). Place all page images in a left rail for consistent navigation. You can also set `Back` behavior on a separate back-arrow PNG.

**Add the PNG image to the README**: I added a comment in the top of this README to show where to include the PNG image for your navigation bar. In Power BI, the image acts as a clickable navigation element with bookmarks.

---

## 8. Performance recommendations

* **Import vs DirectQuery:** Import the dataset (103k rows is fine). Keep columns narrow and remove unused columns before loading.
* **Reduce cardinality:** Replace long text columns with keys when possible (e.g., vehicle type IDs).
* **Use measures not calculated columns** where possible (measures are evaluated on the fly, calculated columns increase model size).
* **Avoid row-by-row operations** in measures for large visuals. Use summarized operations (SUMMARIZE, GROUPBY) carefully.
* **Aggregate tables**: create pre-aggregated summary tables for very-heavy visuals (hourly rollups) if you expand the dataset later.

---

## 9. How to reproduce / deliverables

1. Files to deliver: Power BI `.pbix`, exported `README.md` (this file), relationship diagram PNG, images used for page navigation (PNG icons), and an export CSV of sample rows.
2. Steps to reproduce:

   * Open `Uber_Trip_Analysis.pbix` in Power BI Desktop.
   * Load the `Trip Details`, `Location Table` and `Calendar`.
   * Recreate relationships and mark date table.
   * Paste the DAX measures above.
   * Add slicers: Date range (between), City, Dynamic Measure.
   * Add bookmarks and link them to image actions for page navigation.
3. Publish to Power BI Service if you want web sharing, and configure dataset refresh if source is live.

---

## 10. Drill-Down and Interactivity

The dashboard includes an **interactive drill-down feature** connecting the **Overview page KPIs** — **Total Bookings**, **Total Booking Value**, and **Total Trip Distance** — to the **3rd dashboard (Trip Details Analysis)**.

Users can simply **click any KPI card** to navigate directly to the detailed report view for that specific metric. This enables deeper insights into **booking-level data**, **route patterns**, and **time-based trends**.

The drill-through link is implemented using **Power BI’s “Drillthrough” functionality**, ensuring filters are passed automatically to the **Details page**, making the exploration **smooth**, **dynamic**, and **user-friendly**.
