# Investigation Summary: isolate Switch Logic

Date: 2026-05-11

## 1. Overview

This summary captures the investigation into the `isolate` switch on the MTK/OpenWrt wireless stack. It is written for the next developer who needs to resume the work without replaying the entire debugging session.

The main questions are:

1. What did the user actually observe?
2. Which parts have already been explained and fixed?
3. What is still broken?
4. Which hypotheses have already been ruled out or weakened?
5. Where should the investigation continue?

## 2. Current Status at a Glance

If only one section is read, it should be this one.

1. The original report combined two different problems.
2. The `isolate -> NoForwarding` path has already been traced, fixed, and validated.
3. The remaining issue is that `hairpin_mode` often lags one reload behind the intended `isolate` state.
4. The current evidence points more strongly to reload sequencing or missing final resynchronization than to `mtwifi_cfg` failing to detect `isolate` changes.

## 3. What the User Originally Saw

The user initially reported two symptoms at the same time:

1. Enabling `isolate` did not make the Linux bridge port `hairpin_mode` follow the expected state. At times it even looked inverted.
2. The MTK DAT field `NoForwarding` also appeared inverted relative to `isolate`.

Those symptoms looked like one bug, but they turned out to come from two different paths:

1. The `NoForwarding` side was affected by stale or synthesized `isolate` reaching the MTK shell/Lua path.
2. The `hairpin_mode` side remained even after `NoForwarding` was fixed: the bridge port state still did not reliably land in the correct final state on the same reload.

The rest of the investigation only makes sense if these are treated as separate problems.

## 4. Current Reproduction Pattern

The remaining bug is now much clearer than it was at the beginning.

1. When only the `isolate` option is changed, `mtwifi_cfg` sees the change and `NoForwarding` follows it.
2. Even so, after that reload completes, `hairpin_mode` may still keep the old value.
3. On the next reload, `hairpin_mode` usually catches up to the already-correct `isolate` state.
4. The second reload does not necessarily need another `isolate` change. Changing any other wireless option can also make `hairpin_mode` catch up.

In other words, the remaining bug no longer looks like an inverted value. It looks like a one-reload-late synchronization problem.

## 5. Control Path

The relevant MTK/OpenWrt control path is:

1. The user changes wireless configuration and runs `wifi reload <device>`.
2. The OpenWrt `wifi` script forwards the request to `network.wireless` and netifd.
3. netifd calls the MTK driver shell entrypoint `drv_mtwifi_setup()`.
4. `drv_mtwifi_setup()` assembles JSON and invokes `/sbin/mtwifi_cfg setup`.
5. `mtwifi_cfg` translates the input into DAT updates and triggers MTK-specific runtime actions such as interface down/up or broader reload behavior.
6. netifd then continues its own wireless bring-up and bridge-side synchronization.

One point needed repeated clarification during the investigation:

1. `wifi reload MT7981_1_2` is the outer OpenWrt reload command.
2. `mtwifi_cfg setup` is only an internal step inside `drv_mtwifi_setup()`.

They are not competing mechanisms. `mtwifi_cfg setup` is part of the path executed during `wifi reload`.

Relevant files:

- `package/network/config/wifi-scripts/files/sbin/wifi`
- `package/network/config/wifi-scripts/files/lib/netifd/netifd-wireless.sh`
- `package/mtk/applications/mtwifi-cfg/files/netifd/mtwifi.sh`
- `package/mtk/applications/mtwifi-cfg/files/mtwifi-cfg/mtwifi_cfg`

## 6. Resolved Issue: Why `NoForwarding` Used to Look Wrong

This was the first root cause that had to be separated from the remaining `hairpin_mode` problem.

### 6.1 What Was Happening

Even after `isolate` had been removed from UCI, the MTK side could still see an effective `isolate=true`. That made the DAT field `NoForwarding` look inverted or stale.

This is why the original report sounded like "`NoForwarding` is logically reversed relative to `isolate`".

### 6.2 Root Cause

The root cause was not the DAT meaning of `NoForwarding`. It was the standard shell helper path.

The stock `for_each_interface()` in `netifd-wireless.sh` calls `_wireless_set_brsnoop_isolation()`. Under some bridge conditions, that helper synthesizes `isolate=1` even when UCI does not explicitly contain `isolate`.

As a result, the MTK driver script was not always seeing the user-authored value of `isolate`. In some cases it was seeing a helper-generated value instead.

### 6.3 Fix

The completed fix was:

1. Stop using the stock `for_each_interface()` in `package/mtk/applications/mtwifi-cfg/files/netifd/mtwifi.sh`.
2. Replace it with the local `mtwifi_for_each_interface()`.
3. Avoid the helper that synthesizes `isolate=1`.

### 6.4 Validation

After that change:

1. `isolate=true` makes `NoForwarding` change from `0` to `1`.
2. Removing `isolate` makes `NoForwarding` change from `1` to `0`.

That behavior was confirmed in `mtwifi.log`.

At this point, `isolate -> NoForwarding` should be considered fixed. The remaining work should not restart from the assumption that `NoForwarding` is still being computed incorrectly.

Relevant files:

- `package/network/config/wifi-scripts/files/lib/netifd/netifd-wireless.sh`
- `package/mtk/applications/mtwifi-cfg/files/netifd/mtwifi.sh`
- `package/mtk/applications/mtwifi-cfg/files/mtwifi-cfg/mtwifi_cfg`

## 7. Findings That Are Now Fairly Solid

### 7.1 netifd Does Not Implement the Main Hairpin Logic Backwards

The base OpenWrt/netifd logic is correct.

Source inspection confirmed that:

1. `wireless.c` parses `isolate` into `wireless_isolate`.
2. `system-linux.c` uses that value when configuring wireless bridge port attributes.
3. When `wireless_isolate=true`, netifd sets `hairpin_mode=0`.

So the statement "netifd itself writes hairpin backwards" is not supported by the source.

### 7.2 Moving `json_dump | /sbin/mtwifi_cfg setup` Is Not the Real Fix

That line reordering was investigated and is not the meaningful change.

The effective change was the custom iterator, not the position of the `mtwifi_cfg setup` line.

Why:

1. `mtwifi_cfg setup` reads the JSON already assembled by the driver shell script.
2. The later `wireless_add_vif` calls use a separate notify path.
3. That notify path only updates interface-side metadata such as `ifname` in netifd. It does not modify the JSON that was already passed into `mtwifi_cfg`.

So line reordering is not the reason the `NoForwarding` issue disappeared, and it is not a convincing explanation for the remaining `hairpin_mode` issue.

### 7.3 Encryption Is No Longer the Best Primary Explanation

At one stage, `sae` or `sae-mixed` looked like the likely trigger because some runs correlated with encryption changes.

More testing weakened that theory.

Encryption may still influence which runtime path is taken, but it is not the most stable explanation for the remaining bug. The more stable pattern is that changing only `isolate` often does not update `hairpin_mode` immediately, while the next arbitrary reload usually makes it catch up.

Encryption should therefore be treated as a secondary correlation, not the main stage conclusion.

## 8. What the Logs Already Prove

The current `mtwifi.log` evidence proves at least three things:

1. `mtwifi_cfg` does see `isolate` changes.
2. Those changes do produce DAT diffs, specifically through `NoForwarding`.
3. Other settings also produce DAT diffs, for example `ieee80211r` through `FtSupport`, and encryption through `AuthMode`, `PMFMFPR`, and related fields.

This matters because it weakens a simple hypothesis that was natural early on:

"Maybe an isolate-only change never shows up in `mtwifi_cfg` diff logic, so no relevant reload happens."

The logs do not support that simple explanation. At the DAT layer, `isolate` changes are clearly visible.

That shifts suspicion away from "did the diff exist at all?" and toward "what happens after the diff is detected, and in what order?"

## 9. Best Current Explanation

The strongest current explanation is:

1. `isolate` now reaches the MTK control path correctly.
2. `NoForwarding` now tracks it correctly.
3. Even so, the same reload may still leave the bridge port `hairpin_mode` at the old value.
4. The next reload, or another unrelated configuration change that causes a new runtime cycle, finally brings `hairpin_mode` into sync.

That makes the remaining issue look much more like reload sequencing or missing final resynchronization than like a pure diff-detection failure.

## 10. Most Suspicious Locations

### 10.1 `mtwifi_cfg` Uses a Coarse Diff-to-Restart Policy

`mtwifi_cfg` does more than write DAT files. It also decides which MTK runtime actions to trigger based on `cfg_diff`.

On the DBDC path, any non-empty `cfg_diff` can lead to broad interface down/up behavior rather than a narrowly scoped action for the field that changed.

This means:

1. An isolate-only change is already enough to trigger runtime work, because it produces a `NoForwarding` diff.
2. That runtime work may still be too coarse or badly ordered to guarantee correct bridge-side state by the end of the same reload.

### 10.2 `mtwifi_down()` / `mtwifi_up()` Do Not Explicitly Restore Bridge Hairpin State

The current MTK runtime path is mostly:

1. interfaces down
2. interfaces up
3. `startwapp.sh`

It does not explicitly reapply bridge port `hairpin_mode`.

So even if the MTK reload path definitely ran, that alone does not prove the final bridge state should be correct at the end of the cycle.

### 10.3 The Same-Cycle Ordering Between MTK Runtime Actions and netifd Bridge Sync Is Highly Suspect

This is now one of the best places to continue.

Two possibilities are especially plausible:

1. The MTK down/up sequence disrupts bridge state first, and netifd does not perform one final bridge resync at the very end of the cycle.
2. netifd does synchronize the bridge state, but a later runtime action recreates the bridge membership or interface state and overwrites the correct result.

Either version matches the current symptom:

1. The first reload changes the logical state but leaves `hairpin_mode` stale.
2. The second reload, or a later unrelated change, makes the state catch up.

### 10.4 hostapd or Later BSS Recreation Is Still a Secondary Suspect

The hostapd `ucode` path can use configuration hashing to decide when to remove and re-add BSS or interface state.

That means there may still be a later runtime path after the MTK logic that perturbs bridge membership again.

This has not been proven to be the decisive last writer, but it remains a legitimate candidate and should stay on the table.

## 11. Conclusions the Next Developer Can Inherit

The following points should be treated as established stage conclusions:

1. The original report combined two different problems.
2. The old `NoForwarding` problem came from synthesized `isolate=1` in the standard shell helper path.
3. That problem was fixed by switching `mtwifi.sh` to the local `mtwifi_for_each_interface()`.
4. `isolate -> NoForwarding` is now behaving correctly.
5. The remaining issue is that `hairpin_mode` often lags one reload behind the intended `isolate` state.
6. The current logs do not support the simple theory that `mtwifi_cfg` completely misses isolate-only diffs.
7. The strongest remaining suspicion is same-cycle sequencing, restart granularity, or a later overwrite of bridge state.

## 12. What Is Still Unresolved

The following points are still open and should not be treated as proven facts:

1. Whether the decisive failure happens inside the MTK restart path or after it.
2. Whether netifd bridge synchronization runs too early and later runtime actions overwrite it.
3. Whether hostapd or another runtime component rebuilds bridge-side state after netifd has already synchronized it.
4. Whether the precise bug should be described as a missing final resync in the same cycle or as a later overwrite within the same cycle.

## 13. Where to Continue the Investigation

The next round of work should not restart from "did `isolate` get passed in at all?" That question has effectively been answered.

The best places to continue are:

1. `/sbin/mtwifi_cfg setup` as invoked from `drv_mtwifi_setup()`.
2. The diff handling and `mtwifi_down()` / `mtwifi_up()` sequence inside `mtwifi_cfg`.
3. The netifd bridge-side synchronization that happens after `wireless_set_up`.
4. Any later BSS or bridge-member recreation path that can run after netifd thinks setup is complete.

One-line summary of the current state:

The active bug is no longer about the meaning of `isolate`. It is about why the same reload that makes the correct logical isolate decision does not always leave the final runtime `hairpin_mode` in the matching state.
