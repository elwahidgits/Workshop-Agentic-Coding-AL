# Agentic Coding for AL — Workshop Starter Repo

Starter repository for the **Agentic Coding for AL** workshop.

Workshop materials: [training.katson.com/agentic-coding-al](https://training.katson.com/agentic-coding-al)

---

## Branches

| Branch | Purpose |
|---|---|
| `main` | Empty starter — clone this when you begin |
| `ready` | Completed implementation for the reference |

---

## Business case

A car rental company wants to run its operations in Business Central:

- **Fleet management** — maintain a register of cars available for rent
- **Booking & availability** — manage future bookings with a visual calendar so staff can instantly see which cars are free and avoid double-booking
- **Pricing & Billing** — charge customers per day of rental; prices are effective on the rental start date, not the booking date; apply duration discounts (7 days → 10 %, 30 days → 20 %); bill long-term customers every month and short-term customers on car return
- **Customer records** — store driver's licence and passport details per customer
- **Pickup & return protocols** — guide employees through a structured handover checklist, attach photos and identity documents, and generate a printed rental agreement
- **Security deposits** — collect a refundable deposit (20 % of the total booking value) at pickup and return it in cash on car return
- **Live car tracking** — see the real-time location of all cars on a map directly inside Business Central, fed from a third-party GPS service

---

## Local development environment

Run the AL-Go local dev environment script to spin up a Business Central Docker container, compile all apps and configure `launch.json`:

```powershell
.\.AL-Go\localDevEnv.ps1
```

Requires Docker Desktop running Windows containers. See `.AL-Go/localDevEnv.ps1` for all available parameters.# Workshop-Agentic-Coding-AL
# Workshop-Agentic-Coding-AL
