# Investigation: TalkBack's NOS navigation fix vs. SandFly's pivot handling

Date: 2026-11-11, corrected 2026-11-12

**Update (2026-11-12): the original verdict below was wrong.** It was reached by
code-reading alone, never confirmed on a device. Reproduced live on a Pixel 10 Pro
(`adb logcat`, diagnosis mode, NOS app, real finger swipes): the exact "map miss"
condition this note calls `OrderedTraversalController`'s pre-fix shape fired
repeatedly (`findNext(), can't find WorkingTree for AccessibilityNodeInfo`, 11
times in 0.7s) on a pivot that was otherwise perfectly valid and visible
(`refresh()` succeeded, text unchanged — not a "rebound", just absent from the
freshly-built `nodeTreeMap`). SandFly's `focus-after-pivot-left` fix never
engaged, because it only reacts when the pivot is *invalid* — it was valid here,
just missing from the map. See the "Confirmed finding" section below, which
replaces the original "Verdict".

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

## Original verdict (superseded, kept for the record)

**Not porting.** SandFly's existing `focus-after-pivot-left` fix covers the
reported scenario more precisely than TalkBack's generic viewport-edge fallback
(it carries scroll-direction/distance information that a plain bounds snapshot
doesn't), and the generic fallback is a no-op or redundant in the one path that
matters here. Changing the shared utility's `null` contract for the other, untested
callers is a real risk with no confirmed upside.

This reasoning missed a case: it assumed a map miss in `nodeTreeMap` always
coincides with `focusBeforeScroll.refresh()` failing or the node being invisible
(the branch `focus-after-pivot-left` actually covers). Live reproduction shows
that's false — the map miss and pivot invalidity are independent conditions.

## Confirmed finding (2026-11-12, on-device)

Reproduced in the NOS app on a Pixel 10 Pro, TalkBack diagnosis mode on,
`adb logcat` capturing `FocusProcessor-LogicalNav` and `OrderedTraversalCont`.
Opened an article, swiped forward repeatedly.

- After roughly 10 forward swipes, a burst of 11
  `findNext(), can't find WorkingTree for AccessibilityNodeInfo` warnings fired
  within 0.7 seconds, while temporary diagnostic logging confirmed the pivot
  (the same paragraph, byte-for-byte identical text before and after `refresh()`)
  was valid and visible both times `handleViewAutoScrolledForDirectionalNavigationWithDefaultTarget`
  ran — i.e. `validAccessibilityFocus` was `true`, so `focus-after-pivot-left`
  (the branch for an *invalid* pivot) never engaged at all. The map miss happened
  from inside the *valid*-pivot branch's call to `TraversalStrategyUtils.searchFocus`.
  Each retry re-entered `onDirectionalNavigationAction` hoping the next scroll
  event would expose the target; it never did. Reading went silent.
- The next manual swipe re-entered navigation through a different call path (no
  `FocusProcessor-LogicalNav` log at all this time), hit the same map miss once,
  and fell through to a coarse whole-window search that landed focus on the
  "Omhoog navigeren" (navigate up) floating button — confirming exactly the
  failure mode the original TalkBack fork commit describes: "a coarser
  whole-tree search that could land focus many items away without announcing
  anything in between."

Two independent call paths hit the same shared-utility map miss and failed two
different ways, which is why a call-site-only fix (mirroring
`focus-after-pivot-left`'s style) wouldn't have covered both. The fix was ported
into `OrderedTraversalController.findNext()`/`findPrevious()` itself, scoped
*only* to the `tree == null` ("not a key in `nodeTreeMap`") branch: fall back to
the node nearest the relevant edge of the visible viewport (topmost for forward,
bottommost for backward), exactly as the original TalkBack fork fix does. The
other `tree == null`-case return (reaching `tree.getNext()`/`getPrevious()` and
getting `null`, i.e. a genuine end-of-list) is untouched, which limits the
blast radius for the other callers (`FullScreenReadActor`, `SearchScreenOverlay`,
`NodePathDescription`, the list-edge check, `LocalTraversalOrder.kt`) that the
original analysis worried about — none of them depend on the "pivot missing
from the map" case specifically, only on genuine list-edge `null`s, which this
change doesn't alter.

Verified: unit tests (`testPhoneDebugUnitTest`, `testPhoneFullDebugUnitTest`)
pass except for a pre-existing, unrelated cluster
(`SettingsProfileStoreTest`, `SettingsProfilesTest`, `ReportInboxTest` — backup/
report-inbox code, confirmed failing identically on unmodified `master`). On
device, the exact repro that reliably failed twice no longer reproduces after
the fix.

Fix submitted as PR: https://github.com/aaron-gh/SandFly/pull/81
(branch `fix/nos-pivot-missing-from-traversal-map`, off a fresh `master` —
kept separate from this notes branch).

**Update (reworked after maintainer review):** the `OrderedTraversalController`
change above was reverted. The maintainer correctly pointed out that its
fallback returns the tree's own root almost every time (the root is always the
first `nodeTreeMap` entry, and its bounds span its whole subtree, so it always
has the most extreme top/bottom), not a node near the pivot — confirmed by
reading `createWorkingTree()`. The fix now lives entirely in
`FocusProcessorForLogicalNavigation.java`, anchored to the pivot's own live
position within the one local tree missing it, explicitly skipping that tree's
root entry, wired into two call sites confirmed by testing to hit this
(`handleViewAutoScrolledForDirectionalNavigationWithDefaultTarget` and a second,
independent path, `focusNextItemIfShown`, found only because testing reached
it). Re-verified on-device in both directions.

A separate, narrower skip also turned up in testing: one specific
image-caption element (a duplicated node pair at identical bounds, one of
class `ipc`) is consistently skipped in both directions, with no map-miss
warning preceding it — not this bug, left open for future investigation.

## Rebound recycled view (separate, still open)

Neither fix handles a *rebound recycled view*: a recycled `AccessibilityNodeInfo`
view coming back for a **different** item after a scroll. `refresh()` then
succeeds and the node is visible, so it's treated as a valid pivot even though its
on-screen identity has changed — a map **hit**, not a miss, so the new fallback
above doesn't fire either. SandFly's own code acknowledges this
(`FocusProcessorForLogicalNavigation.java:4952-4957`). Not implicated in the
confirmed finding above (the diagnostic logging showed `rebound=false` both
times), but still an open, separate risk for future investigation.
