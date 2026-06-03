# API Communication & Security Architecture

This document details the communication flow between the React frontend client and the FastAPI backend server, and the security layers implemented to protect the Project Sahyadri 2.0 platform.

---

## 1. How API Communication Works

The React frontend and FastAPI backend communicate asynchronously over HTTP/HTTPS protocols using JSON payloads. 

### Step-by-Step Call Lifecycle

```
[ Frontend: Farmer Action ]
(e.g., Click 'Analyze' or slide Quantity)
           |
           v
[ State Compilation ]
- Converts localized input (e.g., 'सोयाबीन') to English DB key ('SOYABEAN')
- Checks if JWT Auth Token is present in localStorage
           |
           v
[ Client Dispatch (Fetch) ]
- Sends request: GET https://localhost:8000/api/recommend?commodity=SOYABEAN...
- Includes Authorization Header: "Bearer <token>" (for chat/upload)
           |
           v
[ Nginx Reverse Proxy Gateway ]
- Decrypts SSL/TLS wrapper, validates headers, and forwards to FastAPI
           |
           v
[ FastAPI Backend Processing ]
- Limiter verifies rate limits
- JWT verification dependency validates token
- Middleware injects X-Request-ID tracking header
- Router fetches DB results, runs TFT forecasting, and builds recommendations
           |
           v
[ JSON Return Payload ]
- Backend responds with status code 200 OK + JSON payload + Security Headers
           |
           v
[ Frontend Rendering ]
- React catches JSON, updates UI state variables, and triggers redraw of charts
```

---

## 2. Security Layers: What, Why, and the Logic

We implemented **four distinct security layers** to protect the application from common web vulnerabilities (OWASP Top 10), data theft, and resource exhaustion.

### Layer 1: Nginx SSL/TLS Encryption Gateway
* **What it is:** A reverse proxy gateway that acts as the entry point for all incoming web requests. It intercepts traffic on port 8000 (HTTP) and enforces secure port 443 (HTTPS) communication using `sahyadri.crt` and `sahyadri.key` certificates.
* **Why we did it:** To prevent **Man-in-the-Middle (MITM) attacks**. Without encryption, all data traveling between the farmer's browser and our server (including login passwords and JWT session keys) travels over public Wi-Fi or cellular networks in plain text. A hacker could easily intercept (sniff) these packets.
* **The Logic:** Nginx terminates the SSL/TLS wrapper. It encrypts the request data using public/private key cryptography, ensuring that only the client's browser and our server can read the message contents. It strictly enforces secure protocols (**TLS v1.2** and **TLS v1.3**).

### Layer 2: SlowAPI IP-Based Rate Limiter
* **What it is:** A rate-limiting middleware connected to a local Redis database (with memory fallback) that counts incoming requests per client IP address.
* **Why we did it:** To prevent **Denial of Service (DoS/DDoS) attacks** and bot scrapers. Since calling the deep learning TFT model and the Gemini AI API requires CPU/GPU resources and costs money per token, a malicious script calling these endpoints millions of times could freeze our servers or rack up massive API bills.
* **The Logic:** Every request is logged under the user's IP. If an IP exceeds the endpoint thresholds (e.g., more than 15 chat queries or 5 image uploads per minute), the rate limiter blocks execution and returns a `429 Too Many Requests` status code, preserving server resources.

### Layer 3: JSON Web Token (JWT) Authentication
* **What it is:** A token-based user authorization mechanism. Secured APIs (like chatbot conversation and vision diagnosis) require a valid token to be passed in the `Authorization: Bearer <JWT_TOKEN>` header.
* **Why we did it:** To protect **user privacy** and restrict paid AI features to verified users. Without this, anyone could call our `/api/chat` or `/api/upload-image` endpoints directly using command-line tools like curl, using up our Gemini token quotas.
* **The Logic:**
  1. On login, the server encrypts the user's ID and session details using a secret key (symmetric signature) and returns a signed token to the browser.
  2. The browser saves this token in `localStorage`.
  3. For secure requests, the browser attaches the token.
  4. The backend decrypts the token using the secret key. If the signature is valid and has not expired, access is granted.

### Layer 4: HTTP Request Tracing & Server Masking
* **What it is:** An asynchronous custom middleware in FastAPI that tracks processing time, intercepts uncaught exceptions, and overwrites standard server headers.
* **Why we did it:** To prevent **Information Disclosure** and aid in **System Auditing/Debugging**. If a database query fails and the server crashes, standard frameworks print a detailed stacktrace on the screen, showing database schemas, directories, and code snippets. Hackers use these to plan SQL injection. Masking prevents this.
* **The Logic:**
  * **X-Request-ID:** Every query is tagged with a unique UUID trace token. This allows administrators to search backend logs and trace the exact timeline of a single user action.
  * **Server Header Masking:** Standard responses return `Server: Uvicorn` or `X-Powered-By: FastAPI`. We override this to return `Server: Sahyadri Secure Shield` to hide our underlying technology stack.
  * **Crash Override:** If a code crash occurs, the middleware intercepts the 500 error page and replaces it with a generic, safe response: *"An internal server error occurred."* along with the `X-Request-ID` for tracking.
