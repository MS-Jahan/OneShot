# Review: OneShot WPS Tool — 3 Bug Fixes

## Original Prompt (for reference)
> 1. Why clean resources one after another (and then get error)?
> 2. WPS locked networks aren't showing (in red) sometimes.
> 3. Device busy error → trying after 5 seconds and then it works (missing sleep?).

---

## Blockers

None.

---

## Majors

### M1 — Retry loop does not log that it is retrying (iw_scanner, line 1735)
**Problem:** The retry loop for "Device or resource busy" silently sleeps and retries. The user gets no feedback that the scan is being retried, making it look like the tool is hanging for 2–4 seconds.
**Recommendation:** Add a print like `print("[!] Interface busy, retrying scan in 2s...")` inside the retry branch so the user knows what's happening.

### M2 — WPS locked regex fix may not be the root cause (line 1748)
**Problem:** The regex change from `(0x[0-9]+)` to `(0x[0-9a-fA-F]+)` is technically correct but unlikely to be the main reason WPS locked networks intermittently don't show in red. The `AP setup locked` field from `iw scan` is almost always `0x01` (locked) or `0x00` (unlocked) — both match the original regex fine. The more likely cause is that some APs **don't always advertise the `AP setup locked` field** in their WPS Information Element at all, especially in cached/partial scan results from `iw`. The current code defaults `"WPS locked": False` when the field is missing, so these networks silently appear white instead of red.
**Recommendation:** Keep the regex fix (it's a valid hardening), but acknowledge that the intermittent red-display issue is most likely caused by APs not reporting the field. A deeper fix could compare against a known-locked AP list or check the WPS State value (State 2 + configured can sometimes imply locked).

---

## Minors

### m1 — `_cleaned_up` flag is correct but not reset on re-use
**Problem:** The `_cleaned_up` flag (line 1596–1598) correctly prevents double cleanup. However, if for any future reason someone wanted to re-initialize a Companion object or reuse it, the flag would prevent cleanup from ever running again. This is a non-issue in the current code flow (Companion objects are never reused), but it's worth noting for future-proofing.
**Impact:** None today, but could mask bugs if code evolves.

### m2 — `cleanup()` catches too broadly in one path
The `auto_attack_mode` function (lines 1993–1996, 2038–2041) uses bare `except:` when calling `companion.cleanup()`. This swallows **all** exceptions including `KeyboardInterrupt` and `SystemExit`. Consider narrowing to `except Exception:` at minimum.

---

## Nits

### n1 — No user-visible message during scan retry
As noted in M1 — the silent `time.sleep(2)` + `continue` at line 1736 would benefit from a brief log. Low-effort improvement.

### n2 — Redundant `pass` after print in cleanup exception handler
Line 1608: `pass` after the `print(...)` in the except block is unnecessary. The function returns naturally. Pure style nit.

---

## Summary

The three changes address all three issues from the original prompt:

| Issue | Fix | Verdict |
|---|---|---|
| Double "Cleaning up resources…" + crash | `_cleaned_up` guard flag | ✅ Correct — prevents double free |
| WPS locked not red sometimes | Hex regex broadened | ⚠️ Hardening, but likely not root cause — intermittent missing field from APs is more probable |
| Device busy → 5s fallback | Retry loop with 2s sleep inside `iw_scanner` | ✅ Correct — handles busy gracefully |

### Next Actions
1. **Add a log message** inside the busy-retry loop (M1) — easy one-liner.
2. **Consider that the WPS locked intermittency** is probably unfixable at the `iw` parsing level (M2) — the data simply isn't in the scan output sometimes. The regex fix is still a valid improvement, just may not resolve the observed issue entirely.
3. Narrow the bare `except:` to `except Exception:` in `auto_attack_mode` (m2) — optional.
