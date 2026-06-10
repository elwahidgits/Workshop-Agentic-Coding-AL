# Architecture

> Reflects the current state of the system. Updated when a feature branch merges into main.

This document maps every requirement from `specs/brief.md` to standard Business Central (BC) functionality first, identifies the residual gaps, and records the implementation approach we will follow. Standard BC source code references are paths inside `external/MSDyn365BC` (the Microsoft public source mirror); Microsoft Learn URLs are cited where they document end-user behaviour.

The cross-cutting design intent is: **a rental car is a `Non-Inventory Item` with `Base Unit of Measure = DAY`**, **a booking is a standard `Blanket Sales Order` with one Item line per car**, **billing produces a separate `Sales Order` per 30-day period plus a final Sales Order at return**, **pricing & duration discounts live in standard `Price List Line` records**, **deposits are posted through the standard Bank Deposits feature**, and we extend (rather than replace) standard Customer, Sales Header and Sales Line objects. Custom tables/pages are introduced only where standard BC has no usable building block.

---

## 1. Fleet management — register of cars available for rent

### 1.1 What standard BC already provides

- **`Item` master record** (`Microsoft.Inventory.Item.Item`, table 27) supports `Type = Non-Inventory` via `enum 27 "Item Type"` value `2 "Non-Inventory"`. Evidence: `external/MSDyn365BC/Base Application/Inventory/Item/Item.Table.al` line 206 (`field(10; Type; Enum "Item Type")`) and `Base Application/Inventory/Item/ItemType.Enum.al` lines 7–14.
- **`Base Unit of Measure`** (field 8 of `Item`, see `Item.Table.al` line 147) lets us define the car's pricing/quantity unit; we register a `Unit of Measure` with code `DAY` and use it as the Base UoM, so every Sales Line quantity is "rental days" without conversion.
- **`Item Card`** and **`Item List`** pages give an out-of-the-box editable register with statistics, units of measure, prices, dimensions, posting groups, picture, and no.-series. Evidence: `Base Application/Inventory/Item/ItemCard.Page.al`, `ItemList.Page.al`.
- **`Item Category`** (table 5722, `Base Application/Inventory/Item/ItemCategory.Table.al`) lets us segment the fleet (e.g., *Compact*, *SUV*, *Van*) for pricing, reporting and category-level price-list rows.
- **`Fixed Asset`** (table 5600 in `Base Application/FixedAssets/...`) is the right place for the financial side of the vehicle (acquisition, depreciation, disposal). The Item is linked to a Fixed Asset via dimensions or a shortcut field on the table extension.
- The **standard Item `Picture`** field (a `Media` field) holds the primary vehicle photo at no extra cost.
- Microsoft Learn documents the Non-Inventory item type and its use for things you sell but do not stock: <https://learn.microsoft.com/dynamics365/business-central/inventory-how-register-new-items>.

### 1.2 Gaps remaining

- The `Item` table has no native fields for vehicle-specific master data: licence-plate number, VIN, make, model, year, current mileage, fuel type, transmission, insurance policy, MOT/inspection due date, vehicle category. None of the standard fields cleanly carry this semantics.
- There is no built-in "current mileage" cue.

### 1.3 Implementation decision

- Use the standard **`Item` table with `Type = Non-Inventory`** as the car master record. Each car is one Item; the Item `No.` is the internal car ID and `Description` is the friendly name. `Type = Non-Inventory` is correct because we never hold inventory of a specific licence plate — the unit is "rental days" of that one car.
- Define a `Unit of Measure` with code **`DAY`** and set it as **`Base Unit of Measure`** on every car item. All Sales Lines use this UoM; quantity literally equals rental days.
- Use **`Item Category`** for category (Compact, SUV, Van, …); avoid creating a parallel "Car Category" table. Category-level prices and discounts attach via `Asset Type = Item Category` on the Price List (see section 3).
- Add a **`tableextension` on `Item`** with the small set of vehicle fields the brief implies (licence plate, VIN, make, model, year, current mileage, fuel type, transmission, next inspection date). Mark identifiers as `DataClassification = CustomerContent`.
- Add a **`pageextension` on `Item Card`** that surfaces those fields in a *"Vehicle"* group, plus a `FactBox` link to the related `Fixed Asset` and to attached photos via the standard **Document Attachments FactBox**.
- The standard Item `Picture` field holds the primary vehicle photo at no extra cost.
- A separate booking/pricing/availability data model is **not** introduced — those concerns are handled by the standard Sales and Pricing modules (sections 2 and 3 below).

---

## 2. Booking & availability — future bookings with a visual calendar

### 2.1 What standard BC already provides

- **Items can be sold on Sales documents.** `Sales Line.Type = Item` is the default and is supported on every `Sales Header.Document Type`, including `Blanket Order`. Evidence: `Base Application/Sales/Document/SalesLine.Table.al` `Type` field; `Base Application/Sales/Document/SalesHeader.Table.al` `Document Type` enum value `"Blanket Order"`.
- **`Blanket Sales Order`** is a standard document type representing a long-term agreement that ships partial Sales Orders over time. Microsoft Learn: <https://learn.microsoft.com/dynamics365/business-central/sales-how-blanket-orders>. The standard handoff is the **`Blanket Sales Order to Order`** codeunit (codeunit 87, `Base Application/Sales/Document/BlanketSalesOrdertoOrder.Codeunit.al`) and its YN wrapper (`BlnktSalesOrdtoOrdYN.Codeunit.al`), which copy lines and reduce the outstanding quantity on the Blanket.
- **`Sales Header.Requested Delivery Date`** (field 5790, `SalesHeader.Table.al` line 3432) is a standard date already on every Sales Header, including Blanket Orders. Semantically it is exactly "the date the customer wants the goods", which for a rental is the pickup date.
- **`Sales Quote → Sales Order`** conversion is also standard for the short-circuit case where a tentative reservation is wanted before signing the agreement.
- The brief's *availability* requirement reduces to "for a given date range, which cars are not on any open booking?" — which is a query against open `Sales Line` rows joined to their `Sales Header`. No reservation ledger is required.
- Microsoft Learn confirms Blanket Order behaviour and the partial-order pattern: <https://learn.microsoft.com/dynamics365/business-central/sales-how-blanket-orders>.

### 2.2 Gaps remaining

- A Blanket Sales Order has no native concept of a "rental finish" date. `Requested Delivery Date` covers the start, but the end of the rental window must be added.
- There is no native double-booking guard for Items on overlapping Sales documents.
- The standard **Blanket Sales Orders** list is a grid, not a colour-coded Gantt/calendar.
- No standard Gantt control add-in ships with BC.

### 2.3 Implementation decision

- **Booking document = standard `Blanket Sales Order`** (`Sales Header.Document Type::"Blanket Order"`) with a single `Item` Sales Line per car. We do **not** create a custom "Rental Header" table.
- **Rental Start Date = `Sales Header.Requested Delivery Date`** (existing field 5790). No new "start" field; we just reuse the standard field. The Sales Header `Order Date` is also aligned to `Requested Delivery Date` so the standard best-price resolver returns the price effective on pickup day (see section 3).
- **Rental Finish Date = a new custom field** on a `tableextension` of `Sales Header`. The extension also carries `Next Billing Date` (used by the billing-cycle scheduler in section 3) and the pickup/return capture fields described in section 5.
- The Sales Line `Quantity` on the Blanket Order is initialised to `Rental Finish Date − Requested Delivery Date` (in days, in the `DAY` UoM). It is the *agreed total* for the rental; partial Sales Orders draw it down as billing cycles are produced (standard Blanket-to-Order behaviour).
- **Availability** is queried directly from open Sales Lines:
  - For a given car (`Item No.`) and a date window `[from, to]`, the query selects `Sales Line` where `Type = Item`, `No.` matches, joined to `Sales Header` where `Document Type ∈ {Blanket Order, Order, Quote}`, `Status` is not closed, and `[Requested Delivery Date, Rental Finish Date]` overlaps `[from, to]`.
  - **No separate availability table**, no `Res. Capacity Entry` writes, no parallel reservation ledger. The standard Sales tables are the single source of truth.
- **Double-booking guard** is an event subscriber on `Sales Line.OnInsert`/`OnModify` and on `Sales Header.OnValidate` for `Requested Delivery Date` and `Rental Finish Date`. It re-runs the availability query above for the affected Item and date range and errors on overlap.
- **Custom actions on the Blanket Sales Order** (page extension on `Blanket Sales Order` page 507) — three buttons that drive the rental lifecycle, all detailed in their respective sections:
  - **Pickup wizard** — section 5; creates and releases the *first* period Sales Order, sets `Next Billing Date`, posts the security deposit (section 6).
  - **Prolong action** — section 5; updates `Rental Finish Date`, increases the Blanket line quantity, and creates an additional period Sales Order if the prolongation crosses a billing-cycle boundary.
  - **Return wizard** — section 5; creates the *final* Sales Order for unbilled days plus extra charges and refunds/forfeits the deposit.
- **Visual calendar (Fleet Booking Board)**: ship a custom AL **`ListPlus` page** sourced from `Item` filtered to `Type = Non-Inventory` (rows = cars) with a **JavaScript control add-in** that renders a horizontal date timeline and overlays open Blanket Order rental periods. The page reads its data from the same Sales Line query that powers availability — so the calendar and the double-booking guard share one source of truth. The page is read-only; double-clicks drill to the underlying Blanket Sales Order. We acknowledge this is custom UI; standard BC does not have a Gantt and the brief explicitly demands one.

---

## 3. Pricing & Billing

### 3.1 What standard BC already provides

- **Price List / Price List Line** (`Base Application/Pricing/PriceList/PriceListHeader.Table.al`, `PriceListLine.Table.al`, table 7001) supports per-Item prices with `Starting Date`, `Ending Date`, `Minimum Quantity`, `Unit Price`, and `Line Discount %` fields, scoped by `Asset Type` (Item / Item Category / …) and `Source Type` (All Customers / Customer / Customer Price Group / …).
- **Best Price calculation** automatically resolves the lowest unit price and the highest line-discount percentage for a given Order Date and Quantity. Microsoft Learn: <https://learn.microsoft.com/dynamics365/business-central/sales-how-record-sales-price-discount-payment-agreements#best-price-calculation> — *"Is the order date within the starting and ending date of the price/discount agreement?"* and *"Is there a minimum quantity requirement that is fulfilled?"*.
- **`Blanket Sales Order to Order`** (codeunit 87, `Base Application/Sales/Document/BlanketSalesOrdertoOrder.Codeunit.al`) is the standard mechanism that copies the relevant lines from a Blanket onto a fresh Sales Order, decrementing the outstanding quantity on the Blanket. We invoke it programmatically per billing cycle.
- **`Sales-Post`** (codeunit 80) posts the resulting Sales Order to a Sales Invoice and Customer Ledger Entry.
- **Job Queue** (`Job Queue Entry`, table 472) schedules background tasks on a daily cadence — used here to produce period invoices automatically.

### 3.2 Gaps remaining

- The brief's **duration discount table (≥7 days → 10%, ≥30 days → 20%)** is a perfect fit for `Price List Line.Minimum Quantity` + `Line Discount %`; there is no gap.
- The brief's **"prices effective on rental start date, not booking date"** is exactly how the best-price calculation works when the Sales Order's `Order Date` equals the rental pickup date; no gap.
- BC's Blanket-to-Order pattern is **manually triggered**. There is no native "produce one Sales Order every 30 days from this Blanket" scheduler, and no native rule for what the period invoice's posting date should be. We add this thin layer.
- BC has no native concept of a "final" return-time Sales Order distinct from a periodic one.

### 3.3 Implementation decision

- **Daily price** = `Item.Unit Price` on the car item, or `Price List Line` rows with `Asset Type = Item` (per-car) or `Asset Type = Item Category` (per-segment), scoped by `Source Type = All Customers` (or specific groups).
- **Duration discounts** = two `Price List Line` discount rows on the car item (or its `Item Category`):
  - `Minimum Quantity = 7`, `Line Discount % = 10`.
  - `Minimum Quantity = 30`, `Line Discount % = 20`.
  No custom code; the Best Price engine picks the highest discount whose `Minimum Quantity` is satisfied for that period's Sales Order quantity.
- **Effective-date pricing** uses the standard best-price engine. Each period Sales Order created from the Blanket has its `Order Date` set so the price effective on the rental start day is the one that applies to every cycle of this rental (the rental's contracted price does not move when the calendar rolls over a price change).
- **Billing model = one Sales Order per 30-day cycle, plus a final Sales Order at return**, all created from the same Blanket Sales Order via codeunit 87:
  - **Posting Date for a cycle Sales Order** is computed by the rule:
    - if `(Rental Finish Date − Today) < 30 days` → `Posting Date := Rental Finish Date` (the rental is about to end inside this cycle, so the cycle is short and the cycle invoice closes the period);
    - else `Posting Date := Rental Start Date + 30 days × n` (where `n` is the cycle ordinal — i.e. the cycle bills exactly 30 days forward).
  - The cycle Sales Order's line quantity equals the days actually covered by that cycle.
  - This unifies short-term and long-term rentals: a 5-day rental produces a single final Sales Order on return; a 90-day rental produces two cycle Sales Orders (days 1–30, 31–60) plus a final on return.
- **Job Queue scheduler** — a daily Job Queue Entry runs a small custom codeunit that:
  1. Filters open Blanket Sales Orders where `Next Billing Date ≤ Today`.
  2. For each, calls codeunit 87 (`Blanket Sales Order to Order`) to create the next period Sales Order, applies the Posting Date rule above, and releases the order.
  3. Advances `Next Billing Date := Next Billing Date + 30 days` on the Blanket header.
  Standard `Sales-Post` is then invoked manually or by a separate posting Job Queue task — we do not duplicate posting logic.
- **First period Sales Order** is **not** created by the Job Queue. It is created by the **Pickup wizard** (section 5) at the moment of pickup, which also sets `Next Billing Date := Rental Start Date + 30 days`. This guarantees the customer gets billed on day 1 of the rental, not on the daily Job Queue's next run.
- **Final Sales Order** is **not** created by the Job Queue either. It is created by the **Return wizard** (section 5), which:
  - Computes the still-unbilled days from `Last Billed Date` (= the latest cycle's covered end) to the actual return date.
  - Adds extra-charge Sales Lines for mileage overage, damage, late return, etc. (these are normal Sales Lines on the same Sales Order, not a separate document).
  - Sets the Sales Order's Posting Date to the actual return date.
  This avoids any race between a same-day Job Queue cycle invoice and the return invoice.
- We **do not** use Subscription Billing, Service Contracts, or Recurring Sales Lines. The Blanket-to-Order pattern with the Job Queue scheduler is enough and stays inside the standard Sales module — no Premium licence dependency, no separate billing engine to learn, and one document trail per rental.

---

## 4. Customer records — driver's licence and passport

### 4.1 What standard BC already provides

- The **`Customer`** table (`Base Application/Sales/Customer/Customer.Table.al`, table 18) stores name, address, contact, payment terms, dimensions, and supports table extensions for new fields. It also already exposes `Registration Number` (field 25, `Text[50]`) for e.g. national tax registration — pattern proven for adding identifying numbers.
- **Customer Picture** is built in (`CustomerPicture.Page.al`).
- The **Document Attachment** subsystem (`Foundation/Attachment/DocumentAttachment.Table.al`, table 1173, with Pages `DocumentAttachmentDetails`, `DocAttachmentListFactbox`, codeunit `Document Attachment Mgmt`) is **table-agnostic** and can attach files (images, PDFs) to *any* table by `Table ID + No.`. Microsoft Learn: <https://learn.microsoft.com/dynamics365/business-central/ui-how-add-link-to-record>. Files can be stored in external file storage if needed (<https://learn.microsoft.com/dynamics365/business-central/across-store-document-attachments-externally>).
- **Sensitive data classification** is supported via `DataClassification = EndUserIdentifiableInformation` and the Data Classification framework.

### 4.2 Gaps remaining

- No native fields for `Driver's License No.`, `Driver's License Expiry`, `Passport No.`, `Passport Expiry`, `Date of Birth`. These are personal identifiers absent from a standard B2B Customer card.

### 4.3 Implementation decision

- Add a **`tableextension` on `Customer`** with the four/five identity fields above, all classified `EndUserIdentifiableInformation`. Add `OnValidate` checks for non-empty values and for *expiry > pickup date* (the latter enforced from the Pickup wizard, not on the Customer card itself).
- Add a **`pageextension` on `Customer Card`** that puts the new fields in a dedicated *"Identification"* group with tooltips and includes the standard **Document Attachments FactBox** so scanned licence and passport images are uploaded directly to the customer. No custom file table is needed.
- Use the existing Customer Picture for the customer photo if desired.

---

## 5. Pickup & return protocols — checklist, photos, identity copies, printed agreement

### 5.1 What standard BC already provides

- **Document Attachments** on Sales documents (table 1173) accept any file type including images, with a friendly FactBox attached to Blanket Sales Orders, Sales Orders, and Posted Sales Invoices via the standard `Document Attachment Mgmt` codeunit. Photos can be captured from a tablet/phone and attached.
- **Document printing** is the standard report mechanism. The Sales report layouts (`Standard Sales - Order Conf.`, report 1305, `Base Application/Sales/Document/StandardSalesOrderConf.Report.al`) use Word/RDLC layouts and are routinely customised into agreement-style documents bound to `Sales Header`.
- **Sales Document Status workflow** (`Sales Header.Status` enum: Open, Released) provides a controlled state machine; codeunit 87 plus the standard Release codeunit cover the "release first cycle order" step.
- **`Standard Sales Code`** (`Base Application/Sales/Document/StandardSalesCode.Table.al`) lets us preload a templated set of comment lines into every rental Blanket Order.
- BC does **not** ship a structured "checklist wizard" page generator; custom wizard pages remain the right pattern.

### 5.2 Gaps remaining

- No native multi-step pickup/return wizard with mandatory photo and ID-copy capture, mileage in/out, fuel level in/out, damage notes.
- No native "rental agreement" report layout.
- No native field on Sales Header to record pickup-time and return-time mileage, fuel level, condition notes.

### 5.3 Implementation decision

- **Capture data on the Blanket Sales Header**: the Sales Header `tableextension` from section 2 also carries the pickup/return data — `Pickup Mileage`, `Return Mileage`, `Pickup Fuel %`, `Return Fuel %`, `Pickup Condition Notes`, `Return Condition Notes`, `Pickup Completed`, `Return Completed`, plus `Last Billed Date` (for the return-time unbilled-days computation). Pickup and return both happen at the agreement (Blanket) level, not at any individual period invoice.
- **Capture files via standard Document Attachments on the Blanket**: vehicle photos at pickup and return, ID copies, signed agreement scan — all attached to the Blanket Sales Order via the existing FactBox. They survive even after period Sales Orders are posted and archived.
- **Pickup wizard** (custom page) launched from a button on the Blanket Sales Order:
  1. Walks pickup checklist (mileage, fuel, photos, ID verification — including expiry > today).
  2. Persists captured data to the Blanket Sales Header extension and Document Attachments.
  3. **Creates and auto-releases the first period Sales Order** by invoking codeunit 87 (`Blanket Sales Order to Order`) and the standard Release codeunit. Posting Date for that order is set per the rule in §3.3.
  4. **Sets `Next Billing Date := Rental Start Date + 30 days`** on the Blanket extension so the daily Job Queue picks it up correctly from cycle 2 onwards.
  5. **Posts the security deposit via the Bank Deposits feature** (see §6).
  6. Prints the rental agreement (standard report layout bound to the Blanket Sales Header).
- **Prolong action** (button on the Blanket Sales Order, not a wizard):
  1. Asks for the new `Rental Finish Date` and validates against availability (§2.3).
  2. Updates `Rental Finish Date` on the Blanket header and increases the Blanket line `Quantity` by the additional days.
  3. **If** the prolongation pushes the rental past the next `Next Billing Date` and there is more than ~30 days until the new finish, immediately creates an additional period Sales Order via codeunit 87 (otherwise the next cycle is left to the Job Queue or the Return wizard, whichever arrives first).
- **Return wizard** (custom page) launched from a button on the Blanket Sales Order:
  1. Walks return checklist (mileage, fuel, photos, condition).
  2. Persists captured data; assembles the **extra charges** as additional Sales Lines (mileage overage, fuel difference, damage, late return).
  3. **Creates the final Sales Order** for any still-unbilled days (`Last Billed Date` → return date) plus the extra-charge lines, via codeunit 87. Posting Date = actual return date.
  4. Auto-releases and (optionally, by setup flag) auto-posts the final Sales Order via standard `Sales-Post`.
  5. **Refunds or forfeits the deposit via the Bank Deposits feature** (see §6). Forfeited portions are added as a Sales Line on the final Sales Order so the revenue is recognised through the normal posting.
  6. Closes the Blanket Sales Order.
- **Templated checklist content** is loaded once into a `Standard Sales Code` of comment lines and inserted on every rental Blanket Order via the standard "Get Recurring Sales Lines" mechanism, so the instructions stay editable without code changes.
- **Rental agreement print** = a report extension (or a small new report bound to `Sales Header`) with a Word/RDLC layout matching the legal agreement template. Triggered from the Pickup wizard's final step ("Print & Sign"). Re-uses the standard report runtime, no custom posting.

---

## 6. Security deposits — 20% refundable, paid at pickup, returned in cash on return

### 6.1 What standard BC already provides

- **Bank Deposits** is the standard BC feature for registering money received (and refunded) against a bank account in a single document with one line per customer payment. The Base Application contains the navigation hooks (`Base Application/Bank/Deposit/Microsoft.Bank.Deposit.Namespace.al`, codeunits `Open Deposit Page` 1505, `Open Deposit List Page` 1506, etc.); the actual document objects ship in the Microsoft Bank Deposits extension app. Microsoft Learn: <https://learn.microsoft.com/dynamics365/business-central/bank-create-bank-deposits>.
- A Bank Deposit document posts each line as a payment (or refund) against a customer ledger, balanced against the bank account — exactly the journal entries we need for receiving and returning a deposit, with no custom posting logic.
- **Customer Ledger Entries** can carry a payment that is left **open**, so the refund line at return naturally applies against the original deposit entry.
- **Sales Order Prepayments** exist (`Sales Header."Prepayment %"`, codeunit 442 `Sales-Post Prepayments`) but are the wrong instrument here: they post to a Prepayment G/L treated as revenue-in-advance and are netted off the final invoice — the brief requires the deposit to be **refundable in cash**, not credited against the rental fee.

### 6.2 Gaps remaining

- No native action on a Blanket Sales Order to "post the deposit receipt now" or "refund the deposit now". The wizards must drive the Bank Deposit document programmatically.
- No native concept of forfeiting (partially or fully retaining) a posted deposit at return — but this can be expressed as a normal Sales Line on the final Sales Order plus a smaller refund.

### 6.3 Implementation decision

- **Use the standard Bank Deposits feature.** Do not use Sales Order Prepayments (cited reasons above). Do not invent a custom deposit-posting routine.
- On the Sales Header `tableextension` (already present from §2): add `Deposit Amount`, `Deposit Bank Deposit No.` (back-reference to the posted Bank Deposit document), `Deposit Posted` (Boolean), `Deposit Refunded` (Boolean), `Deposit Forfeited Amount`.
- **`Deposit Amount` is computed at pickup** as `20% × <Blanket Order total at pickup>`, so any prolongation that increases the total does not retroactively change the deposit unless a setup flag explicitly demands re-billing.
- **Pickup wizard** (§5, step 5):
  1. Looks up the Bank Account for cash (from a small setup record on the existing `Sales & Receivables Setup` extension or a dedicated `Rental Setup` table — deposit %, bank account, deposit description template).
  2. Programmatically **creates and posts a Bank Deposit document** with one line: `Account Type = Customer`, `Document Type = Payment`, `Amount = Deposit Amount`, description references the Blanket Order No.
  3. Stores the resulting Posted Bank Deposit No. on `Deposit Bank Deposit No.` and sets `Deposit Posted := true`.
- **Return wizard** (§5, step 5):
  1. Determines the refundable portion: `Refund Amount := Deposit Amount − Deposit Forfeited Amount` (forfeited portion comes from damage assessment in the return checklist).
  2. **If `Deposit Forfeited Amount > 0`**, adds a Sales Line on the final Sales Order with the forfeited amount (line type `G/L Account` or a dedicated `Item Charge`/penalty item per setup), so the forfeited portion is recognised as revenue through the normal Sales Order posting.
  3. **If `Refund Amount > 0`**, programmatically creates and posts a Bank Deposit document with one refund line (`Account Type = Customer`, `Document Type = Refund`, `Amount = Refund Amount`, applied against the original deposit Customer Ledger Entry).
  4. Sets `Deposit Refunded := true`.
- **Setup**: one record on the `Sales & Receivables Setup` extension (or a small dedicated setup table) — deposit % (default 20), Bank Account No. for cash deposits, Forfeit G/L Account / Item Charge No., Bank Deposit description template. Business can change the percentage without code.
- **Reports**: standard Bank Account Statement and the open Customer Ledger Entries reconcile outstanding deposits at any point — no custom report needed.

---

## 7. Live car tracking — real-time map inside Business Central

### 7.1 What standard BC already provides

- **`Online Map`** module (`Base Application/eServices/OnlineMap/OnlineMapManagement.Codeunit.al`, codeunit 802) is in the box. It builds a Bing Maps URL for a *single* address and opens it in the browser. Microsoft Learn: <https://learn.microsoft.com/dynamics365/business-central/admin-online-map>.
- **`Geolocation`** table (table 806, `Base Application/eServices/OnlineMap/Geolocation.Table.al`) is a generic Lat/Long holder used by the camera/location capabilities of the mobile clients.
- **Control Add-ins** are the supported mechanism to embed JavaScript-rendered UI (e.g., an interactive map) in a BC page. <https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/devenv-control-addin-overview>.
- **`HttpClient`** is the supported way to call third-party REST APIs from AL. Microsoft Learn: <https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/methods/devenv-httpclient-data-type>.
- **Job Queue** (`Job Queue Entry`, table 472) schedules background polling.

### 7.2 Gaps remaining

- The standard Online Map is *not* a real-time fleet view. It is a one-shot URL builder for a single address. It does not consume a live GPS feed and cannot show multiple moving vehicles.
- No native control add-in renders an interactive map with multiple markers.

### 7.3 Implementation decision

- **GPS provider integration is genuinely custom** and is the largest single area of code in this project.
- Add a small **`Vehicle GPS Position`** table keyed on `Item No.` with `Latitude`, `Longitude`, `Speed`, `Heading`, `Last Updated`. We re-use the existing `Geolocation` table type semantics but as a separate table because we need extra columns and a per-Item clustered key. (Alternative considered: extend `Geolocation` directly — rejected because the standard table is currently used by the camera APIs and we should not change its semantics.)
- A **codeunit `Fleet GPS Connector`** uses `HttpClient` to call the third-party GPS provider's REST API and `UPSERT`s rows in `Vehicle GPS Position`. Authentication tokens live in **Isolated Storage** per the BC security guidance. Polling cadence is configured on the existing **Job Queue** subsystem — no custom scheduler.
- A **`Fleet Live Map`** page (List Plus, source `Item` filtered to `Type = Non-Inventory`) embeds a JavaScript **control add-in** (Leaflet/Bing Maps SDK) that receives the latest positions as a JSON payload from the page's controller and renders markers. Clicking a marker drills to the Item Card.
- A **setup page extension** (or a dedicated `Fleet GPS Setup` table) holds the API base URL, API-key reference (key stored in Isolated Storage, not in the table), and refresh interval.
- We deliberately do **not** persist a full position history — the third-party provider is the source of truth. We keep only the latest position per Item.

---

## Key design decisions

1. **A car is an `Item` of `Type = Non-Inventory` with `Base Unit of Measure = DAY`**, not a custom Vehicle table and not a `Resource` — every booking, pricing, billing and reporting flow then runs on the standard Sales/Pricing plumbing without UoM conversions.
2. **A booking is a standard `Blanket Sales Order` with one Item line per car**, not a custom Rental document — Blanket Sales Orders are first-class in `Sales Header.Document Type` and have a standard `Make Order` mechanism (codeunit 87).
3. **Rental Start Date reuses `Sales Header.Requested Delivery Date` (existing field 5790); Rental Finish Date is a single new field** on a `Sales Header` `tableextension`. No new dates beyond what the rental requires.
4. **Each billing cycle is its own Sales Order created from the Blanket via codeunit 87**, with `Posting Date = Rental Finish Date` if the remaining rental is under 30 days, otherwise `Rental Start Date + 30 days × n` — unifying short-term and long-term billing in one model.
5. **A daily Job Queue task creates cycle 2…N automatically** by filtering Blanket Orders where `Next Billing Date ≤ Today`, advancing `Next Billing Date` by 30 days each time.
6. **The Pickup wizard creates and releases the first cycle Sales Order; the Return wizard creates the final Sales Order** (unbilled days plus extra charges). The Job Queue never produces the first or the last invoice — eliminating any race between scheduled and event-driven posting.
7. **Duration discounts are configured as `Price List Line` rows with `Minimum Quantity` of 7 and 30** — the Best Price engine handles it; we write zero AL pricing code.
8. **Availability is a query against open Sales Lines using `Requested Delivery Date` and `Rental Finish Date`** — no separate availability table, no `Res. Capacity Entry` writes, no parallel reservation ledger. The standard Sales tables are the single source of truth, and the same query backs the Fleet Booking Board calendar and the double-booking guard.
9. **Security deposits are posted through the standard Bank Deposits feature** by the Pickup wizard (deposit) and the Return wizard (refund), not as Sales Order Prepayments — Prepayments would be wrongly recognised as revenue and would net off the final invoice.
10. **Customer identity documents are stored as `Document Attachments` on the Customer**, identity numbers as a `tableextension` classified `EndUserIdentifiableInformation` — no custom file or identity tables.
11. **Pickup/return data is captured on Blanket `Sales Header` extension fields plus `Document Attachments`**, and the rental agreement is a standard report layout — no parallel posting routine.
12. **The two genuinely custom UIs are the Fleet Booking Board (Gantt-style calendar) and the Fleet Live Map**, because BC does not ship comparable controls; both are JavaScript control add-ins on AL pages backed by standard `Item` data and a thin GPS-position table.