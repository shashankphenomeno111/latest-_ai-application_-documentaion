# Spatial Matching, Proximity Analytics & Dynamic Maps Routing

This document details the spatial engineering, distance mathematics, and mapping redirection techniques used in Project Sahyadri 2.0 to calculate routes from a farmer's location (District center vs. GPS fields) to regional facilities.

---

## 1. Spatial Processing Architecture

The backend's [spatial_service.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/backend/app/utils/spatial_service.py) coordinates query filters and distance scoring when a user requests local asset analytics:

```
                  [ API Request received ]
               (/api/spatial or /api/recommend)
                             |
                             v
                  [ Resolve District Center ]
             (Look up default lat/lng coordinate)
                             |
                             v
               [ Is browser GPS data passed? ]
                             |
         +-------------------+-------------------+
         | (Yes)                                 | (No)
         v                                       v
[ Set Origin = GPS Coordinates ]      [ Set Origin = District center coordinates ]
(High-precision farmer field)         (Fallback geographic centroid)
         |                                       |
         +-------------------+-------------------+
                             |
                             v
                  [ Match District Aliases ]
           (PUNE/POONA, NASIK/NASHIK, OSMANABAD/DHARASHIV)
                             |
                             v
              [ Query DB Operational Tables ]
         (Warehouses, DCCBs, Soil Labs, KVK Stations)
                             |
                     Were results found?
                             |
         +-------------------+-------------------+
         | (Yes)                                 | (No)
         v                                       v
[ Return matched assets list ]        [ Run Statewide Proximity Query ]
                                      (Fetch all statewide facilities, calculate
                                       Haversine distance, return top-5 closest)
                             |
                             v
                [ Build Google Maps URLs ]
            (Maps routing: Origin -> Destination)
```

---

## 2. Spatial Query Logic & Robust Alias Matching

To prevent query failures due to spelling differences or spelling reforms, the system handles district name aliases:
1. **District Aliasing:** The utility `get_aliases_for_district` retrieves synonyms (e.g. `AHMEDNAGAR` maps to `AHILYANAGAR`, `OSMANABAD` maps to `DHARASHIV`, `AURANGABAD` maps to `CHHATRAPATI SAMBHAJINAGAR`).
2. **Database Queries:** Runs SQL matches using wildcard `LIKE` operators against all synonyms to fetch records.
3. **Statewide Fallback (Proximity Search):**
   > [!TIP]
   > If a farmer queries a district that lacks a specific facility in the database (e.g. a district without a Soil Testing Lab), the pipeline executes a **statewide fallback query**. It pulls all statewide facilities, computes the distance to each, sorts them, and returns the **5 closest facilities** cross-district. This guarantees the user is never left with an empty screen.

---

## 3. Distance Calculation: The Haversine Formula

To calculate the straight-line distance between two points on the Earth's surface without expensive geospatial GIS server extensions, the backend runs the **Haversine Formula**:

$$d = 2 r \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta \text{lat}}{2}\right) + \cos(\text{lat}_1)\cos(\text{lat}_2)\sin^2\left(\frac{\Delta \text{lon}}{2}\right)}\right)$$

Where:
* $\text{lat}_1, \text{lon}_1$ are the starting coordinates (farmer field GPS or district center).
* $\text{lat}_2, \text{lon}_2$ are the destination coordinates (warehouse or bank).
* $r$ is the Earth's average radius ($6371.0 \text{ km}$).
* $d$ is the calculated travel distance in kilometers.

---

## 4. Dynamic Google Maps Routing Redirection

The exact technique used to construct routes is handled by the `_build_maps_url` function, which maps parameters depending on the active coordinate modes.

The system builds URLs conforming to the standard **Google Maps Directions API** structure:
$$\text{https://www.google.com/maps/dir/?api=1\&origin=}\{\text{Origin}\}\text{\&destination=}\{\text{Destination}\}$$

### A. Dynamic Origin Determination
* **Mode 1: GPS Mode (High Precision)**
  * Triggered when the browser's location API returns the user's current GPS position (`navigator.geolocation.getCurrentPosition`).
  * The frontend passes `latitude` and `longitude` to the backend.
  * The backend sets:
    $$\text{Origin} = \text{"latitude,longitude"} \quad (\text{e.g., } \text{origin=18.4088,76.5603})$$
* **Mode 2: District Mode (Regional Fallback)**
  * Triggered when GPS is disabled or permission is denied.
  * The backend falls back to the regional text query, URL-encoding the district center:
    $$\text{Origin} = \text{url\_encode("DistrictName, Maharashtra, India")}$$

### B. Dynamic Destination Optimization
Rather than passing raw coordinates for the destination (which might pin an empty field or back road near the facility), the system builds a text query containing the facility's business name:
$$\text{Destination} = \text{url\_encode("MSWC Warehouse } + \text{PlantName} + \text{, Address, Maharashtra, India")}$$

#### Why use this naming technique?
> [!IMPORTANT]
> Passing the business name string (e.g. `MSWC Warehouse Udgir`) instead of coordinate pairs (`18.39, 77.12`) forces Google Maps to lookup the verified business location. This ensures:
> 1. The user lands directly on the official business listing page with active contact phone numbers and reviews.
> 2. The routing engine guides the vehicle to the **actual main gate / parking entrance** instead of a random coordinate fence line.
