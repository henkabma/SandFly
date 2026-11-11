# Investigation: TalkBack's NOS navigation fix vs. SandFly's pivot handling

Date: 2026-11-11

## Background

A personal fork of Google's TalkBack (`github.com/henkabma/talkback`, not part of
SandFly's lineage) carries a fix, commit `299ab2bb`, "Fix silent skip of multiple
nodes during linear navigation after large auto-scroll". Reproduced in the NOS news
app (`nl.nos.app`): a forward swipe from a paragraph could skip a dozen subsequent
paragraphs/headings, landing far down the screen, because
`OrderedTraversalController.findNext()`/`findPrevious()` returned `null` whenever the
pivot node (the node the user was focused on) was no longer present in the
freshly-rebuilt `nodeTreeMap` (e.g. its view got recycled after a large or
multi-step auto-scroll, including on paginated/infinite-scroll lists). Callers then
fell back to a coarser whole-tree search that could land focus many items away
without announcing anything in between.

The fix: when the pivot lookup misses, fall back to the node nearest the relevant
edge of the *current visible viewport* (topmost for forward, bottommost for
backward), found via `getBoundsInScreen()` over the nodes in `nodeTreeMap`.

This note records whether SandFly (built from Backtalk, from Google's upstream
TalkBack — a different lineage than the fork above) needs the same patch.

## Finding

SandFly's `OrderedTraversalController.findNext()`/`findPrevious()`
(`utils/src/main/java/com/google/android/accessibility/utils/traversal/OrderedTraversalController.java:331-357`)
are still in the pre-fix shape: a map miss just logs a warning and returns `null`.

However, SandFly's own navigation code no longer feeds a stale pivot into those
methods on the path that matters. Commit `7ae61ba71` ("focus-after-pivot-left")
already solves the NOS scenario at the call site, not in the shared utility:

- `FocusProcessorForLogicalNavigation.java:5267-5421` detects an invalid pivot after
  an auto-scroll and, instead of searching from the stale node, goes straight to
  `findFirstFocusInNodeTree` on the freshly scrolled content.
- For pivot loss during a manual swipe, `TraversalTreeCache.pivotAfterRemoval`
  (`TraversalTreeCache.kt:252-283`) picks a replacement using the last swipe
  direction, corrected by `ScrollAhead`'s remembered scroll deltas
  (`rememberScrolledAway`, `TraversalTreeCache.kt:210-216`; `ScrollAhead.java:221-227`)
  — i.e. it knows *how far and which way* the content moved, not just a bounds
  snapshot.

Eight related SandFly commits were checked
(`fd1ff9de5`, `d31036736`, `f958074df`, `61c302c92`, `1d45fb2c8`, `7ae61ba71`,
`cb9c095d3`, `cadd3b570`) plus `243b1a2f8` (ScrollAhead). Only `7ae61ba71` actually
prevents pivot eviction from causing a bad jump; the others are speed
improvements, scroll-distance tuning (which if anything *increases* exposure to
eviction, were it not handled), or unrelated web-content focus handling (web
descendants are never in `nodeTreeMap`,
`OrderedTraversalController.java:112-114`).

Tracing what TalkBack's generic fallback would actually return in SandFly's
window-rooted trees: forward, it resolves to the tree root, which behaves the same
as the `findFirstFocusInNodeTree` call SandFly already makes explicitly. Backward,
the same fallback resolves to the (non-focusable) root with nothing before it,
i.e. no better than today's `null`. So porting the fix as-is would be redundant for
the case it was built for, and would additionally change the `null` contract for
every other caller that reaches `findNext`/`findPrevious` through
`TraversalStrategyUtils.searchFocus` — `FullScreenReadActor`, `SearchScreenOverlay`,
`NodePathDescription`, the list-edge check (`TraversalStrategyUtils.java:324-330`),
and `LocalTraversalOrder.kt:225-229` — none of which have a confirmed bug report
tied to this.

## Verdict

**Not porting.** SandFly's existing `focus-after-pivot-left` fix covers the
reported scenario more precisely than TalkBack's generic viewport-edge fallback
(it carries scroll-direction/distance information that a plain bounds snapshot
doesn't), and the generic fallback is a no-op or redundant in the one path that
matters here. Changing the shared utility's `null` contract for the other, untested
callers is a real risk with no confirmed upside.

## Open gap (unrelated to the port question, present in both codebases)

Neither fix handles a *rebound recycled view*: a recycled `AccessibilityNodeInfo`
view coming back for a **different** item after a scroll. `refresh()` then
succeeds and the node is visible, so it's treated as a valid pivot even though its
on-screen identity has changed — a map **hit**, not a miss, so no fallback branch
in either implementation ever fires. SandFly's own code acknowledges this
(`FocusProcessorForLogicalNavigation.java:4952-4957`). This is the more interesting
residual risk and a candidate for future investigation, independent of the
TalkBack patch.

## Revisit this if

A concrete report surfaces of focus silently skipping content during continuous
reading (`FullScreenReadActor`) or screen search (`SearchScreenOverlay`) after a
large scroll — those are the callers not covered by `focus-after-pivot-left`. Check
logs for `"can't find WorkingTree"`
(`OrderedTraversalController.java:334`) firing on a real device; if it never fires,
the underlying condition isn't being hit in practice.
