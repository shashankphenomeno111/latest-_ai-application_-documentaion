# Interactive Market Dashboard Architecture

This document details the functional and technical architecture of the **Interactive Market Dashboard** in Project Sahyadri 2.0. It explains how the FastAPI backend and React frontend coordinate to calculate transport logistics, pledge finance scenarios, and render the custom 3D Ribbon Price Chart.

---

## 1. Step-by-Step Architectural Flow

The dashboard operates on a reactive cycle, executing calculations in real-time as the farmer interacts with the sliders:

```
[ Farmer Input Controls ] 
 (Crop, District, Quantity, Days)
           |
           v (Reactive State Updates)
[ Front-end State Dispatch ]
 (Sends API request to /api/recommend)
           |
           +----> District Fallback Mode: Requests standard regional calculations.
           +----> GPS Geolocation Mode: Passes exact user Lat/Lng.
           |
           v
[ Backend recommend_agent.py Optimization ]
 1. Queries spatial tables (Warehouses, DCCBs, APMC coordinates).
 2. Loads TFT model forecasts from forecast_agent.py.
 3. Runs vehicle selection ledger based on crop quantity.
 4. Calculates Net Revenue optimization for Sell vs. Hold.
           |
           v (Structured JSON Response Payload)
[ Front-end Rendering Engine ]
 1. Renders Logistics Ledger & exact Google Maps Direction URLs.
 2. Re-calculates Pledge Finance loan interest & storage rent sliders.
 3. Projects SVG path variables to draw the 3D Ribbon Chart.
```

---

## 2. The Core Backend Recommendation Solver

The recommendation engine, defined in [recommend_agent.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/backend/app/agents/recommend_agent.py), executes a mathematical financial optimization across competing APMC mandis, private mandis, and corporate buyers.

### A. Spatial matching & Crop Parameters
1. **Facility Querying:** Looks up the closest MSWC Warehouse and DCCB Bank branch relative to the target district (or user GPS coordinates).
2. **Crop-Specific Mapping:** If specific parameters are missing from the databases, it maps them dynamically based on the commodity:

| Commodity Class | Storage Rent (₹/qtl/month) | Loan-To-Value (LTV) |
| :--- | :--- | :---: |
| **Wheat, Rice, Maize, Cotton** | ₹10.00 | 80% (70% for Cotton) |
| **Gram, Pigeon Pea (Tur), Mug** | ₹11.00 | 75% |
| **Soyabean (Soybean), Sugarcane** | ₹12.00 | 75% (60% for Sugarcane) |
| **Turmeric, Chilli** | ₹13.00 | 70% |
| **Potato, Onion** | ₹14.00 – ₹15.00 | 65% |
| **Tomato** | ₹18.00 | 50% |

### B. Quantity-Based Vehicle Selection
To model freight costs accurately, the system selects transport vehicles based on load volume:
* **0 – 10 Quintals:** Mini Pickup (12.0 km/L) 🚐
* **10 – 30 Quintals:** Pickup Truck (8.0 km/L) 🚚
* **30 – 80 Quintals:** Medium Truck (5.0 km/L) 🚚
* **80 – 200 Quintals:** Heavy Truck (4.0 km/L) 🚛
* **200+ Quintals:** Multiple Heavy Trucks 🚛
* *Alternative:* Tractor + Trolley (3.5 km/L) 🚜 (if distance $\le$ 50 km and quantity is under 40 qtl).

### C. The Net Revenue Mathematical Models

For each market node, the engine compares two scenarios:

#### Scenario 1: Immediate Spot Liquidation (Sell Now)
$$\text{Net Revenue}_{\text{Immediate}} = (\text{Spot Price} \times Q) - \text{Transport Cost} - \text{Mandi Fees}$$
* **Transport Cost:** $\text{Diesel Price (₹95.00/L)} \times \left( \frac{\text{Distance}}{\text{Mileage}} \right) \times N_{\text{trucks}} + \text{Toll} + 500.0$ (handling base).
* **Mandi Fees:** APMCs charge ₹45.00/qtl, Private mandis charge ₹20.00/qtl, Direct Corporates charge ₹0.00/qtl.

#### Scenario 2: Pledge Warehousing & Holding (Hold for D Days)
$$\text{Net Revenue}_{\text{Hold}} = (\text{Forecasted Price}_D \times Q) - \text{Transport Cost} - \text{Mandi Fees} - \text{Rent}_D - \text{Interest}_D$$
* **Storage Rent:** $Q \times \text{Rent Rate} \times \left( \frac{D}{30} \right)$
* **Bank Interest:** $\text{Loan} \times \text{DCCB Interest Rate (p.a.)} \times \left( \frac{D}{365} \right)$
  *(where $\text{Loan} = \text{Spot Price} \times Q \times \text{LTV}$)*

If holding yields an additional gain of **$\ge$ ₹1,000** over immediate selling, the system advises **`HOLD`**; otherwise, it defaults to **`SELL_NOW`**.

---

## 3. Front-end Dashboard Architecture

The frontend is implemented in React ([Dashboard.jsx](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/frontend/src/components/Dashboard.jsx)), featuring a dark, premium, dashboard styling.

### A. Searchable Custom Select Dropdowns
* **Component:** `SearchableSelect`
* **z-Index Stacking Fix:** Traditional select containers suffer from clipping bugs when positioned inside layout cards. We resolved this by dynamically setting the trigger's wrapper z-index (e.g., `z-50` when expanded, `z-20` when closed), forcing dropdown panels to overlay all elements cleanly.
* **Full-text Localized Querying:** Filters choices against both English database keys and Marathi/Hindi translations on the fly.

### B. Geolocation (GPS mode)
* Clicking **"Use GPS Location"** triggers `navigator.geolocation.getCurrentPosition()`. 
* When coordinates are approved, the dashboard switches to GPS Mode, sending coordinates (`latitude`, `longitude`) to `/api/recommend`. The backend then overrides district-center routing to calculate transport costs based on the farmer's exact fields.

### C. 3D Isometric Ribbon Price Chart
To compare multiple markets simultaneously without cluttered lines, the dashboard builds a custom **3D Ribbon Chart** directly on an SVG canvas:

```
               Vertical Axis (Price Scaling)
                           ^
                           |    / (Depth Axis: Market Rank Z)
                           |   /
                           |  /
                           | /
                           +-------------------------> Horizontal Axis (Time idx X)
```

1. **Isometric Projection Formulas:** Maps 3D coordinate space $(x, y, z)$ into 2D screen coordinates $(px, py)$ using trigonometric projection:
   $$\begin{aligned}
   px &= 75 + x \cos(12^\circ) + z \cos(-22^\circ) \\
   py &= 155 - y + x \sin(12^\circ) + z \sin(-22^\circ)
   \end{aligned}$$
2. **Depth-Correct Z-Staggering (Painter's Algorithm):** To prevent ribbons from overlaying in front of one another incorrectly, each market's width is projected onto the depth axis ($z$) based on its net revenue rank:
   * **Rank 0 (Best Market):** $Z \in [16, 22]$ (front-most)
   * **Rank 1 (Second Best):** $Z \in [8, 14]$ (middle)
   * **Rank 2 (Third Best):** $Z \in [0, 6]$ (back-most)
   * **Rendering Sort:** The code sorts markets by rank and renders the back-most ribbon first (Rank 2), progressing forward to Rank 0. This guarantees a clean 3D perspective layering.
3. **Bezier Curve Interpolation:** Uses Cubic Bezier curves (SVG `C` tags) to smooth the pricing trajectories across the weeks:
   * Control points are generated at $1/3$ and $2/3$ of the width step to prevent sharp corners and create smooth pricing ribbons.

### D. Pledge Finance Recalculation Engine
* Features responsive sliders for Quantity and Days. 
* recency: Recalculates variables client-side immediately upon slider movements without firing slow API calls:
  * $\text{Warehouse Storage Cost} = Q \times \text{Warehouse Rent Rate} \times (D / 30)$
  * $\text{DCCB Cash Advance} = Q \times \text{Spot Rate} \times \text{LTV}$
  * $\text{Interest Accrued} = \text{DCCB Cash Advance} \times \text{Bank Interest Rate} \times (D / 365)$

### E. Logistics Ledger & Google Maps exact routing
* Exposes full cost breakdowns (diesel cost, petrol options, handling, toll fees).
* **Exact directions query:** Generates a Google Maps direction link. Rather than querying generic coordinates (which show empty map areas), it searches exact business registries:
  * e.g., `https://www.google.com/maps/dir/?api=1&origin=18.40,76.56&destination=APMC+Market+Nanded,+Nanded,+Maharashtra,+India`
  * This forces the Google Maps app to load the exact business card and navigate to its entrance.
