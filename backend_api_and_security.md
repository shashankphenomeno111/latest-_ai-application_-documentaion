# Backend Architecture, API Endpoints & Security Shield

This document explains the technical implementation details of the FastAPI backend application server of Project Sahyadri 2.0.

---

## 1. Core Server Framework

The backend is built using **FastAPI**, an asynchronous, high-performance web framework, and run via the **Uvicorn** ASGI server.

### Dual Protocol Hosting (HTTP/HTTPS)
During startup, the server scans for local SSL certificate paths:
* `sahyadri.crt` (SSL Certificate)
* `sahyadri.key` (Private Key)

If these keys are found, Uvicorn initializes in **HTTPS mode** on port `8000` (or the configured `PORT` env variable) to secure all client-server communication using SSL/TLS. If the certificate keys are missing, the server defaults to standard HTTP.

---

## 2. Production Security & Middleware Architecture

The backend contains several layers of security to prevent DDoS attacks, rate exhaustion, and information exposure.

```
       [ Client Request ]
               |
               v
  +-------------------------+
  |  Trace ID & Logging     | <-- Injects X-Request-ID, starts stopwatch
  +------------+------------+
               |
               v
  +-------------------------+
  |  SlowAPI Rate Limiter   | <-- Monitors client IP query counts via Redis
  +------------+------------+
               |
               v
  +-------------------------+
  |  JWT Authentication     | <-- Verifies user identity for secured endpoints
  +------------+------------+
               |
               v
  +-------------------------+
  |  Error Masking Shield   | <-- Catches system crashes, hides debug stacktraces
  +------------+------------+
               |
               v
     [ FastAPI Route Handler ]
```

### A. Rate Limiting (SlowAPI)
To prevent brute-forcing and endpoint abuse, the server utilizes **SlowAPI** linked to a **Redis cache cluster** (falling back to in-memory tracking if Redis is unreachable). 
Different endpoints have distinct rate limits:
* `/api/health`: 10 requests / minute
* `/api/chat`: 15 requests / minute
* `/api/tts`: 20 requests / minute
* `/api/forecast`: 30 requests / minute
* `/api/upload-image`: 5 requests / minute

### B. Access Authorization (JWT Authentication)
Secured endpoints (like `/api/chat` and `/api/upload-image`) are guarded by JSON Web Token (JWT) verification. The endpoint uses FastAPI's `Depends(get_current_user)` to validate the token in the request header and fetch the corresponding user object.

### C. Request Tracing & Telemetry Middleware
An asynchronous HTTP middleware wraps every incoming query:
1. **Trace ID Injection:** Checks if the request contains an `X-Request-ID` header. If missing, it generates a unique UUID tracker. This ID is attached to all logs created during that query cycle.
2. **Process Timer:** Tracks execution duration and returns it in the response header as `X-Process-Time`.
3. **Server Masking:** Modifies the standard HTTP header response to return `Server: Sahyadri Secure Shield`. This hides the fact that the backend is running FastAPI/Uvicorn, protecting it from signature-specific scanning bots.
4. **Unhandled Exception Masking:** In production, if a database connection fails or a runtime crash occurs, the middleware catches the exception and overrides the default FastAPI debug page. Instead, it returns a masked generic error message: *"An internal server error occurred. Please contact system administrators."* along with the `request_id` to prevent code exposure.

---

## 3. Operational API Catalog

The server exposes 8 key endpoints to power the React dashboard and AI chatbot:

### 1. System Diagnostics (`GET /api/health`)
Checks the status of the entire infrastructure stack. It probes:
* The PostgreSQL/SQLite connection pool.
* Redis availability (running a `.ping()`).
* The PyTorch TFT model check (verifying if weights are loaded or if it's running in heuristic fallback mode).

### 2. Multi-Language Helper (`GET /api/suggest`)
Retrieves lists of all commodities and districts registered in the database. 
* **Dynamic Translation:** If the query specifies Marathi (`lang=mr`) or Hindi (`lang=hi`), the backend translates all strings on-the-fly using a batch translation utility, returning localized label pairs.

### 3. Price Forecasting Engine (`GET /api/forecast`)
Retrieves forecast projections.
* **Logic:** Resolves the latest date in the database for the given commodity and district, then requests price forecasts for the next 4 weeks from [forecast_agent.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/backend/app/agents/forecast_agent.py).

### 4. Spatial Infrastructure (`GET /api/spatial`)
Exposes locations of physical assets in a given district:
* **Logic:** Returns lists of warehouses, DCCB cooperative bank branches, KVK stations, and soil testing labs. If the request contains the user's GPS coordinates (`latitude`, `longitude`), it uses spatial calculations to sort assets by distance.

### 5. Strategy Recommendation Engine (`GET /api/recommend`)
Calculates the optimal financial strategy for post-harvest sales.
* **Logic:** Pulls the 4-week price predictions from the forecasting module, checks distance and storage rates of local warehouses, checks DCCB interest rates, and runs an optimization model to recommend the top-3 options (selling immediately vs. storing and selling later).

### 6. Conversational Agent Chatbot (`POST /api/chat`)
The central entry point for user dialog.
* **Logic:** Validates JWT access, receives the query message, and forwards it to the multi-agent router (`route_and_execute`).

### 7. Vision-Based Crop Diagnostics (`POST /api/upload-image`)
Enables image upload for crop leaves.
* **Logic:** Receives raw image bytes and mime types, and invokes Gemini Vision API under-the-hood (`diagnose_crop_image`) to detect diseases and return treatment remedies.

### 8. Voice Speech Streaming (`GET /api/tts`)
Enables text-to-speech feedback.
* **Logic:** Converts text inputs into localized speech audio streams (MP3) using `generate_tts_audio` and returns them as a chunked `StreamingResponse` for immediate web playback.

---

## 4. SPA Hosting Integration
To host the application as a unified package, the backend serves the compiled React Single-Page Application (SPA) static files:
* If compiled files exist under `frontend/dist/`, the server mounts the assets subdirectory.
* A catch-all route `/{full_path:path}` intercepts all non-API paths and serves `index.html`. This allows client-side routers (like React Router) to function correctly without raising 404 errors during page refreshes.
