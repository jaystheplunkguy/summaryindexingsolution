# Mobile Journey: Simple Summary Indexing

**1 summary index + 25 scheduled searches (29 once steps 03/07/10/35 are added), running every 5 minutes.**
No lookups, KV stores, rules files or macros.

## How it works

1. **Scheduled search per step (`MOB_STEPnn`)**: your existing master query, cut just before the
   `All Order Types` line. That top part already works out one row per order (latency, success, order type).
   It is unchanged, apart from the small additions listed under "Edits" below. A 6-line footer:
   * sets `_time` to when the order started the step and `order_key` to the order/correlation id
   * keeps only orders that started in **one 5-minute slot, a set wait ago** (see the table below),
     so each order is written **once**, after it has had time to finish
   * writes the rows with `collect index=summary_mob_journey`
   Searches run **every 5 minutes**, a few minutes apart. Each run searches only the wait time plus 60 minutes of
   raw data, so there are no timeouts.

   | Steps | Wait before an order is judged | Data delay on the dashboard |
   |---|---|---|
   | 01, 02, 04, 05 | 5 min | about 5–10 min |
   | 06, 08, 11, 12, 19, 22 | 10 min | about 10–15 min |
   | 09, 33 | 15 min | about 15–20 min |
   | 20, 21, 23–31, 34 | 30 min | about 30–35 min |
   | 32 | 40 min | about 40–45 min |
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
2. Add `savedsearches.conf` to an app. Check that the searches run every 5 minutes, a few minutes apart.
3. **Backfill**: run each `MOB_STEPnn` search by hand, one day at a time. Replace the 4 footer lines from
   `| addinfo` to `| fields - info_* slot_end` with
   `| where _time >= relative_time(now(),"-2d@d") AND _time < relative_time(now(),"-1d@d")`
   (step back a day each run, and set the time picker to cover that day plus the step's wait and 60 minutes).
4. **Side-by-side test**: for 2–3 days, compare each new panel with the current raw panel over the same
   **closed** time range (ending at least 1 hour ago). The numbers should match.
5. Switch the Grafana panel queries over.

## Things to know
* **Data delay:** each step is 5–45 minutes behind (table above). Today, orders still in progress count as
  failures. Now an order is judged only after its wait, so the dip at the right-hand edge of the charts goes away.
* **Slow orders:** an order still running when its step's wait ends counts as a failure. Check each wait against
  that step's current P99 latency. To change a step's wait, change the number in its footer
  `floor((info_max_time - <seconds>) / 300)` **and** set `dispatch.earliest_time` to the new wait plus 60 minutes.
* **Pending steps 03/07/10/35:** use the same recipe. Cut the master query at `All Order Types`, add the footer,
  then add the Grafana header to the tail.
* **Fix later:** these issues in the original queries are carried over unchanged, so the numbers stay the same as today.
  * Step 04's order-id regex `^[A-Z]2[4-7]` looks year-coded.
  * Step 06 adds a fixed `+36000` (+10h), which ignores daylight saving.
  * Steps 20–34 set `is_portin` whenever `serviceNumberTransferType` is present, so Port Out orders are counted as Port In.
