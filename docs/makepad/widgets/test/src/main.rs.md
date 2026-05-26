# main.rs — WidgetTree correctness and performance test runner

**File path**: `widgets/test/src/main.rs` (439 lines)
**Core purpose**: Binary test harness for `WidgetTree` — validates tree construction, lookup, mutation under load, and benchmarks.

## Content
- **Mock infrastructure**: `MockHandle` and `MockWidget` implement `WidgetNode` with managed children and `skip_widget_tree_search` support for simulating subtree pruning
- **`basic_lookup_test`**: Creates a 3-level tree (root → dock → file_tree), verifies path lookups from root at each depth
- **`immediate_portal_item_lookup_test`**: Tests `find_within_from_borrowed` on a dynamically inserted portal child — the item must be resolvable immediately
- **`hammer_insert_and_lookup`**: Stress-tests 2000 iterations of replacing a portal's single child and immediately resolving the grandchild; periodically refreshes the portal's borrowed tree to exercise stale-node pruning
- **`dock_like_hammer`**: Simulates Studio's deep dock tree (8 levels) with repeated page-flip content replacement while verifying file_tree remains discoverable
- **`placeholder_root_survives_dirty_test`**: Verifies that a placeholder root (created via `refresh_from_borrowed` without a `WidgetRef`) retains its children after `mark_dirty` and also after upgrading via `seed_from_widget`
- **`benchmark_lookup_1000`**: Benchmarks 200k `find_within` lookups (ns/lookup) across 1000 siblings, comparing: (a) full scan, (b) subtree with `skip_widget_tree_search`, and (c) direct `HashMap` lookup; reports speedup ratios
- **`benchmark_dirty_mutation_bounds`**: Benchmarks 1000 iterations of `mark_dirty` + lookup to measure the cost of dirty-bound invalidation, with and without subtree skip
- Prints "widget_tree tests passed" on success
