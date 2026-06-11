# Roadmap

> Ordered so each feature is buildable on top of what came before. Update `status` as work progresses.

---

## 1. Fleet Foundation

**Delivers:** The `DAY` unit of measure and `Non-Inventory` Item type are in place so every subsequent car and booking can use them.

**Status:** `planned`

**How it's built:** Create a `Unit of Measure` record with code `DAY` and verify the `Non-Inventory` Item type is enabled — configuration only, no table extension needed ([tech-design §1.3](specs/tech-design.md#13-implementation-decision)).

**Done when:** A user can create an Item, set `Type = Non-Inventory`, and select `DAY` as Base Unit of Measure without error.

---

## 2. Car Register

**Delivers:** Each rental car is a searchable Item card carrying vehicle-specific fields (licence plate, VIN, make, model, year, mileage, category) alongside a photo and Fixed Asset link.

**Status:** `planned`

**How it's built:** `tableextension` on `Item` for vehicle fields; `pageextension` on `Item Card` surfacing a *Vehicle* group, Document Attachments FactBox, and Fixed Asset link ([tech-design §1.3](specs/tech-design.md#13-implementation-decision)).

**Done when:** A user can open a car's Item Card, fill in all vehicle fields, attach a photo, and see the linked Fixed Asset.

---

## 3. Customer Identity

**Delivers:** Customers carry driver's licence and passport numbers with expiry dates, plus attached scanned copies.

**Status:** `planned`

**How it's built:** `tableextension` on `Customer` with identity fields classified `EndUserIdentifiableInformation`; `pageextension` on `Customer Card` with an *Identification* group and Document Attachments FactBox ([tech-design §4.3](specs/tech-design.md#43-implementation-decision)).

**Done when:** A user can open a customer, enter licence/passport fields, and attach scanned documents to the customer record.

---

## 4. Rental Setup

**Delivers:** Admins can configure deposit percentage, cash bank account, forfeit G/L account, and billing defaults in one place without touching code.

**Status:** `planned`

**How it's built:** Small setup table (or extension on `Sales & Receivables Setup`) with deposit %, Bank Account No., Forfeit G/L Account, and Bank Deposit description template ([tech-design §6.3](specs/tech-design.md#63-implementation-decision)).

**Done when:** A setup page opens, all fields are editable, and the values are readable by a test codeunit.

---

## 5. Pricing Configuration

**Delivers:** Daily rental rates and automatic duration discounts (10 % at 7 days, 20 % at 30 days) are applied by the standard Best Price engine without custom code.

**Status:** `planned`

**How it's built:** `Price List Line` rows on the car Item (or Item Category) with `Minimum Quantity = 7 / 30` and `Line Discount % = 10 / 20`; no AL pricing code ([tech-design §3.3](specs/tech-design.md#33-implementation-decision)).

**Done when:** Creating a Sales Order line for a car with quantity ≥ 7 automatically applies the 10 % discount, and ≥ 30 applies 20 %.

---

## 6. Booking Document

**Delivers:** Staff can create a rental booking as a Blanket Sales Order with a start and finish date; the system blocks double-booking for the same car.

**Status:** `planned`

**How it's built:** `tableextension` on `Sales Header` adds `Rental Finish Date` and `Next Billing Date`; event subscribers on `Sales Line.OnInsert/OnModify` run the availability overlap query and error on conflict ([tech-design §2.3](specs/tech-design.md#23-implementation-decision)).

**Done when:** A Blanket Sales Order with a car and date range can be saved; a second overlapping booking for the same car is rejected with a clear error.

---

## 7. Fleet Booking Board

**Delivers:** A visual Gantt-style calendar shows all cars as rows and their booked periods as coloured bars so staff can instantly see what is free.

**Status:** `planned`

**How it's built:** Custom `ListPlus` page sourced from `Item` (filtered `Non-Inventory`) with a JavaScript control add-in rendering a date timeline; data fed from the same Sales Line availability query used by the double-booking guard ([tech-design §2.3](specs/tech-design.md#23-implementation-decision)).

**Done when:** Opening the Fleet Booking Board shows all cars, renders existing bookings as bars, and double-clicking a bar opens the underlying Blanket Sales Order.

---

## 8. Rental Agreement Report

**Delivers:** A printable rental agreement document can be generated from a Blanket Sales Order, ready for the customer to sign at pickup.

**Status:** `planned`

**How it's built:** New report bound to `Sales Header` with a Word/RDLC layout using the standard report runtime — no custom posting ([tech-design §5.3](specs/tech-design.md#53-implementation-decision)).

**Done when:** Running the report from a Blanket Sales Order produces a correctly formatted agreement PDF containing customer, car, and date details.

---

## 9. Pickup Wizard

**Delivers:** An employee is guided through a step-by-step pickup checklist (mileage, fuel, photos, ID verification), the first cycle invoice is created, and the security deposit is collected — all from one flow.

**Status:** `planned`

**How it's built:** Custom wizard page launched from the Blanket Sales Order; persists checklist data to the `Sales Header` extension fields; invokes codeunit 87 to create the first period Sales Order; programmatically creates and posts a Bank Deposit for the deposit amount; prints the rental agreement ([tech-design §5.3](specs/tech-design.md#53-implementation-decision), [§6.3](specs/tech-design.md#63-implementation-decision)).

**Done when:** Completing the wizard sets `Pickup Completed = true`, a released Sales Order exists for the first billing period, a posted Bank Deposit exists for 20 % of the booking value, and `Next Billing Date` is set to rental start + 30 days.

---

## 10. Billing Scheduler

**Delivers:** Long-term rentals are invoiced automatically every 30 days without manual intervention.

**Status:** `planned`

**How it's built:** Custom codeunit registered as a daily `Job Queue Entry`; filters open Blanket Orders where `Next Billing Date ≤ Today`; calls codeunit 87 to produce the next cycle Sales Order with the correct Posting Date; advances `Next Billing Date` by 30 days ([tech-design §3.3](specs/tech-design.md#33-implementation-decision)).

**Done when:** After the Job Queue runs, a Blanket Order past its `Next Billing Date` has a new released Sales Order covering the next 30-day period and `Next Billing Date` is advanced.

---

## 11. Return Wizard

**Delivers:** An employee is guided through the return checklist; the final invoice (including any extra charges) is created and posted, and the deposit is refunded or partially forfeited.

**Status:** `planned`

**How it's built:** Custom wizard page launched from the Blanket Sales Order; persists return data to the `Sales Header` extension; creates the final Sales Order via codeunit 87 with extra-charge lines; posts a Bank Deposit refund and optionally adds a forfeit Sales Line; closes the Blanket Order ([tech-design §5.3](specs/tech-design.md#53-implementation-decision), [§6.3](specs/tech-design.md#63-implementation-decision)).

**Done when:** Completing the wizard sets `Return Completed = true`, a posted Sales Invoice exists for all unbilled days plus any extra charges, the deposit is refunded (or the forfeited amount appears on the invoice), and the Blanket Order is closed.

---

## 12. Fleet Live Map

**Delivers:** A real-time map inside Business Central shows the current GPS location of every rented car, fed from a third-party provider.

**Status:** `planned`

**How it's built:** `Vehicle GPS Position` table (per-Item lat/long/speed); `Fleet GPS Connector` codeunit polls the provider REST API via `HttpClient` on a Job Queue schedule with the API key in Isolated Storage; `Fleet Live Map` page embeds a Leaflet/Bing Maps JavaScript control add-in ([tech-design §7.3](specs/tech-design.md#73-implementation-decision)).

**Done when:** Opening the Fleet Live Map page shows a marker for each active car at its current GPS position, refreshed on the configured interval, with a click drilling to the Item Card.
