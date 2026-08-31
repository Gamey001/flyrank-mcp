# Reorder Report

Warehouse: MSP-3
Reorder threshold: 10 units (config.json)
Source: demo/inventory.csv, maintenance log 2026-08-29

## SKUs at or below threshold

### FR-2298 — rotor bearing
- Qty on hand: 4 (6 below the threshold of 10)
- Unit price: 312.00 USD
- Reason: The maintenance log of 2026-08-29 flags FR-2298 as "stock critically low" and confirms the reorder threshold of 10. At 4 units it is the deepest shortfall in inventory.

### FR-4501 — brake actuator
- Qty on hand: 9 (1 below the threshold of 10)
- Unit price: 1150.00 USD
- Reason: Qty 9 is at/below the threshold, and the maintenance log notes the supplier lead time has been extended to 6 weeks — the reorder should be placed immediately so replenishment is not delayed further.

## Not included

- FR-1042 (hydraulic seal kit), qty 17 — above threshold; log records it passed batch QA, no action.
- FR-3310 (cabin filter), qty 63 — above threshold; no notes.
