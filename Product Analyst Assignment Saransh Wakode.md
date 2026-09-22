# Product Analyst Intern — Take-Home Assignment

## Task 1 — Documentation vs API Responses

I compared the API documentation with all three response files and found the following inconsistencies:

| Issue                     | Documentation says                                                                          | Actual response                                                                  | Impact                                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Unsupported status        | `status` is one of `pending`, `shipped`, `delivered`, `cancelled`                           | `ord_1003` has `status: refunded`                                                | A client using the documented enum may reject or mishandle this order.                      |
| Incorrect total           | `total = subtotal + tax + shipping`                                                         | `ord_1004`: 6200 + 511 + 599 = 7310, but API returns 6810                        | Revenue, invoice and reporting calculations can be wrong.                                   |
| Missing customer email    | Customer email is always present                                                            | `ord_1005` has `email: null`                                                     | Systems expecting an email may fail or require special handling.                            |
| Wrong monetary format     | All amounts are integers in the smallest currency unit                                      | `ord_1006` returns `44.0`, `3.63`, `5.99`, `53.62`                               | Different clients may interpret the same value differently, causing incorrect calculations. |
| Pagination/order behavior | Results are most recent first; `has_more` controls whether another page should be requested | Page 1 is oldest-to-newest and says `has_more: false`, while another page exists | A client could stop after page 1 and miss orders.                                           |
| Missing-order response    | A nonexistent order should return HTTP 404                                                  | `order_ord_9999.json` returns HTTP 200 with `order: null`                        | Clients may treat the request as successful instead of handling it as a missing resource.   |

### Most serious issue

I would prioritize the **monetary amount inconsistency**. Financial systems depend on amounts having a consistent unit and format. `ord_1006` uses decimal values even though the documentation says amounts are integer values in the smallest currency unit. Combined with the incorrect `total` on `ord_1004`, this can directly affect revenue reporting and reconciliation.

## Task 2 — Revenue Calculation

I would **not present a single fully reliable revenue number from these payloads without making assumptions**.

The main problem is `ord_1006`: its monetary fields are formatted as dollars (`44.0`, `3.63`, `5.99`, `53.62`), while the other orders use cents. If I interpret `ord_1006` as dollars and convert it to cents, the six returned orders total **32,803 cents = $328.03** using the API's `total` fields.

However, `ord_1004` should mathematically total 7,310 cents rather than the returned 6,810 cents. Correcting that would make the total **$333.03**.

I would therefore report **$328.03 as the raw normalized API total, with a warning that it is not a reliable final revenue figure**. Before using it for finance reporting, I would ask the API owner to confirm the monetary unit for `ord_1006`, correct `ord_1004`, and confirm pagination because page 1 says `has_more: false` even though another response exists.

## Task 3 — Customer Reply and Bug Report

### Reply to Priya

Hi Priya,

Thanks for flagging this. I found a few data issues in the Meridian API that can explain reconciliation differences, including an order with an incorrect total and inconsistent monetary formatting on another order.

I would not treat the current API totals as fully reliable for the monthly report until these issues are corrected. I’m also checking the pagination behavior to make sure no orders are being missed.

I’ll flag these issues to engineering with the affected order IDs so they can be corrected and the report can be reconciled.

Best,
Saransh

### Bug Report

**Title:** Meridian Orders API returns inconsistent monetary values and incorrect order totals

**Affected orders:** `ord_1004`, `ord_1006`

**What to look at:**

* `ord_1004`: subtotal `6200`, tax `511`, shipping `599`, total `6810`
* `ord_1006`: monetary fields are returned as decimals (`44.0`, `3.63`, `5.99`, `53.62`)

**What happens:**
`ord_1004` violates the documented calculation because 6200 + 511 + 599 = 7310. `ord_1006` violates the documented integer/smallest-unit format.

**Expected behavior:**
All monetary fields should use the documented smallest currency unit and integer format, and `total` should always equal `subtotal + tax + shipping`.

**Impact:** Finance reports and API clients can calculate incorrect revenue.

**Suggested fix:** Correct the affected payloads, enforce the monetary schema, and add validation so an order cannot be returned when its total does not equal its components.
