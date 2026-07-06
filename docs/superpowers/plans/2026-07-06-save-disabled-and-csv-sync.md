# Save-but-don't-connect + CSV sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Change device injection so cracked networks are saved to the device but NOT auto-connected, and add a one-shot `--sync-to-device` command that imports every network already recorded in `reports/stored.csv` into the device's saved WiFi.

**Architecture:** Extract the device-injection logic from `Companion.__addNetworkToDevice` into **module-level functions** (`_device_has_network`, `add_network_to_device`) so they can be called both from inside `Companion.__saveResult` and from a new module-level `sync_stored_to_device(reports_dir)` — without instantiating `Companion` (whose `__init__` spawns a `wpa_supplicant` process, making it unsuitable for a sync-only code path). The inject function saves networks with `set_network $id disabled 1` so they don't auto-connect, and dedups via `wpa_cli list_networks`. A new `--sync-to-device` CLI flag, handled early in `main()`, calls `sync_stored_to_device` then exits before any scan.

**Tech Stack:** Python 3 stdlib (`subprocess`, `shlex`, `csv`, `os`). Target: Termux on rooted Android. No new dependencies.

## Global Constraints

- Single file touched: `oneshot.py`.
- Max line length: 120 (`.flake8`). Note: `flake8`/`pyflakes` are NOT installed in the dev env; verify via `python3 -m py_compile oneshot.py` + `awk 'NR!=2223 && length>120' oneshot.py` (line 2223 is pre-existing, outside our diff).
- Python 3 stdlib only.
- Untrusted input (ESSID/PSK from CSV or live crack) MUST pass through `shlex.quote` before entering any shell string. Existing pattern: `shlex.quote(f'"{value}"')` for wpa_cli args.
- All new `subprocess.run` calls MUST set `timeout=` and be wrapped in try/except so failures never crash the run.
- `cmd wifi connect-network` MUST NOT be used — it auto-connects, which this plan explicitly removes.
- Do NOT instantiate `Companion` for the sync path — its `__init__` (oneshot.py:1075) calls `self.__init_wpa_supplicant()` which spawns a wpa_supplicant process bound to a wireless interface.

---

### Task 1: Extract + rewrite device-injection as module functions (save-disabled + dedup)

**Files:**
- Modify: `oneshot.py` — replace instance method `Companion.__addNetworkToDevice` (oneshot.py:1343-1401) with two module-level functions; update the call site in `Companion.__saveResult` (oneshot.py:1341).

**Interfaces:**
- Consumes: module-level `subprocess`, `shlex` (already imported).
- Produces:
  - Module function `_device_has_network(essid: str) -> bool`
  - Module function `add_network_to_device(essid: str, wpa_psk: str) -> None`
  - These are used by Task 2's `sync_stored_to_device`.

- [ ] **Step 1: Add the two module-level functions**

Place them at module scope, immediately ABOVE `class Companion:` (oneshot.py:1045) — so they're defined before the class that uses them:

```python
def _device_has_network(essid):
    """Return True if a network with this SSID is already saved in the
    device's wpa_supplicant. Returns False on any error (treat as 'not
    present' so the caller attempts the add). Never raises."""
    try:
        result = subprocess.run(
            ["su", "-c", "wpa_cli list_networks"],
            capture_output=True,
            text=True,
            timeout=10,
        )
        if result.returncode != 0:
            return False
        # list_networks prints: network_id / ssid / bssid / flags (tab-separated).
        # Match the ssid column exactly so 'Net' doesn't false-match 'Netgear'.
        for line in result.stdout.splitlines():
            cols = line.split("\t")
            if len(cols) >= 2 and cols[1] == essid:
                return True
        return False
    except Exception:
        return False


def add_network_to_device(essid, wpa_psk):
    """Best-effort: save a cracked WPA/WPA2 network to a rooted device's
    saved WiFi WITHOUT auto-connecting. Network is added disabled so the
    user must enable it manually. Never raises."""
    if not essid or not wpa_psk:
        return
    try:
        root_check = subprocess.run(
            ["su", "-c", "id"],
            capture_output=True,
            text=True,
            timeout=10,
        )
        if root_check.returncode != 0 or "uid=0" not in root_check.stdout:
            print("[!] Root not available — skipping auto-add to device.")
            return

        if _device_has_network(essid):
            print(f"[i] Network '{essid}' already saved on device, skipping.")
            return

        # wpa_cli wants ssid/psk wrapped in literal double-quotes; quote the
        # whole double-quoted value so any char (incl. ' or ") stays inert.
        wpa_essid = shlex.quote(f'"{essid}"')
        wpa_psk_arg = shlex.quote(f'"{wpa_psk}"')
        # add_network + configure + DISABLE + save_config. disabled=1 means the
        # network is saved to wpa_supplicant.conf but will not be selected for
        # auto-connect until the user enables it.
        wpa_cmd = (
            "id=$(wpa_cli add_network | tail -n1); "
            f"wpa_cli set_network $id ssid {wpa_essid}; "
            f"wpa_cli set_network $id psk {wpa_psk_arg}; "
            "wpa_cli set_network $id disabled 1; "
            "wpa_cli save_config"
        )
        result = subprocess.run(
            ["su", "-c", wpa_cmd],
            capture_output=True,
            text=True,
            timeout=30,
        )
        if result.returncode == 0:
            print(
                f"[+] Network '{essid}' saved to device "
                "(disabled, no auto-connect)."
            )
        else:
            print(
                "[!] Could not add network to device: "
                f"{(result.stderr or result.stdout).strip()}"
            )
    except Exception as e:
        print(f"[!] Auto-add to device failed: {e}")
```

- [ ] **Step 2: Replace the call site + delete the old method**

In `Companion.__saveResult` (oneshot.py:1341), change:

```python
        self.__addNetworkToDevice(essid, wpa_psk)
```

to:

```python
        add_network_to_device(essid, wpa_psk)
```

Then **delete** the entire old `Companion.__addNetworkToDevice` method (oneshot.py:1343-1401, from its `def` line through the final `print(f"[!] Auto-add to device failed: {e}")` line). The method body's logic now lives in the module function above; nothing else in the file calls the old name (verify in Step 4).

- [ ] **Step 3: Verify compile + line length**

Run:
```bash
cd /home/ubuntu/projects/oneshot
python3 -m py_compile oneshot.py && echo "COMPILE_OK"
awk 'length>120 && NR!=2223 {print NR": "length}' oneshot.py
```
Expected: `COMPILE_OK`; no awk output.

- [ ] **Step 4: Verify no dangling references to the old method**

Run:
```bash
grep -n "__addNetworkToDevice\|connect-network\|add_network_to_device\|disabled 1" oneshot.py
```
Expected:
- `__addNetworkToDevice`: zero matches (old instance method fully removed).
- `connect-network`: zero matches (cmd wifi path removed).
- `add_network_to_device`: ≥2 matches (the `def` + the call in `__saveResult`).
- `disabled 1`: one match (in the new module function).

- [ ] **Step 5: Verify no-root path still works (no device required)**

Run:
```bash
cd /home/ubuntu/projects/oneshot
python3 -c "
import oneshot
# Call the module function directly with no root present; it must NOT raise
# and must print the 'Root not available' note (or succeed if running as root).
oneshot.add_network_to_device('TestSSID', 'TestPSK12345')
print('returned cleanly')
"
```
Expected: prints either `[!] Root not available — skipping auto-add to device.` or `[+] Network 'TestSSID' saved...`, then `returned cleanly`. No traceback. (If it actually adds a network on a rooted CI host, clean up the test SSID manually afterward — acceptable for verification.)

- [ ] **Step 6: Commit**

```bash
git add oneshot.py
git commit -m "refactor: extract device-injection to module fns; save disabled (no auto-connect); add SSID dedup"
```

---

### Task 2: Add `--sync-to-device` flag + CSV-driven bulk import

**Files:**
- Modify: `oneshot.py` — add module function `sync_stored_to_device`; add `--sync-to-device` argparse option; handle the flag in `main()` before scan starts.

**Interfaces:**
- Consumes: module function `add_network_to_device` (Task 1), module-level `csv` + `os` (already imported).
- Produces: module function `sync_stored_to_device(reports_dir: str) -> None`. CLI: `--sync-to-device` boolean flag; when set, program runs the sync then `sys.exit(0)` without scanning.

- [ ] **Step 1: Add `sync_stored_to_device` module function**

Place it immediately AFTER `add_network_to_device` (added in Task 1), still at module scope and above `class Companion:`:

```python
def sync_stored_to_device(reports_dir):
    """Read <reports_dir>/stored.csv and inject every saved network into the
    device that isn't already present. Skips rows with empty ESSID or WPA PSK.
    Never raises."""
    filename = reports_dir + "stored.csv"
    if not os.path.isfile(filename):
        print(f"[!] No stored credentials to sync ({filename} missing).")
        return
    processed = 0
    skipped = 0
    try:
        with open(
            filename, "r", newline="", encoding="utf-8", errors="replace"
        ) as file:
            csvReader = csv.reader(file, delimiter=";", quoting=csv.QUOTE_ALL)
            try:
                next(csvReader)  # skip header row
            except StopIteration:
                print("[i] stored.csv is empty, nothing to sync.")
                return
            for row in csvReader:
                if len(row) < 5:
                    continue
                # columns: Date, BSSID, ESSID, WPS PIN, WPA PSK, [Lat, Lon]
                essid = row[2]
                wpa_psk = row[4]
                if not essid or not wpa_psk:
                    skipped += 1
                    continue
                # add_network_to_device dedups internally via
                # _device_has_network, so re-running sync is idempotent.
                add_network_to_device(essid, wpa_psk)
                processed += 1
        print(
            f"[i] Sync complete: {processed} processed, "
            f"{skipped} skipped (missing ESSID/PSK)."
        )
    except OSError as e:
        print(f"[!] Failed to read {filename}: {e}")
```

- [ ] **Step 2: Add the `--sync-to-device` argparse option**

Find the existing `--write` argument definition (search `--write` in the argparse section, around oneshot.py:2225). After the `--write` block, add a new argument inside the same parser:

```python
        "--sync-to-device",
        action="store_true",
        default=False,
        help="Import all networks from reports/stored.csv into the device's "
        "saved WiFi (rooted only), then exit. Idempotent.",
    )
```

Match the indentation of the surrounding `add_argument` calls exactly — inspect the `--write` block first and mirror it.

- [ ] **Step 3: Handle the flag early in `main()`**

Find where `args = parser.parse_args()` runs. Immediately AFTER `args` is parsed and BEFORE any scan/interface setup, add:

```python
    if getattr(args, "sync_to_device", False):
        # One-shot CSV -> device import, then exit without scanning.
        reports_dir = (
            os.path.dirname(os.path.realpath(__file__)) + "/reports/"
        )
        sync_stored_to_device(reports_dir)
        sys.exit(0)
```

Note: `reports_dir` is computed the same way as in `Companion.__init__` (oneshot.py:1089) so the sync reads the same file the scanner writes. Do NOT instantiate `Companion` here.

- [ ] **Step 4: Verify compile + line length + flag presence**

Run:
```bash
cd /home/ubuntu/projects/oneshot
python3 -m py_compile oneshot.py && echo "COMPILE_OK"
awk 'length>120 && NR!=2223 {print NR": "length}' oneshot.py
python3 oneshot.py --help 2>&1 | grep -A1 "sync-to-device"
```
Expected: `COMPILE_OK`; no awk output; `--sync-to-device` appears in `--help` with its description.

- [ ] **Step 5: Verify no-root dry run**

Run:
```bash
cd /home/ubuntu/projects/oneshot
python3 oneshot.py --sync-to-device 2>&1 | head -20
echo "exit=$?"
```
Expected: either `[!] No stored credentials to sync (...missing).` if no CSV, or per-network `[!] Root not available — skipping auto-add to device.` lines + final `[i] Sync complete: ...`. Exit code 0, no scan starts.

- [ ] **Step 6: Verify with a real CSV (no root)**

Create a small test CSV to confirm the read path works end to end:

```bash
cd /home/ubuntu/projects/oneshot
mkdir -p reports
cat > reports/stored.csv <<'EOF'
"Date";"BSSID";"ESSID";"WPS PIN";"WPA PSK";"Latitude";"Longitude"
"01.01.2026 12:00";"AA:BB:CC:DD:EE:FF";"SyncTestSSID";"12345670";"testpassword123";"";""
EOF
python3 oneshot.py --sync-to-device 2>&1 | head -20
echo "exit=$?"
rm -f reports/stored.csv
```
Expected: `[!] Root not available — skipping auto-add to device.` for the `SyncTestSSID` row, then `[i] Sync complete: 1 processed, 0 skipped.`, exit 0. (Restore or leave deleted per repo state — `stored.csv` is gitignored scratch, but check `git status` after.)

- [ ] **Step 7: Commit**

```bash
git add oneshot.py
git commit -m "feat: add --sync-to-device flag to import stored.csv networks to device"
```

---

## Manual verification (rooted Android/Termux)

1. **Live crack, save-but-don't-connect:** trigger a crack on a rooted device. Confirm the network appears in Android WiFi settings as saved but is NOT connected automatically; user must tap to connect.
2. **Idempotent re-run:** after a crack, run `python3 oneshot.py --sync-to-device`. Confirm `[i] Network '<ssid>' already saved on device, skipping.` for the just-cracked one — proves dedup works.
3. **CSV import:** with `reports/stored.csv` populated and root available, run `python3 oneshot.py --sync-to-device`. Confirm every row with non-empty ESSID+PSK appears saved-but-disabled on device, and `[i] Sync complete: N processed, 0 skipped.`
4. **Adversarial ESSID** (contains `"`, `'`, `;`): confirm no shell breakage; network saved with name intact.
5. **Repeated import safety:** run `--sync-to-device` twice. Second run must not create duplicate saved entries — every row says "already saved, skipping."

## Caveats (known limitations, not blockers)

- `set_network $id disabled 1` is the wpa_supplicant-level disable. On some Android versions the framework's `WifiConfigStore` may re-enable a saved network on the next WiFi toggle. If observed, follow-up work should investigate Android's `autojoin` framework flag — out of scope here.
- Dedup matches on SSID only (BSSID in `wpa_cli list_networks` is `00:00:00:00:00:00` when not associated, so unusable as a key). Two different APs broadcasting the same SSID with different PSKs (rare) will collide: the second is skipped. Acceptable for a password-recovery tool.

## Self-Review notes

- Spec coverage: save-without-auto-connect (Task 1 Step 1 `disabled 1`), dedup (Task 1 Step 1 `_device_has_network` + guard), CSV import (Task 2 Step 1), CLI trigger (Task 2 Steps 2-3), repeated-add safety (dedup + manual test 5). Files-still-saved requirement: untouched — both tasks only modify device-injection code + add a CLI flag; `Companion.__saveResult` file writes (oneshot.py:1308-1339) remain exactly as shipped.
- No placeholders: all code blocks complete. The earlier draft's "investigate Companion.__init__ and choose a path" is resolved — module functions avoid instantiation entirely (called out in Global Constraints + Architecture).
- Type consistency: `add_network_to_device(essid, wpa_psk)` and `_device_has_network(essid)` names match between Task 1 definitions and Task 2's `sync_stored_to_device` calls.
- Name-mangling trap avoided: by extracting to module functions, no private-method call from outside the class exists.
