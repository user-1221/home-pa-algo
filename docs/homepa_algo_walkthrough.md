## Home-PA Grid Scheduler Walkthrough

This note captures the current logic in `algo_1.3.ipynb`. It supersedes the legacy write-up and keeps pace with the “score-first, travel-constrained” allocator that now powers the notebook cells.

---

### 1. High-Level Flow

1. **Score Suggestions**  
   `need` and `importance` are clamped to `[0, 1]`; their sum is cached as the selection score.

2. **Greedy Capacity Check**  
   Suggestions are traversed in score-descending order. Each one is kept if its entire `duration` (there is no stretching in 1.3) fits inside the remaining minutes of the still-open gaps.

3. **Apply Location Preferences**  
   Every selected suggestion is inspected for an optional `location_preference` flag (`"near_home"`, a concrete address, or `None`). Location-constrained items must anchor to a gap whose preceding or succeeding event is at the preferred location. The user is assumed to start and end at home, so “near_home” suggestions always have a viable boundary. These constrained suggestions are fixed to compatible gaps before the rest of the optimizer runs.

4. **Respect the Permutation Limit**  
   If more candidates survive than `permutation_limit` allows, the lowest-score extras are dropped before search. The same dropping rule is used whenever an ordering fails to embed.

5. **Find the Lowest-Travel Order**  
   Distance-based calculations are now only needed when a suggestion cites a concrete address. For those batches we enumerate every permutation (bounded by `permutation_limit`) between the current cursor and the last gap’s end location, tracking raw grid distance as travel cost.

6. **Assign Blocks With Travel Buffers**  
   The chosen order is streamed through `assign_order_to_gaps`. Each suggestion travels from the current cursor into its location, consumes a single contiguous block equal to `duration`, and then hands back the cursor. If the travel-plus-block minutes overflow the active gap, we exit that gap (booking travel to the end location) and continue with the next one.

7. **Recover From Failures**  
   When assignment fails—because travel buffers or durations no longer fit—the allocator drops the lowest-score candidate from the batch, then reorders and retries. If no candidate succeeds, the outer loop exits.

8. **Repeat Until Done**  
   The outer loop keeps pulling from the remaining suggestions until capacity is exhausted or nothing eligible remains. Final stats include travel cost, unused gap minutes, permutation counts, movement logs, and the list of dropped suggestions.

---

### 2. Key Data Structures

| Symbol | Description |
| ------ | ----------- |
| `Gap` | Holds the free window’s duration plus the start/end coordinates that determine travel buffers. |
| `Suggestion` | Requested duration in minutes plus an optional `location_preference` that can be `"near_home"`, a specific address, or `None`. |
| `ScheduledBlock` | One uninterrupted chunk of a suggestion placed in a gap. |
| `MovementLogEntry` | Records every hop the allocator makes, including distance and minutes consumed. |
| `AllocationState` | Carries persistent gap usage, cursor location, and labels so later passes continue seamlessly. |
| `ScheduleResult` | Final outcome: ordered suggestions, scheduled blocks, dropped candidates, travel cost, unused minutes, permutation stats, and the movement log. |

---

### 3. Parameter Cheatsheet

| Parameter | Where | Impact |
| --------- | ----- | ------ |
| `need`, `importance` | `Suggestion` | Overall score = `clamp01(need) + clamp01(importance)`; higher scores gain priority throughout the loop. |
| `duration` | `Suggestion` | Single-block commitment. Version 1.3 no longer stretches suggestions beyond this value. |
| `location_preference` | `Suggestion` | Optional value that forces placement near home or at a specific address. Constrained suggestions are allocated before the general permutation search. |
| `max_distance` | `schedule_suggestions` arg | Only applied when a suggestion names a specific address; it bounds the follow-up travel calculations during permutation search. |
| `permutation_limit` | `schedule_suggestions` arg | Caps both the batch size and the factorial search. Anything beyond the limit is dropped, lowest score first. |
| `TRAVEL_MINUTES_PER_DISTANCE` | Module constant | Converts Euclidean distance into travel minutes (default `3.0`). Higher values reserve more buffer when entering or exiting gaps. |
| `tolerance` | `schedule_suggestions` arg | Floating-point guardrail used when comparing travel + block totals to the remaining gap minutes. |

**Practical tip:** Watch for interactions between longer durations and tighter gaps. Since 1.3 requires the full duration to fit alongside travel buffers, even high-score items can be dropped if the cursor starts far from their location.

---

### 4. Movement Log

Each travel segment—whether entering a suggestion or exiting a gap—is recorded (`from_label -> to_label`) along with:

- Which entity we travelled from (gap boundary or previous suggestion).
- The destination suggestion or gap end.
- Euclidean distance in grid units.
- Travel minutes derived from `TRAVEL_MINUTES_PER_DISTANCE`.

The notebook prints the log both in the formatted summary and inside the demo cell so you can audit how the allocator traversed the grid.

---

### 5. Extending the Notebook

1. **Adjust Parameters Quickly**  
   Edit the sample `Suggestion` objects or the module-level constants to explore different scoring mixes, distances, and travel costs.

2. **Inject New Data**  
   Replace the demo lists with real inputs as long as every suggestion and gap has coordinates in the shared grid system.

3. **Inspect Movement Trade-offs**  
   Review the movement log after each run. Long travel arcs often indicate that `max_distance` is too generous or that the current cursor should hop to the next gap sooner.

4. **Add Post-Processing**  
   `ScheduleResult` keeps `allocated_minutes`, `movement_log`, and `dropped_suggestions`, so downstream dashboards can reuse the outputs without recomputing permutations.

---

### 6. Troubleshooting

- **Many suggestions dropped early:** Confirm they sit within `max_distance` and that the full duration plus travel fits inside at least one remaining gap.
- **Gaps left partially unused:** Travel buffers can quietly consume minutes when the cursor daisy-chains distant items. Consider reordering the inputs or tightening the distance threshold.
- **Permutation search explodes:** Lower `permutation_limit` or partition suggestions by geography so each batch respects the limit.
- **Unexpected travel spikes:** Double-check coordinates; outliers or swapped axes inflate travel most dramatically.

---

Use this walkthrough as a companion when tweaking parameters or porting the logic into another runtime.  The notebook code mirrors everything described here, so you can follow along cell-by-cell as you experiment. 

