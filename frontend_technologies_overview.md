# Frontend Technologies & Features Overview

This document provides a beginner-friendly overview of the frontend technologies, languages, and features used to build the user interface of Project Sahyadri 2.0.

---

## 1. Core Programming Languages

The user interface (UI) is built using three foundational web technologies:

* **JavaScript (JSX):** The interactive "brain" of the application. It dynamically loads data from the database, responds to user clicks, and exchanges messages with the Python backend. Files use a `.jsx` extension, which allows writing HTML-like structure directly inside JavaScript code.
* **HTML (HyperText Markup Language):** The "skeleton" of the page. It defines the structure and layout (e.g., input forms, sliders, button elements, and image containers).
* **CSS (Tailwind CSS v4):** The styling and design layer. We use **Tailwind CSS**, which provides utility classes. Instead of writing custom style sheets, design parameters (like colors, margins, text sizes, and glassmorphism styling) are applied directly to elements.

---

## 2. Key Frameworks & Libraries (The "Building Blocks")

To accelerate development and create a smooth user experience, several frameworks and libraries are integrated:

* **React 19 (UI Component Library):** 
  React splits the interface into self-contained, reusable blocks called **Components** (like modular widgets). 
  * *Specialty:* It allows only specific sections of a page (like the Chatbox, Weather cards, or Warehouse panels) to update when new data arrives, avoiding slow, full-page browser reloads.
* **Vite 8 (Build Tool):**
  A high-speed bundler that compiles, optimizes, and compresses all frontend code files so that the application loads instantly in a farmer's web browser.
* **Recharts (Interactive Graphs):**
  A specialized graphing library used to draw price forecasts and historical market price graphs. It supports smooth transitions, tooltips on hover, and updates dynamically.
* **Lucide React (Icons):**
  A vector icon pack providing crisp, scalable icons (such as suns, rain clouds, shields, and shopping carts) that adjust nicely to mobile and desktop screens.

---

## 3. Key Frontend Specialties & Features

* **Asynchronous API Integration:** The UI communicates with the FastAPI backend in real-time via `fetch` calls. Selecting a new district or commodity filter recalculates spatial markers and forecast projections instantaneously without refreshing the tab.
* **Modern Glassmorphism UI:** Features a sleek, dark-themed background with translucent elements, backdrop filters, glowing states, and micro-animations to deliver a modern, premium feel.
* **Real-time Markdown Rendering:** The agricultural guides returned by the Gemini AI agent are written in Markdown. React parses this formatting on the fly into clean HTML paragraphs, headers, and lists.
* **Instant Client-side Translation:** Standard UI headings, selectors, and warnings translate instantly between **English, Hindi, and Marathi** in the browser using a static dictionary lookup ([`translations.js`](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/frontend/src/utils/translations.js)).
