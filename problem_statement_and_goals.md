# Problem Statement & Project Goals

This document outlines the core agricultural challenges in Maharashtra that **Project Sahyadri 2.0** addresses, the financial impact of these challenges on farmers, and the specific goals the project achieves.

---

## 1. The Core Problem: The Decision Dilemma

Farmers in Maharashtra face a critical, double-sided dilemma immediately following their harvests:

```
                  +---------------------------------------+
                  |           POST-HARVEST PHASE          |
                  +-------------------+-------------------+
                                      |
                                      v
                  +-------------------+-------------------+
                  |          THE CRITICAL QUESTION        |
                  +-------------------+-------------------+
                                      |
                   +------------------+------------------+
                   |                                     |
                   v                                     v
      +----------------------------+       +----------------------------+
      |       WHEN TO SELL?        |       |       WHERE TO SELL?       |
      |   (Sell immediately, or    |       |  (APMC mandi, private      |
      |   hold crop for a rise?)   |       |  market, corporate buyer?) |
      +----------------------------+       +----------------------------+
```

Historically, farmers are forced to answer these questions with incomplete, highly localized, and static information, leading to sub-optimal decisions that significantly reduce their net household incomes.

---

## 2. Key Pain Points

### A. Extreme Information Asymmetry
Farmers rely on fragmented, disconnected systems to make selling decisions:
* **Market Prices:** Tracking Agmarknet or physical boards at local APMCs.
* **Weather Reports:** Local updates from IMD (India Meteorological Department).
* **Storage Options:** Inquiring in-person about space availability at Maharashtra State Warehousing Corporation (MSWC) warehouses.
* **Logistics Costs:** Calling local truck operators for custom freight rates.
* **Credit Facilities:** Contacting cooperative societies for credit lines.

There is no platform that fuses these distinct domains into a unified, actionable recommendation.

### B. Distress Selling
Without access to intermediate liquidity, smallholder farmers often resort to **distress selling**—selling their entire harvest immediately at the nearest mandi at whatever spot price is available. They do this because:
1. They need immediate cash to pay off crop loans, purchase inputs for the next cycle, or cover family expenses.
2. They are unaware of nearby storage options or how to obtain low-interest **Warehouse Receipt Loans** from District Central Cooperative Banks (DCCB).

### C. Financial Impact
At scale, this lack of decision intelligence leads to losses of **₹500 to ₹1,500 per quintal** on sub-optimal sales. For a small farmer with a modest yield, this represents the difference between a profitable season and entering a cycle of debt.

---

## 3. Concrete Example: The Soybean Dilemma

Consider a typical scenario for a soybean farmer located in **Latur, Maharashtra**:

```
+---------------------------------------------------------------------------------+
|                                 WITHOUT SAHYADRI                                |
+---------------------------------------------------------------------------------+
| 1. Harvests Soybeans in Latur.                                                  |
| 2. Immediately transports crop to Latur APMC (nearest mandi).                   |
| 3. Sells at the current spot rate: ₹4,200/qtl.                                  |
| 4. Pays high transportation fees and faces localized buyer cartels.              |
|                                                                                 |
| * NET OUTCOME: Low profit margin, immediate distress liquidation.               |
+---------------------------------------------------------------------------------+

                                         VS

+---------------------------------------------------------------------------------+
|                                  WITH SAHYADRI                                  |
+---------------------------------------------------------------------------------+
| 1. Sahyadri's TFT model projects Nanded APMC price will rise by ₹160/qtl        |
|    in 14 days.                                                                  |
| 2. The spatial index finds MSWC Udgir Warehouse has 1,400 tonnes of free capacity|
|    and is located 42km away.                                                     |
| 3. Sahyadri finds a DCCB Bank branch nearby offering a Warehouse Receipt Loan   |
|    at 7% interest to cover immediate cash needs.                                |
| 4. The math engine calculates:                                                  |
|    Estimated price upside - (Storage fees + Transport to Udgir + Loan interest) |
|                                                                                 |
| * RECOMMENDED STRATEGY: Hold 14 days, store at MSWC Udgir, take a DCCB loan,    |
|   and sell at Nanded APMC.                                                      |
| * NET OUTCOME: Net gain of ₹6,400 per hectare!                                  |
+---------------------------------------------------------------------------------+
```

---

## 4. Project Goals: What We Are Achieving

Project Sahyadri 2.0 is designed to bridge these gaps by achieving four core objectives:

### 1. Data Fusion (Multi-Domain Aggregation)
Fusing multiple data streams to build a comprehensive map of the agricultural ecosystem:
* **Market Dynamics:** Historical transactions, arrival volumes, and price fluctuations from government APMCs, private mandis, and corporate buyers.
* **Geospatial Assets:** Location and capacity indices of government warehouses.
* **Financial Integration:** Physical bank locations, District Central Cooperative Banks (DCCB), and available credit channels.
* **Environmental Data:** Temperature, precipitation, and soil properties.

### 2. Probabilistic Deep Learning Price Forecasting
Instead of simple linear regressions, we train a **Temporal Fusion Transformer (TFT)** on Kaggle GPUs (`Tesla T4 x2`). It outputs:
* **Optimistic Price Bounds (90th Percentile):** What farmers can make in a high-demand scenario.
* **Expected Modal Prices (50th Percentile):** The most likely future market rate.
* **Conservative Price Bounds (10th Percentile):** Lower bounds to manage risk, ensuring farmers don't lose money on storage overheads if prices drop.

### 3. Location-Aware Network Routing
Using geographic coordinates to calculate actual travel distances, times, and fuel/logistics overheads. This helps farmers compare local APMCs against distant mandis by factoring in transport costs.

### 4. Interactive, Multi-Agent Decision Interfaces
Creating intuitive user interfaces:
* **Interactive 3D Dashboards:** To visualize arrival volumes and price forecasts.
* **Voice-Enabled AI Chatbot:** An intelligent assistant (powered by specialized agents for forecasting, weather, diseases, and spatial recommendations) that communicates in local languages (Marathi, Hindi) and supports voice input for hands-free queries.
