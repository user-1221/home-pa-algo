## Home-PA Grid Scheduler Walkthrough

This note captures the refreshed logic used in `algo_1.3.ipynb`.  It is meant to replace the original documentation that shipped with the legacy notebook and offers a concise guide to the new “minimum-first, stretch-later” allocator, with a special focus on the parameters that shape the result.

---

### 1. High-Level Flow

1. **Evaluate Suggestions**  
   Each suggestion carries a `need` and `importance` signal.  These are clamped to `[0, 1]` and summed to produce a ranking score.

2. **Initial Filtering by Location**  
   Suggestions are eligible only if they lie within `max_distance` units of at least one gap boundary (start or end).  This avoids later permutation work for clearly incompatible items.

3. **Greedy Selection of Minimum Blocks**  
   Candidates are considered in score-descending order.  A suggestion is admitted when its **minimum commitment** (`base_duration`) fits in the remaining global capacity.  The process repeats after every allocation pass so any unused time can be repurposed for the next-best suggestions.

4. **Travel-Aware Permutation Search**  
   Within each batch we enumerate every ordering (bounded by `permutation_limit`) and pick the sequence that minimises the travel cost from the current gap cursor to each suggestion and, finally, to the last gap’s closing location.

5. **Gap Assignment with Stretching**  
   The allocator places at least one block for every ordered suggestion.  If `splittable` is true, additional base-sized blocks are appended (up to `max_multiples`) as long as travel buffers and gap capacity allow.  Travel time is subtracted as “slack” before and after each block so we never overrun a gap.

6. **Iterate Until Gaps Are Full or Suggestions Exhausted**  
   Any leftover gap time triggers another selection round with the remaining suggestions.  We continue until capacity is consumed or no compatible suggestions remain.

---

### 2. Key Data Structures

| Symbol | Description |
| ------ | ----------- |
| `Gap` | Holds the free window’s duration plus the start/end coordinates that determine travel buffers. |
| `Suggestion` | Minimum block (`base_duration`), maximum multiples (`max_multiples`), location, and whether it can be split. |
| `ScheduledBlock` | One uninterrupted chunk of a suggestion placed in a gap. |
| `MovementLogEntry` | Records every hop the allocator makes, including distance and minutes consumed. |
| `AllocationState` | Carries persistent gap usage, cursor location, and labels so later passes continue seamlessly. |
| `ScheduleResult` | Final outcome: ordered suggestions, scheduled blocks, dropped candidates, travel cost, unused minutes, permutation stats, and the movement log. |

---

### 3. Parameter Cheatsheet

| Parameter | Where | Impact |
| --------- | ----- | ------ |
| `need`, `importance` | `Suggestion` | Combined score drives selection order.  A higher sum means the suggestion is considered earlier when capacity is scarce. |
| `base_duration` | `Suggestion` | **Minimum commitment** that must fit before a suggestion can enter the schedule.  Also the size of each additional block for splittable suggestions. |
| `max_multiples` | `Suggestion` | Caps how many base-duration blocks we will allocate.  A value of 1 means “place once, never stretch”. |
| `splittable` | `Suggestion` | When `False`, all blocks must fit contiguously in a single gap; when `True`, the allocator may spread blocks across gaps while respecting travel buffers. |
| `max_distance` | `schedule_suggestions` arg | Defines the spatial eligibility.  Larger values accept more suggestions but can increase travel cost. |
| `permutation_limit` | `schedule_suggestions` arg | Bounds the factorial search.  Set it low (e.g., 6–8) to balance search quality against performance. |
| `TRAVEL_MINUTES_PER_DISTANCE` | Module constant | Converts grid distance to minutes.  Increasing this constant creates wider travel buffers, which can squeeze out lower-priority suggestions but yields more realistic itineraries. |
| `tolerance` | `schedule_suggestions` arg | Softens floating‑point comparisons.  Tighten it to prevent near-equal fits; relax it if you encounter “no room” errors due to rounding noise. |

**Practical tip:** The interplay between `base_duration`, `max_multiples`, and `TRAVEL_MINUTES_PER_DISTANCE` is critical.  Large base blocks or long travel buffers can quickly consume a gap, so revisit those numbers whenever you notice excessive unused minutes or aggressive stretching.

---

### 4. Movement Log

Every stretch phase logs the travel segment that precedes it (`from_label -> to_label`), capturing:

- Which entity we travelled from (gap boundary or previous suggestion).
- The destination suggestion or gap end.
- Euclidean distance in grid units.
- Travel minutes derived from `TRAVEL_MINUTES_PER_DISTANCE`.

The notebook prints the log both in the formatted summary and in the demo cell so you can audit how the allocator traversed the grid.

---

### 5. Extending the Notebook

1. **Adjust Parameters Quickly**  
   Update the sample `Suggestion` definitions to try new scores, base durations, or maximum multiples.  Tweak `max_distance`, `permutation_limit`, and `TRAVEL_MINUTES_PER_DISTANCE` at the top of the code cell to explore different spatial constraints.

2. **Inject New Data**  
   Swap the sample lists for real inputs.  The only requirement is that every suggestion and gap carries coordinates on the shared (0–10) grid.

3. **Inspect Movement Trade-offs**  
   After each run, skim the movement log to see which hops dominate travel minutes.  That insight usually guides where to lower base durations or tighten the distance threshold.

4. **Add Post-Processing**  
   `ScheduleResult` exposes `allocated_minutes`, `movement_log`, and `dropped_suggestions`.  These can feed downstream dashboards or fairness checks without re-running the heavy permutation search.

---

### 6. Troubleshooting

- **Many suggestions dropped early:** Verify their locations are within `max_distance` of a gap boundary and that their `base_duration` fits after accounting for travel buffers.
- **Gaps left partially unused:** Review the movement log; travel time may be consuming most of the window.  Try reducing `max_multiples` for long-distance suggestions.
- **Permutation search explodes:** Lower `permutation_limit` or pre-group suggestions (e.g., by region) so each batch fits within the limit.
- **Unexpected travel spikes:** Ensure coordinates sit on the intended grid (0–10).  Outliers inflate travel conversions quickly.

---

Use this walkthrough as a companion when tweaking parameters or porting the logic into another runtime.  The notebook code mirrors everything described here, so you can follow along cell-by-cell as you experiment. 

