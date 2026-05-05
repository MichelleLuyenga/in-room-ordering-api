# In-Room Ordering API: Unified F&B Transaction Layer

[![Status: Demo](https://img.shields.io/badge/Status-Demo-gold)](#)
[![Auth: JWT Scoped](https://img.shields.io/badge/Auth-JWT%20Reservation%20Scoped-blue)](#)
[![Integration: POS Simulation](https://img.shields.io/badge/Integration-POS%20Simulation-black)](#)

## 🍽️ Project Overview
A robust REST API designed to handle the full lifecycle of a room service order—from the guest's mobile device to the kitchen display system (KDS) and eventually to the guest's folio charge.

### Core Logic:
*   **Unified API Layers:** Separate endpoints for Guests (ordering), Kitchen (status management), and Billing (folio posting).
*   **POS Abstraction:** Designed to normalize disparate order models (Item-first vs. Check-first) found in major POS systems like Simphony or Lightspeed.

## 🛡️ Security & Reliability
*   **Reservation-Scoped JWT:** Guests authenticate via their booking reference rather than an account, ensuring that orders are always tied to a verified stay without requiring guest registration.
*   **Resilient Webhooks:** Implemented a retry queue with exponential backoff and a dead-letter log for kitchen status updates to handle unstable network environments.

## 🛠️ Technical Stack
*   **Backend:** Node.js & PostgreSQL
*   **Authentication:** JWT (JSON Web Tokens)
*   **Events:** Webhook delivery system

---
**Role:** Hospitality Technology Engineer | [michellevision.com](https://michellevision.com)
