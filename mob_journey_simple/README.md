# Mobile Journey: Simple Summary Indexing

**1 summary index + 25 scheduled searches (29 once steps 03/07/10/35 are added).**
No lookups, KV stores, rules files or macros.

## How it works

1. **Scheduled search per step (`MOB_STEPnn`)**: your existing master query, cut just before the
   `All Order Types` line. That top part already works out one row per order (latency, success, order type).
   It is unchanged, apart from the small additions listed under "Edits" below. A 6-line footer:
   * sets `_time` to when the order started the step and `order_key` to the order/correlation id
   * keeps only orders that started in **one 15-minute slot, 60 minutes ago** (15 minutes ago for
     single-event steps 01/02/04/05), so each order is written **once**, after it has had time to finish
   * writes the rows with `collect index=summary_mob_journey`
   Each run searches only the last 3 hours of raw data (1 hour for single-event steps), so there are no timeouts.
2. **Grafana panel (49 files in `grafana/`)**:
   ```
   index=summary_mob_journey step="21"
   | dedup order_key
   <the rest of your original query from "All Order Types" onwards - unchanged>
   ```
   The output columns, SLO targets and `$order_type` handling are all the same as today.

## Edits to the original per-order SPL (only these)
| Step | Change | Why |
|---|---|---|
| 04 | `max(_time) AS t_decision` added to `stats` | needs a time (your percentile query already has it) |
| 05, 06, 12 | `min(_time) AS first_time` added to `stats` | some orders have no start/end time to anchor on |
| 19 | `min(t_sent)`, `min(t_callback)` added to 2nd `stats` | same as your percentile query |
| 26 | `Disconnect -> Disconnect + Port-Out` moved into the Grafana query | keeps the stored order type single-valued |

## Deploy
1. Create the index (`indexes.conf`). Your Splunk role needs permission to write to it.
2. Add `savedsearches.conf` to an app. Check that the searches run every 15 minutes, a few minutes apart.
3. **Backfill**: run each `MOB_STEPnn` search by hand, one day at a time. Replace the 4 footer lines from
   `| addinfo` to `| fields - info_* slot_end` with
   `| where _time >= relative_time(now(),"-2d@d") AND _time < relative_time(now(),"-1d@d")`
   (step back a day each run, and set the time picker to cover that day plus 3 hours).
4. **Side-by-side test**: for 2–3 days, compare each new panel with the current raw panel over the same
   **closed** time range (ending at least 1 hour ago). The numbers should match.
5. Switch the Grafana panel queries over.

## Things to know
* **Data delay:** the newest hour won't appear yet (15 minutes for steps 01/02/04/05). Today, orders still in progress count as
  failures. Now an order is judged after 60 minutes, so the dip at the right-hand edge of the charts goes away.
* **Slow orders:** an order that takes longer than 60 minutes counts as a failure. To change this for a step,
  change `3600` in its footer **and** `dispatch.earliest_time` to the new wait plus 2 hours.
* **Pending steps 03/07/10/35:** use the same recipe. Cut the master query at `All Order Types`, add the footer,
  then add the Grafana header to the tail.
* **Fix later:** these issues in the original queries are carried over unchanged, so the numbers stay the same as today.
  * Step 04's order-id regex `^[A-Z]2[4-7]` looks year-coded.
  * Step 06 adds a fixed `+36000` (+10h), which ignores daylight saving.
  * Steps 20–34 set `is_portin` whenever `serviceNumberTransferType` is present, so Port Out orders are counted as Port In.
