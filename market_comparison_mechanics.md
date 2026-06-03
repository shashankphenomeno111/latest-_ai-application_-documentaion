# Market Comparison Mechanics & Financial Optimization

This document explains how Project Sahyadri 2.0 compares government APMCs, private mandis, and corporate procurement channels to compute the optimal sales strategy for farmers.

---

## 1. The Market Comparison Flow

When a farmer initiates an analysis on the React dashboard or queries the AI chatbot, the backend executes a multi-step comparison pipeline:

```
[ Frontend Selection ]
 (Crop, District, Quantity)
           |
           v
[ Discovery of Competing Markets ]
 (Searches for APMCs, Private Mandis, and Corporates)
           |
           v
[ Real-time Price Forecasting ]
 (Extracts Spot, 7D, 14D, and 30D forecasts from TFT model)
           |
           v
[ Logistics & Fuel Cost Calculation ]
 (Determines vehicle size, fuel requirements, and mileage cost)
           |
           v
[ Storage & Financing Simulation ]
 (Simulates warehouse rents and DCCB interest rates)
           |
           v
[ Net Revenue Mathematical Compare ]
 (Sorts markets by highest Net Best payout)
           |
           v
[ Frontend Tabbed Comparison UI ]
 (Farmer interacts with Spot vs. 7D vs. 14D vs. 30D rankings)
```

---

## 2. Step 1: Discovering Competing Markets

To offer true arbitrage comparisons, the backend does not limit search results to a single government APMC. It discovers up to **5 candidate market nodes** across three separate trade channels:
1. **APMC Mandis:** State-regulated agricultural markets (government auctions).
2. **Private Markets:** Licensed private mandis and retail purchase points.
3. **Corporate Buyers:** Direct agro-industrial processing hubs (direct corporate procurement).

### Proximity Search Fallback:
If the database contains fewer than 2 active market nodes in the farmer's immediate district, the discovery engine expands its search radius to scan neighboring districts and statewide active nodes, ensuring the comparison lists multiple options.

---

## 3. Step 2: Price Projection Parsing

For each discovered market, the system fetches price projections across four distinct horizons:
* **Spot (Today):** The current benchmark spot price.
* **7-Day Forecast:** Next week's projected rate.
* **14-Day Forecast:** Two weeks' projected rate.
* **30-Day Forecast:** One month's projected rate.

These values are predicted by the Google **Temporal Fusion Transformer (TFT)** model, which incorporates historical price trends and yearly seasonality.

---

## 4. Step 3: Logistics Cost Breakdown

To calculate the cost of transporting the crop from the farm to each market, the system runs an automated logistics analyzer:
1. **Vehicle Allocation:** Dynamically matches the transaction quantity to the most economical vehicle type:
   * **$\le$ 10 Qtl:** Mini Pickup (12.0 km/L) 🚐
   * **10 – 30 Qtl:** Pickup Truck (8.0 km/L) 🚚
   * **30 – 80 Qtl:** Medium Truck (5.0 km/L) 🚚
   * **80 – 200 Qtl:** Heavy Truck (4.0 km/L) 🚛
   * **> 200 Qtl:** Multiple Heavy Trucks ($N = \lceil\text{Qty}/200\rceil$) 🚛
2. **Fuel Cost Computation:** Computes the total diesel fuel expense based on distance and mileage:
   $$\text{Fuel Cost} = \left( \frac{\text{Distance}}{\text{Mileage}} \right) \times \text{Diesel Price (₹95.00/L)} \times N_{\text{trucks}}$$

---

## 5. Step 4: Storage & Financial Financing Simulation

For scenarios where the farmer holds the crop to wait for a price rise, the system simulates warehousing overheads using data from the nearest **MSWC Warehouse** and **DCCB Bank**:
1. **Warehouse Rent:** Storage fees are calculated based on the crop's specific rate (e.g., ₹12/qtl/month for Soyabean) and holding days ($D$):
   $$\text{Storage Rent} = \text{Quantity} \times \text{Monthly Rent Rate} \times \left( \frac{D}{30} \right)$$
2. **DCCB pledge loan value:** To provide immediate cash, the farmer is assumed to take a bank loan against their stored warehouse receipt at the crop's LTV (Loan-To-Value) ratio:
   $$\text{Loan Value} = \text{Quantity} \times \text{Spot Price} \times \text{Crop LTV} \quad (\text{e.g., } 75\% \text{ LTV for Soyabean})$$
3. **Loan Interest Accrued:** The interest cost to service the cash advance is calculated using the nearest DCCB branch's interest rate (e.g. 7.5% p.a.):
   $$\text{Interest Cost} = \text{Loan Value} \times \text{DCCB Interest Rate} \times \left( \frac{D}{365} \right)$$

---

## 6. Step 5: Net Payout Optimization

The solver compares the net payout for both strategies to find the most profitable option:

### Strategy A: Sell Immediately (Today)
$$\text{Net Revenue}_{\text{Immediate}} = (\text{Spot Price} \times \text{Quantity}) - \text{Transport Cost}$$

### Strategy B: Store & Sell Later (Hold for D Days)
$$\text{Net Revenue}_{\text{Hold}} = (\text{Predicted Price}_D \times \text{Quantity}) - \text{Transport Cost} - \text{Storage Rent} - \text{Interest Cost}$$

The backend sorts the market nodes in descending order of their maximum potential return (`net_best`).

---

## 7. Step 6: Frontend Tabbed Comparison UI

On the frontend React dashboard ([Dashboard.jsx](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/frontend/src/components/Dashboard.jsx)), these comparisons are displayed inside the **"Comparative Rankings"** panel:

```
+-------------------------------------------------------------+
|                     COMPARATIVE RANKINGS                    |
|  [ Spot ]     [ 7 Days ]     [ 14 Days ]     [ 30 Days ]    | <-- Timeframe Selection Tabs
+-------------------------------------------------------------+
|                                                             |
|  1. APMC Market Latur    (APMC Mandi)      ₹4,850/qtl  ★BEST|
|     Net Return: ₹2,42,500 | Vehicle: Pickup Truck (25km)    |
|                                                             |
|  2. Reliance Retail Co.  (Corporate Buyer) ₹4,750/qtl       |
|     Net Return: ₹2,37,500 | Vehicle: Pickup Truck (45km)    |
|                                                             |
|  3. Nanded Private Mandi (Private Mandi)   ₹4,680/qtl       |
|     Net Return: ₹2,34,000 | Vehicle: Medium Truck (80km)    |
|                                                             |
+-------------------------------------------------------------+
```

### Farmer Interaction Features:
* **Interactive Tabs:** The farmer can click between **Spot, 7 Days, 14 Days, and 30 Days**. The frontend dynamically re-sorts and updates the list based on the prices projected for that specific timeframe.
* **Arbitrage Badge:** Highlights the highest-paying market option for the selected timeframe.
* **Detailed Breakdown Cards:** Farmers can expand any market entry to inspect the recommended vehicle alternative list, fuel requirements in liters, and mileage calculations.
