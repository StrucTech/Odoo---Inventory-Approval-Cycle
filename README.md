# Odoo — Inventory Approval Cycle (Double Count)

A double-count and manager-approval cycle for Odoo Inventory, configured entirely from
Settings → Technical (no custom module): custom fields, a custom model, automation rules
and server actions. Stock Keepers count in parallel from the Barcode app; nothing
touches stock until an Inventory Manager approves a line that has 2-of-3 agreement.

## Documentation

| Document | Description |
| --- | --- |
| [Production Runbook — Double Count Implementation](docs/Production_Runbook_Double_Count_Implementation.docx) | Step-by-step production rollout (Arabic), final code for every rule and action, pilot validation plan, known pitfalls and rollback. Tested on `nycretail-staging`. |

## How it works

1. **Block Inventory Apply** — only Inventory Managers can apply inventory adjustments
   (including *Apply Now* in Barcode). Stock Keepers use *Wait a Review*.
2. **Start Count** on a warehouse opens a new count session.
3. **Capture rule** — every saved count is moved off the quant into an
   `x_inventory_count` line for the current session:
   - first counter → **Count A** (same user re-saving adds to it)
   - second, different counter → **Count B** + **Diff A-B**
   - a third counter is rejected unless a recount was requested → **Recount (C)**
4. The manager reviews lines in **Inventory → Operations → Adjustments → Double Count Review**
   and uses the actions **Request Recount**, **Find Uncounted Items**,
   **Confirm Not Found** and **Approve & Apply**.
5. **Approve & Apply** requires 2-of-3 agreement and applies the *difference*
   (counted − on-hand at count time) to the current stock, so sales made after the
   count are preserved.

Open lines are never lost: starting a new session keeps pending lines visible, and a
newer count for the same product/location marks the older line **Superseded**.

## Components

| Step | Component | Where |
| --- | --- | --- |
| 1 | Automation rule *Block Inventory Apply - Non Managers* | `stock.move`, on create |
| 2 | Fields `x_count_round`, `x_count_session` | `stock.warehouse` |
| 3 | Model `x_inventory_count` (16 fields, no delete rights) | Custom model |
| 4 | List view `x_inventory_count.list` + *Double Count Review* menu | Inventory / Adjustments |
| 5 | Automation rule *Double Count - Capture* | `stock.quant`, on create and edit of `inventory_quantity` |
| 6 | Server actions: Start Count, Find Uncounted Items, Close Count, Request Recount, Confirm Not Found, Approve & Apply | Warehouse / Inventory Count Line |
| 7 | Pilot validation on one closed branch | — |

Follow the runbook steps **in order** — views and code fail if the fields and model
they reference do not exist yet.

## Line statuses

| Status | Meaning |
| --- | --- |
| Pending Review | Counted, waiting for manager decision |
| Recount Requested | Manager asked for a third count |
| Not Scanned | Product has stock but nobody counted it (from *Find Uncounted Items*) |
| Applied | Approved and applied to stock |
| Superseded | Replaced by a newer count of the same product/location |

## Rollback

Every step can be reverted immediately (archive the rules/actions, remove the menu,
view and model). The only step that changes stock is **Approve & Apply**; a wrong
approval is reversed with a manager adjustment in Physical Inventory. See the
runbook for details.
