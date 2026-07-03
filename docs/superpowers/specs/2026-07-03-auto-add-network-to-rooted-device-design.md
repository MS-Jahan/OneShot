# Auto-add cracked WiFi to rooted device + harden report writes

**Date:** 2026-07-03
**File touched:** `oneshot.py` (single file)

## Problem

On a successful WPS/Pixie attack, OneShot cracks a network and writes the
credentials to `reports/stored.txt` and `reports/stored.csv` (see
`__saveResult`, oneshot.py:1295). On a rooted Android device (Termux) the
operator still has to open WiFi settings and type the recovered PSK by hand to
actually use the network.

Two changes:

1. **Feature:** when running on a rooted device, automatically inject the
   cracked network (ESSID + WPA PSK) into Android's saved WiFi so the device
   can connect without manual entry.
2. **Hardening:** the existing report-file writes in `__saveResult` are
   unguarded — a disk-full or permission error raises and loses the crack.
   Wrap them so a write failure warns and continues.

## Non-goals (YAGNI)

- No new CLI flag. Injection runs automatically only when root is present;
  no root → a one-line note, no behavior change.
- No support for open / WEP / enterprise networks. WPA/WPA2 PSK only (that is
  what OneShot recovers).
- No direct editing of `wpa_supplicant.conf` / `WifiConfigStore.xml`
  (encrypted on modern Android — fragile, rejected).

## Approach

Injection uses Android's shell tooling through `su`:

- **Primary:** `cmd wifi connect-network "<ssid>" wpa2 "<psk>"` — Android 10+.
  Adds and connects in one call.
- **Fallback:** `wpa_cli` sequence (`add_network` / `set_network ssid` /
  `set_network psk` / `enable_network` / `save_config`) when `cmd wifi` is
  absent or fails.

Root is detected by running `su -c id` and checking for `uid=0`.

## Components

### New method: `__addNetworkToDevice(self, essid, wpa_psk)`

Placed on the same class as `__saveResult`. Behavior:

1. Return early if `essid` or `wpa_psk` is falsy.
2. Root check: run `su -c id`; if it fails or output lacks `uid=0`, print
   `[!] Root not available — skipping auto-add to device.` and return.
3. Build the command with **`shlex.quote`** applied to `essid` and `wpa_psk`.
   Credentials are attacker-recovered / untrusted input, so this is a
   shell-injection guard, not cosmetic.
4. Run the `cmd wifi connect-network` command via `su -c`. On non-zero exit or
   error output, fall back to the `wpa_cli` sequence.
5. Print `[+] Network '<essid>' added to device.` on success, `[!] ...` on
   failure.
6. The entire body is wrapped in `try/except Exception` — auto-add is a
   convenience and must never crash the attack flow.

`subprocess.run(..., capture_output=True, text=True, timeout=...)` with a short
timeout so a hung `su` prompt can't block the run.

### Call site

At the end of `__saveResult`, after the report files are written:

```
self.__addNetworkToDevice(essid, wpa_psk)
```

Flow becomes: save files → inject network to device.

### Hardening `__saveResult`

Wrap the two existing `open(...)` write blocks (`.txt` and `.csv`) each in
`try/except OSError`. On failure print
`[!] Failed to write <file>: <err>` and continue to the next block, so a
failure on one file does not lose the other or the device-injection step.

### Import

Add `import shlex` to the import block (top of oneshot.py, near the other
stdlib imports).

## Data flow

```
attack success
  -> __saveResult(bssid, essid, wps_pin, wpa_psk, lat, lon)
       -> write stored.txt   [try/except OSError]
       -> write stored.csv   [try/except OSError]
       -> __addNetworkToDevice(essid, wpa_psk)
            -> root check (su -c id)
            -> cmd wifi connect-network  (fallback: wpa_cli)   [try/except]
```

## Error handling

| Failure | Handling |
|---|---|
| No root | Note printed, return; attack unaffected |
| `cmd wifi` missing/fails | Fall back to `wpa_cli` |
| `wpa_cli` also fails | `[!]` warning, return |
| `su` hangs | `subprocess` timeout, treated as failure |
| Report write (disk/perm) | `[!]` warning per file, continue |
| Any unexpected exception in inject | Caught, `[!]` warning, attack flow continues |

## Testing

Manual (needs rooted Android/Termux — no CI hardware):

1. No-root host: run a crack path calling `__saveResult`; confirm note printed,
   files still written, no crash.
2. Simulate report write failure (read-only `reports/`): confirm `[!]` warning
   and that flow continues into device-injection.
3. Rooted device: confirm cracked network appears in saved WiFi and connects.
4. ESSID containing quotes/spaces/`;`: confirm `shlex.quote` prevents shell
   breakage.
