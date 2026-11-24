---
name: allocate-po-costs
description: >
  For Inventory IQ (unit tracking) users: allocate an additional / indirect cost (freight, duties, insurance,
  pallet fees...) across purchase-order lines by a unit of measure the user chooses — weight, volume, units,
  pallets, or any custom measure — or by a manual split, instead of Finaloop's default automatic allocation
  by value. Keeps each product's measures per company for reuse. Everything goes through the Finaloop MCP
  (purchase-order API); POs in a locked year are never edited. Trigger words: allocate cost, indirect cost,
  additional cost, landed cost allocation, allocate by weight, allocate by volume, allocate by CBM, split
  freight, split shipping cost, duties allocation, container cost, manual allocation, product weight, product
  volume, unit of measure, Inventory IQ allocation.
---

# Inventory IQ — allocate an indirect cost by weight / volume / manual split

**Why this exists:** Finaloop splits a PO's indirect costs across its lines by value (product cost). Many costs —
freight, duties, pallet fees — really follow weight, volume or piece count. This skill lets the user pick the
basis (or split by hand), computes the per-line amounts, and saves them on the PO as a manually allocated cost.

Everything goes through the Finaloop MCP (`search` / `introspect` → `validate` → `execute`, plus
`update_purchase_order`). Field names below are real; re-check enum values and required args with `introspect`
rather than guessing. No code or runtime is required: the math in step 7 is plain arithmetic — do it carefully,
show your work, and check the sums before saving.

## Before you start

Check the Finaloop connection first:

- **No Finaloop tools, or they need sign-in:** tell the user how to connect for the app they are using, then stop:
  - Claude Desktop or claude.ai: Customize > Plugins > Finaloop > Connectors > Connect
  - Claude Code: run `/mcp` and sign in to Finaloop
  - Codex: run `codex mcp login finaloop`
  - Anything else: open the app's plugin settings and connect Finaloop
- **Connected, but `update_purchase_order` isn't among the tools:** tell the user this workflow isn't available on
  their Finaloop plan, then stop.

## Flow at a glance

Setup (once, reused): the user gives per-product measurements (step 1).
Every run: 2 which POs → 3 which cost (existing or new per PO) → 4 allocation basis → 5 which products →
6 measurements present for all of them? → 7 compute + preview → 8 confirm, save, verify.

## Ground rules

- **Never write without an explicit yes.** Always show the full before → after split and ask "OK to apply?".
- One company at a time. If the user has several, ask which.
- Plain business language to the user: no ids, enum values, field names.
- **Locked year:** a PO whose production (in-progress) date is before the company's
  `lockStatus.minimumAllowedEventTime` cannot change its costs. Skip it, tell the user, never work around it.
  If the API still refuses with `LOCKED_PERIOD`, say the period is closed in the books.
- Only cost-adding types are allocated here: SHIPPING, DUTIES_AND_TAX, INSURANCE, VENDOR_FEES,
  CUSTOM_FEES_AND_SERVICES, PALLET_COSTS, SHIPPING_SURCHARGE, PRODUCTION_COSTS, PRODUCTION_LABOR_COSTS, GENERAL,
  OTHER. Discounts / credits are not "additional costs" — send the user to the app for those.
- A measure-based or manual allocation is saved as a **manual allocation** on the PO. Tell the user: from then on
  the app shows that cost as manually allocated, and changing the PO's lines or costs later means re-doing the
  allocation (in the app, or with this skill again).
- Dropshipping POs: the API refuses content changes — skip and say so. Drafts: allowed, but ask first.
- Never hand-patch a number to make the API accept a payload. If a total doesn't reconcile, recompute.

## Step 1 — Product measures (per company, reused every run)

A measure is a **per-unit** number for a product, in a named column: `weight_kg`, `volume_cbm`,
`pallets_per_unit`, `weight_lb`, `cartons_per_unit`… **Any name the user wants; the name carries the unit.** A
product can have several measures at once; each column is a separate basis the user can pick in step 4. Two
built-in bases need no table: `units` (quantity) and `value` (product cost — Finaloop's default).

The measures table: `product id | SKU | product name | measure columns…`, one row per product.

**Where it lives** — depends on where this skill runs:
- If files can be written (e.g. a scratch or project folder), keep it at
  `<work folder>/inventoryiq-cost-allocation/<companyId>/measures.csv` and reload it at the start of every run.
- Otherwise keep it as a markdown table in the conversation / project knowledge: at the end of every run, print
  the full updated table and ask the user to save it and paste it back next time.

1. If a table exists, load it and tell the user which measures are known and how many products have each.
2. If it doesn't, or the user wants to add a measure / update values:
   - List the catalog: `products2(input: { companyId, limit: 100, offset, sort: { field: SKU, direction: ASC },
     filter: { attributes: [ACTIVE] } }) { products { id sku displayName } hasNext }` — page while `hasNext`.
   - Accept a pasted table / CSV / spreadsheet (`sku, weight_kg, volume_cbm, …`) or values typed in chat. Match
     on SKU (exact, trimmed), fall back to product name. **Store by product id**, so duplicate SKUs (several
     products sharing one SKU) are told apart by name. List rows that matched no product. Never guess a value.
   - Units must be consistent inside one column (all kg, or all lb). Adding a measure = adding a column;
     existing columns stay.
   - Save the table, show a summary (products covered / missing per measure).
3. Values given later for new products (step 6) are added to the same table.

## Step 2 — The user picks the POs (never guess)

Ask how they want to point at the POs — one of:

- **Specific POs** — one or many names (e.g. "PO-12, PO-15, PO-18"):
  `supplyChainPurchaseOrders(input: { companyId, filters: { searchField: "<name>", includeDrafts: true },
  sort: { field: PLACEMENT_DATE, direction: DESC }, offset: 0, limit: 50 })`. Names not found → list them back.
- **All POs carrying a cost type** — e.g. "every PO with duties from 2026", optionally narrowed by vendor.
  There is no API filter on cost type: page `supplyChainPurchaseOrders` with `filters: { inProgressDate: { from, to },
  vendorIds, includeDrafts: true }`, `limit: 100`, while `hasNext`, requesting
  `purchaseOrders { id displayName placementDate inProgressDate isDraft isDropshipping supplyChainStatus
  vendor { name } totalCount totalQuantity indirectLineItems { type cost hasAutomaticAllocation } }`, and keep the
  POs whose `indirectLineItems[].type` matches. For a large company insist on a date range or vendor.
- **A filter** — vendor, placement / production date range, status.

Lock check: `company(args: { companyId }) { name lockStatus { minimumAllowedEventTime } }`; a PO with
`inProgressDate < minimumAllowedEventTime` is locked.

**Confirm the POs.** Show a table: PO name, vendor, placement date, production date, status, lines, existing
additional costs (type + amount + automatic/manual), locked yes/no. Locked and dropshipping POs are listed as
excluded. Ask **"These POs — correct?"** and let the user add / drop. Nothing proceeds until they say yes.

Then read every confirmed PO in full (this is the data the save in step 8 is rebuilt from):

```graphql
query($companyId: ID!, $purchaseOrderId: ID!) {
  supplyChainPurchaseOrder(input: { companyId: $companyId, purchaseOrderId: $purchaseOrderId }) {
    id companyId displayName placementDate inProgressDate isDraft isDropshipping notes
    vendor { id name }
    customer { id contactName phoneNumber contactEmail addressLine1 country city state zipCode }
    attachments { id filename url }
    directLineItems { id product { id sku displayName } quantity vendor { id } directCostPerUnit directCostTotal note
      indirectCostPerUnit landedCostPerUnit
      indirectCostsAllocation { indirectLineItem { id } allocatedCostPerUnit } }
    indirectLineItems { id type cost vendor { id name } isDefaultVendor hasAutomaticAllocation shipmentId }
    shipmentGroups { id displayName notes linkedShipment { id } contents { lineItem { id } quantity } }
    directCostTotal indirectCostTotal discountCostTotal landedCostTotal
  }
}
```

## Step 3 — Review and confirm the cost

Two cases; ask which (or infer from what they said, then confirm):

- **Re-allocate a cost already on the POs** — show per PO each cost line: type, vendor, amount, current
  allocation (automatic by value / manual) and today's split per line (`allocatedCostPerUnit × quantity`). The
  user picks which cost line(s) to re-split and may correct the amount. A reused line keeps its id.
- **Add a new cost** — collect **cost type** (plain words → type: "freight" → SHIPPING, "customs" →
  DUTIES_AND_TAX…), **vendor** (`vendors(input: { companyId, searchTerm }) { id name }`; several matches → ask;
  the PO's own vendor when they say "same vendor"), **total amount** (home currency, > 0). With several POs ask
  whether the amount is one grand total to split across POs, or a per-PO amount.

**Confirm the cost.** One table: PO, cost type, vendor, amount before → after. Ask **"These costs and amounts —
correct?"**

## Step 4 — The user picks the allocation basis

Ask: **automatic by a measure, or manual?**

- **Automatic** — list the bases available: every measure column in the table, with its coverage of the
  products on the confirmed POs ("weight_kg — 14 of 16 products"), plus `units` and `value`. The user picks
  **one** basis for this run.
- **Manual** — amounts or percentages per line (or per PO, then per line). Must add up to the total / 100%;
  lines not mentioned get 0.
- **Value (Finaloop's default)** for a *new* cost — the simple path is
  `update_purchase_order(companyId, purchaseOrderId, indirectCosts: [<all existing order-level costs> + new one])`;
  `indirectCosts` **replaces** the order-level costs, so resend the existing ones; the server splits by value.
  Refused with MANUAL_ALLOCATIONS_WOULD_RESET when the PO already has a manual allocation → use step 7/8 with
  basis = value.

## Step 5 — Which products share the cost: all, or some?

Ask: **all products on these POs, or only some?** (duties only on imported lines, pallet fees only on bulky
SKUs…). Show the resulting line list per PO and confirm. Excluded lines get **0** of this cost — say so.

## Step 6 — Are the measurements there for every included product?

For the chosen basis, list every included product with no value in the table (new products are the usual
case). Ask for those values **before allocating** — only those products. Add them to the table so next time
they're known. If the user can't provide one: exclude that line (0 of the cost) or switch basis. The preview
states which lines are excluded and why.

## Step 7 — Compute and preview

Work in exact decimals (write amounts with all digits; don't round intermediate steps). Per PO:

**7a. Weight of each included line** — `w_i = quantity_i × measure_i`
(`units`: `w_i = quantity_i`; `value`: `w_i = quantity_i × directCostPerUnit_i`). Excluded lines: `w_i = 0`.

**7b. Grand total across several POs** (only when the user gave one total): PO share = `total × ΣwPO / Σw all`,
rounded to 2 decimals, the last PO taking `total − sum of the others` so they add up exactly. Then treat each PO
with its share.

**7c. Line totals** — `T_i = amount × w_i / Σw`, rounded to 2 decimals; the **last included line** takes
`amount − sum of the others` so `ΣT_i = amount` exactly. Manual: `T_i` = the user's amounts (percent × amount,
2 decimals, last line absorbs the rounding).

**7d. Per-unit with remainder** — the API stores a per-unit amount for every unit but the last, and a
"remainder" for the last unit, so a line total is exact:
`perUnit_i = round8(T_i / quantity_i)`, `lastUnit_i = round8(T_i − perUnit_i × (quantity_i − 1))`.
(8 decimals; for quantity 1 both equal `T_i`.) Check: `perUnit_i × (quantity_i − 1) + lastUnit_i = T_i`.

**7e. Other costs on the PO keep their allocation.** For every other cost line: if it is automatic
(`hasAutomaticAllocation: true`), its totals per line are its cost split by value (7a–7d with basis value, on
*all* lines); if manual, its per-line total is the stored `allocatedCostPerUnit × quantity` (last line absorbs
any rounding so the cost sums exactly). Discounts/credits (DISCOUNT, CREDIT) are negative allocations.

**7f. Per-line fields the API validates** (4-decimal tolerance, except direct-cost totals, which must match
exactly), for each line:
- `indirectCostPerUnit` = Σ perUnit of all cost-adding costs; `remainderIndirectCostPerUnit` = Σ lastUnit.
- `discountCostPerUnit` / `remainderDiscountCostPerUnit` = the same for discounts/credits, as positive numbers.
- `landedCostPerUnit` = direct + indirect − discount (per unit); `remainderLandedCostPerUnit` the same with the
  remainder values.
- `directCostTotal` = the line's value from the step-2 read, resent unchanged; `indirectCostTotal` = indirectCostPerUnit × (qty − 1) +
  remainderIndirectCostPerUnit; `discountCostTotal` likewise; `landedCostTotal` = direct + indirect − discount.

**7g. PO totals** — `directCostTotal` = the PO's value from the step-2 read, resent unchanged; `indirectCostTotal` = Σ cost-adding `cost`;
`discountCostTotal` = Σ discount/credit `cost` (positive); `landedCostTotal` = direct + indirect − discount.
Cross-check: Σ line `indirectCostTotal` = PO `indirectCostTotal`; Σ line `landedCostTotal` = PO `landedCostTotal`;
for each cost line, Σ over lines of `perUnit × (qty − 1) + lastUnit` = its `cost`.

**Preview** to the user, per PO: SKU, product, qty, measure total (`w_i`), share %, amount of this cost before →
after, landed cost per unit before → after, totals row. Say which lines are excluded. Then ask **"OK to apply?"**.

**Worked example** — duties 500 by volume_cbm, 4 lines:

| SKU | Qty | cbm/unit | w = qty×cbm | T (2dp) | perUnit (8dp) | lastUnit |
|---|---|---|---|---|---|---|
| 897 | 1 | 0.015 | 0.015 | 31.51 | 31.51 | 31.51 |
| 40915 | 11 | 0.008 | 0.088 | 184.87 | 16.80636364 | 16.8063636 |
| 130692 | 3 | 0.025 | 0.075 | 157.56 | 52.52 | 52.52 |
| 12345667 | 5 | 0.012 | 0.060 | 126.06 (last: 500 − 373.94) | 25.212 | 25.212 |

Σw = 0.238; 500 × 0.015 / 0.238 = 31.5126… → 31.51; etc. Check 40915: 16.80636364 × 10 + 16.8063636 = 184.87 ✓.

## Step 8 — Save

Only after a clear yes. Build the input from the step-2 read + step-7 numbers — **every existing line, cost,
shipment group, attachment, note and customer field is resent unchanged**; only the allocated cost and the
computed numbers change:

```graphql
mutation($input: UpsertSupplyChainPurchaseOrderInput!) {
  upsertSupplyChainPurchaseOrder(input: $input) {
    __typename
    ... on UpsertSupplyChainPurchaseOrderSuccess { purchaseOrderId }
    ... on SupplyChainPurchaseOrderValidationError { code message }
  }
}
```

`input` shape (run `introspect UpsertSupplyChainPurchaseOrderInput` for the full type):
- `id`, `companyId`, `displayName`, `shouldValidateDisplayName: false`, `placementDate`, `inProgressDate`,
  `vendorId`, `notes`, `isDraft`, `isDropshipping`, `attachments: [{ id, filename, url }]`,
  `customer` (only when the PO has a customer id; omit otherwise).
- `directLineItems: [{ id, productId, quantity, vendorId, directCostPerUnit, note,
  indirectCostsAllocation: [{ indirectLineItemId, allocatedCostPerUnit: perUnit, remainderAllocationCostPerUnit: lastUnit }
  — one entry per cost line on the PO, including the ones with 0],
  indirectCostPerUnit, remainderIndirectCostPerUnit, discountCostPerUnit, remainderDiscountCostPerUnit,
  landedCostPerUnit, remainderLandedCostPerUnit, directCostTotal, indirectCostTotal, discountCostTotal,
  landedCostTotal }]` — allocations for DISCOUNT/CREDIT lines are sent **negative**.
- `indirectLineItems: [{ id, type, cost, vendorId, isDefaultVendor, hasAutomaticAllocation, shipmentId? }]` —
  the allocated cost gets `hasAutomaticAllocation: false` (new cost: a fresh UUID as `id`,
  `isDefaultVendor` = vendor is the PO's vendor); all others exactly as read.
- `shipmentGroups: [{ id, displayName, notes, supplyChainShipmentId: linkedShipment.id,
  contents: [{ lineItemId, quantity }] }]` — as read.
- `directCostTotal`, `indirectCostTotal`, `discountCostTotal`, `landedCostTotal` from 7g.

`validate` the operation, then `execute`. One PO at a time; a failure on one doesn't stop the others — report each.

**Verify:** re-read the PO (`supplyChainPurchaseOrder` with `indirectLineItems { type cost hasAutomaticAllocation }
directLineItems { product { sku } quantity indirectCostTotal landedCostPerUnit indirectCostsAllocation
{ indirectLineItem { type } allocatedCostPerUnit } } indirectCostTotal landedCostTotal`) and confirm the cost's
amount, the manual flag, and each line's allocation equal the preview. Report before → after per PO in plain
words. If anything differs, say so — don't re-save silently.

Finally, if the measures table isn't stored in a file, print the full updated table for the user to keep.

## Errors in plain words

| API says | Tell the user |
|---|---|
| LOCKED_PERIOD | That PO's period is closed in the books; its costs can't change. |
| PAYMENT_GROUPS_WOULD_BE_DROPPED | This change would unlink payments recorded against the PO. Stop; do it in the app, or only on explicit confirmation resend with `acknowledgedLinkedEntityDeletionIds` = the group ids named. |
| MANUAL_ALLOCATIONS_WOULD_RESET | (from `update_purchase_order`) the PO already has a manual allocation; use the full save in step 8. |
| INDIRECT_COST_TYPE_NOT_ALLOWED / INVALID_INDIRECT_COST | That cost type isn't allowed on this PO. |
| DROPSHIPPING_NOT_EDITABLE | Dropshipping POs can't be edited here. |
| MISSING_DIRECT_COST / MISSING_PLACEMENT_DATE / MISSING_VENDOR | The PO is missing a product cost / date / vendor; add it in the app first. |
| "… does not match computed total …" | Your arithmetic is off; redo step 7 checks. Never nudge a number to pass. |
| UNSUPPORTED_MANAGEMENT_METHOD | The company isn't on Inventory IQ (unit tracking); this skill doesn't apply. |

## Alternative: cost on a shipment instead of the PO

If the cost belongs to a supply-chain shipment (one freight invoice for a container carrying several POs), the
shipment API takes manual allocations as plain totals per PO line — same 7a–7c math, no per-unit fields:
`upsertSupplyChainShipment(input: { …shipment as read…, costs: [{ id, type, cost, vendorId,
manualAllocations: [{ directLineItemId, totalAllocated }] }] })`. Offer it when the user talks about a shipment or
container rather than a PO.
