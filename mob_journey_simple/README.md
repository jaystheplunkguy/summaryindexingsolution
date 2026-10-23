# Mobile Journey: Simple Summary Indexing

**1 summary index + 25 scheduled searches (29 once steps 03/07/10/35 are added), running every 5 minutes.**
No lookups, KV stores, rules files or macros.

## How it works

1. **One scheduled search per step (`MOB_STEPnn`), every 5 minutes, staggered.** Each one is your existing master query, cut
   just before the `All Order Types` line. That top part already works out one row per order (latency, success, order type).
   It is unchanged, apart from the small additions listed under "Edits" below. A 7-line footer:
   * reads only the **last 2 hours** of raw data, so there are no timeouts
   * **saves an order only if one of its events happened in the last 10 minutes**. A row is saved when the order starts and
     again when it completes or changes. Orders with nothing new are not saved again.
   * sets `_time` to when the order started the step, and `written_at` to when the row was saved
   * writes with `collect index=summary_mob_journey`
2. **Grafana panel (49 files in `grafana/`)**, which keeps only the **newest row per order**:
   ```
   index=summary_mob_journey step="21"
   | dedup order_key sortby -written_at
   <the rest of your original query from "All Order Types" onwards - unchanged>
   ```
   The output columns, SLO targets and `$order_type` handling are all the same as today.

## What users see
* **Data is about 5 minutes behind**, close to live.
* **In-progress orders show as failures until they complete**, exactly like today's raw panels, so the right-hand edge of the
  charts behaves the same as now. When the completion arrives, the order is saved again and the panel corrects itself.
* **Late logs are handled.** Any update that arrives within 2 hours of the order starting is picked up.

## Edits to the original per-order SPL (only these)
| Step | Change | Why |
|---|---|---|
| 04 | `max(_time) AS t_decision` added to `stats` | needs a time (your percentile query already has it) |
| 05, 06, 12 | `min(_time) AS first_time` added to `stats` | some orders have no start/end time to anchor on |
| 19 | `min(t_sent)`, `min(t_callback)` added to 2nd `stats` | same as your percentile query |
| 26 | `Disconnect -> Disconnect + Port-Out` moved into the Grafana query | keeps the stored order type single-valued |
| 32 | an unfinished order is also re-saved 22 min after it starts | its "eligible / breached" flag changes at 22 min with no new log event |

## Settings you may tune
| Setting | Where | Default | Meaning |
|---|---|---|---|
| Look-back | `dispatch.earliest_time` | `-2h@m` | An order's later events must arrive within this time of its start. If a step can take longer, raise it (e.g. `-4h@m`). Each run then reads more data. |
| "Recent" window | footer `info_max_time - 600` | 10 min | Orders with an event in this window are re-saved. Must be at least the 5-minute schedule; the extra 5 minutes covers small log delays. |
| Schedule | `cron_schedule` | every 5 min | If you change it, keep the recent window at least as long as the interval. |

## Deploy
1. Create the index (`indexes.conf`). Your Splunk role needs permission to write to it.
2. Add `savedsearches.conf` to an app. Check that the searches run every 5 minutes, a few minutes apart.
3. **Backfill**: run each `MOB_STEPnn` search by hand, one day at a time. Replace the footer lines
   `| addinfo` and `| where changed_at >= info_max_time - 600` with
   `| where _time >= relative_time(now(),"-2d@d") AND _time < relative_time(now(),"-1d@d")`
   (step back a day each run, and set the time picker to cover that day plus 2 hours).
4. **Side-by-side test**: for 2–3 days, compare each new panel with the current raw panel over the same time range. The numbers
   should match. Small differences in the last few minutes are expected while orders are still in progress.
5. Switch the Grafana panel queries over.

## Things to know
* **Rows per order:** usually 1–3 (start, completion, any change). The summary store stays small.
* **Updates later than the look-back:** if an event arrives more than 2 hours after the order started, the run can no longer
  see the start. It may save a less complete row over a good one. Set the look-back above the slowest step's normal duration.
* **Pending steps 03/07/10/35:** use the same recipe. Cut the master query at `All Order Types`, add the footer (with that step's
  event-time fields in `changed_at`), then add the Grafana header to the tail.
* **Fix later:** these issues in the original queries are carried over unchanged, so the numbers stay the same as today.
  * Step 04's order-id regex `^[A-Z]2[4-7]` looks year-coded.
  * Step 06 adds a fixed `+36000` (+10h), which ignores daylight saving.
  * Steps 20–34 set `is_portin` whenever `serviceNumberTransferType` is present, so Port Out orders are counted as Port In.
