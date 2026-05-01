# 🍔 Smart Food Delivery Optimizer

A full-stack web application that optimizes food delivery routing using **Dijkstra's shortest-path algorithm** implemented in C++, with a **Node.js/Express** backend and **EJS** templating for three role-based dashboards.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [How It Works](#how-it-works)
- [API Reference](#api-reference)
- [Order Status Flow](#order-status-flow)
- [Known Issues & Fixes](#known-issues--fixes)
- [Roadmap](#roadmap)

---

## Overview

The Smart Food Delivery Optimizer manages a restaurant's delivery operations across a **10-node weighted graph** representing delivery locations. When a customer places an order, the system:

1. Checks the availability of each delivery person in real time
2. Passes their current positions to a C++ Dijkstra binary
3. Assigns the order to the nearest available delivery person
4. Tracks the delivery from assignment through to completion
5. Updates position state so every future order benefits from accurate routing

The system supports up to **2 concurrent delivery personnel** with automatic order queuing when both are busy.

---

## Features

- 🗺️ **Dijkstra route optimization** — C++ binary computes shortest paths across a weighted graph
- 🔄 **Real-time status tracking** — REST endpoint + auto-refreshing dashboard (3s interval)
- 🚦 **Availability-aware assignment** — prevents the same delivery person being assigned multiple simultaneous orders
- ⏳ **Order queuing** — orders placed when both personnel are busy are automatically assigned when someone becomes free
- 📍 **Position persistence** — delivery positions update after each delivery so routes are always calculated from the correct location
- 👥 **Three role dashboards** — separate views for Customers, Restaurant staff, and Delivery personnel

---

## Tech Stack

| Layer | Technology |
|---|---|
| Web Server | Node.js + Express 5 |
| Templating | EJS 3.1 |
| Route Algorithm | C++ (Dijkstra) |
| Module System | ES Modules (`type: module`) |
| Graph Model | 10-node weighted adjacency graph |

---

## Project Structure

```
smart-food-delivery-optimiser/
├── src/
│   ├── app.js            # Express server — routes, order logic, C++ spawning
│   ├── DELIVERY.js       # Delivery person state management
│   └── delivery.cpp      # Dijkstra algorithm (compile → delivery.exe)
├── views/
│   └── operations/
│       ├── customer.ejs  # Customer order placement UI
│       ├── delivery.ejs  # Delivery dashboard (real-time status)
│       └── restaurant.ejs # Restaurant order prep view
├── public/               # Static assets
├── package.json
└── README.md
```

---

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) v18+
- A C++ compiler (`g++` via GCC or MinGW on Windows)

### 1. Clone & Install

```bash
git clone <your-repo-url>
cd smart-food-delivery-optimiser
npm install
```

### 2. Compile the C++ Algorithm

```bash
# Linux / macOS
g++ -O2 -o src/delivery src/delivery.cpp

# Windows
g++ -O2 -o src/delivery.exe src/delivery.cpp
```

### 3. Start the Server

```bash
node src/app.js
```

### 4. Open the App

| Role | URL |
|---|---|
| Customer | http://localhost:3000/customer |
| Restaurant | http://localhost:3000/restaurant |
| Delivery | http://localhost:3000/delivery |
| Status API | http://localhost:3000/delivery/status |

---

## How It Works

### Delivery Assignment

```
Customer places order (location: 7)
         │
         ▼
Check availability of Guy 1 and Guy 2
         │
   ┌─────┴──────┐
   │            │
Both busy    At least one available
   │            │
   ▼            ▼
Queue order   Send to C++ Dijkstra:
(Waiting)     [customerLoc, guy1Pos or -1, guy2Pos or -1]
                   │
                   ▼
            C++ returns: { assigned, distance, path }
                   │
                   ▼
         Mark assigned person as in_delivery
         Store order with status "Assigned"
```

### After Delivery Completes

```
Delivery person marks order as Delivered
         │
         ▼
Update position → delivery location   ✅
Set status       → "idle"             ✅
Clear currentOrderId → null           ✅
         │
         ▼
Auto-assign any queued "Waiting" orders
```

### Delivery Person State Object

```javascript
{
  position: 0,              // Current graph node (0–9)
  status: "idle",           // "idle" | "in_delivery"
  currentOrderId: null,     // UUID of active order
  lastDeliveryLocation: 0   // Last delivery node
}
```

---

## API Reference

### `GET /delivery/status`

Returns the real-time state of both delivery personnel.

**Response:**
```json
{
  "guy1": {
    "position": 5,
    "status": "in_delivery",
    "currentOrderId": "order-uuid-123"
  },
  "guy2": {
    "position": 0,
    "status": "idle",
    "currentOrderId": null
  }
}
```

### `POST /customer/add`

Places a new order.

**Body:**
```json
{
  "location": 7,
  "items": ["burger", "fries"]
}
```

**Responses:**

| Code | Meaning |
|---|---|
| `200 OK` | Order assigned to a delivery person |
| `202 Accepted` | Both busy — order queued with status `Waiting` |

### `POST /delivery/delivered`

Marks an order as delivered. Updates delivery person position and status.

**Body:**
```json
{ "id": "order-uuid-123" }
```

---

## Order Status Flow

```
Placed → Assigned → Ready → Delivered
           ↑
        Waiting (queued when both busy, auto-assigned when free)
```

| Status | Meaning |
|---|---|
| `Placed` | Order received |
| `Waiting` | Both delivery persons busy; order queued |
| `Assigned` | Delivery person confirmed via Dijkstra |
| `Ready` | Restaurant has prepared the order |
| `Delivered` | Order delivered; position & status updated |

---

## Known Issues & Fixes

All 5 critical bugs identified in the initial codebase have been resolved:

| # | Bug | Fix |
|---|---|---|
| 1 | Position never updated (case sensitivity: `"Guy1"` vs `"guy1"`) | Added `.toLowerCase()` normalization in `DELIVERY.js` |
| 2 | Same delivery person assigned to multiple orders simultaneously | Added `status` field + availability check before Dijkstra |
| 3 | Position always stuck at 0 (restaurant) | Upgraded from plain `number` to full state object |
| 4 | Dijkstra assigned to busy delivery person | Busy persons now passed as `-1` to C++; algorithm skips them |
| 5 | No way to observe delivery state | Added `/delivery/status` endpoint + real-time dashboard |

---

## Roadmap

### Before Production
- [ ] Database persistence (positions/orders reset on server restart)
- [ ] Authentication on role-specific routes
- [ ] Structured error logging (e.g. Winston)
- [ ] Rate limiting on `POST /customer/add`
- [ ] Load testing with concurrent orders

### Feature Enhancements
- [ ] WebSocket real-time updates (replace 3s polling)
- [ ] Accurate ETA based on graph edge weights
- [ ] Multi-stop route optimization per delivery trip
- [ ] Return trip included in Dijkstra cost calculation
- [ ] Order cancellation window

### Nice to Have
- [ ] Analytics dashboard (avg delivery time, orders/hour)
- [ ] Customer-facing live tracking page
- [ ] Priority queue for VIP or time-sensitive orders
- [ ] Mobile app for delivery personnel

---

## License

ISC
