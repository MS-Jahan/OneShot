# Auto-add cracked WiFi to rooted device Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On a successful WPS crack, automatically inject the recovered network into a rooted Android device's saved WiFi, and harden the existing report-file writes so a disk error can't lose the crack.

**Architecture:** Add one private method `__addNetworkToDevice` to the `Companion` class (oneshot.py:1044). It detects root via `su -c id`, then uses `cmd wifi connect-network` (Android 10+) with a `wpa_cli` fallback, escaping the untrusted ESSID/PSK with `shlex.quote`. Wrap the two report-write blocks in `__saveResult` in `try/except OSError`, then call the new method at the end of `__saveResult`.

**Tech Stack:** Python 3 stdlib only (`subprocess`, `shlex`). Target runtime: Termux on rooted Android. No new dependencies.

## Global Constraints

- Single file touched: `oneshot.py`.
- Max line length: 120 (`.flake8`).
- Python 3 stdlib only — no new dependencies.
- WPA/WPA2 PSK networks only. No new CLI flag. Injection is automatic when root present.
- Untrusted input (attacker-recovered ESSID/PSK) MUST pass through `shlex.quote` before entering any shell string.
- No test framework in repo; verification is `flake8`, `python -c` import/parse checks, and documented manual device steps.

---

### Task 1: Harden `__saveResult` report writes + add `shlex` import

**Files:**
- Modify: `oneshot.py` (import block near line 3-21; `__saveResult` at oneshot.py:1295-1332)

**Interfaces:**
- Consumes: nothing new.
- Produces: no signature change to `__saveResult`. Adds module-level `import shlex` used by Task 2.

- [ ] **Step 1: Add the import**

In the import block at the top of `oneshot.py`, after `import csv` (line 18), add:

```python
import shlex
```

- [ ] **Step 2: Wrap the `.txt` write block in try/except**

Replace the current `.txt` block in `__saveResult` (oneshot.py:1307-1312):

```python
        with open(filename + ".txt", "a", encoding="utf-8") as file:
            file.write(f"{dateStr}\nBSSID: {bssid}\nESSID: {essid}\n")
            file.write(f"WPS PIN: {wps_pin}\nWPA PSK: {wpa_psk}\n")
            if latitude is not None and longitude is not None:
                file.write(f"Latitude: {latitude}\nLongitude: {longitude}\n")
            file.write("\n")
```

with:

```python
        try:
            with open(filename + ".txt", "a", encoding="utf-8") as file:
                file.write(f"{dateStr}\nBSSID: {bssid}\nESSID: {essid}\n")
                file.write(f"WPS PIN: {wps_pin}\nWPA PSK: {wpa_psk}\n")
                if latitude is not None and longitude is not None:
                    file.write(f"Latitude: {latitude}\nLongitude: {longitude}\n")
                file.write("\n")
        except OSError as e:
            print(f"[!] Failed to write {filename}.txt: {e}")
```

- [ ] **Step 3: Wrap the `.csv` write block in try/except**

Replace the current `.csv` block in `__saveResult` (oneshot.py:1314-1331):

```python
        writeTableHeader = not os.path.isfile(filename + ".csv")
        with open(filename + ".csv", "a", newline="", encoding="utf-8") as file:
            csvWriter = csv.writer(file, delimiter=";", quoting=csv.QUOTE_ALL)
            if writeTableHeader:
                csvWriter.writerow(
                    [
                        "Date",
                        "BSSID",
                        "ESSID",
                        "WPS PIN",
                        "WPA PSK",
                        "Latitude",
                        "Longitude",
                    ]
                )
            csvWriter.writerow(
                [dateStr, bssid, essid, wps_pin, wpa_psk, latitude, longitude]
            )
        print(f"[i] Credentials saved to {filename}.txt, {filename}.csv")
```

with:

```python
        try:
            writeTableHeader = not os.path.isfile(filename + ".csv")
            with open(filename + ".csv", "a", newline="", encoding="utf-8") as file:
                csvWriter = csv.writer(file, delimiter=";", quoting=csv.QUOTE_ALL)
                if writeTableHeader:
                    csvWriter.writerow(
                        [
                            "Date",
                            "BSSID",
                            "ESSID",
                            "WPS PIN",
                            "WPA PSK",
                            "Latitude",
                            "Longitude",
                        ]
                    )
                csvWriter.writerow(
                    [dateStr, bssid, essid, wps_pin, wpa_psk, latitude, longitude]
                )
            print(f"[i] Credentials saved to {filename}.txt, {filename}.csv")
        except OSError as e:
            print(f"[!] Failed to write {filename}.csv: {e}")
```

- [ ] **Step 4: Verify the file still parses and lints**

Run:
```bash
python -c "import ast; ast.parse(open('oneshot.py').read())" && flake8 oneshot.py
```
Expected: no output (parse OK, no lint errors).

- [ ] **Step 5: Commit**

```bash
git add oneshot.py
git commit -m "harden __saveResult report writes with try/except OSError; add shlex import"
```

---

### Task 2: Add `__addNetworkToDevice` and call it from `__saveResult`

**Files:**
- Modify: `oneshot.py` (add method to `Companion` class after `__saveResult`, ~oneshot.py:1333; add call at end of `__saveResult`)

**Interfaces:**
- Consumes: module-level `import shlex` (Task 1); `subprocess` (already imported, oneshot.py:4).
- Produces: `Companion.__addNetworkToDevice(self, essid, wpa_psk) -> None` — best-effort, never raises.

- [ ] **Step 1: Add the `__addNetworkToDevice` method**

Insert immediately after the end of `__saveResult` (before `def __savePin`, oneshot.py:1334):

```python
    def __addNetworkToDevice(self, essid, wpa_psk):
        """Best-effort: add a cracked WPA/WPA2 network to a rooted device's
        saved WiFi so it can connect without manual entry. Never raises."""
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

            q_essid = shlex.quote(essid)
            q_psk = shlex.quote(wpa_psk)

            # Primary: Android 10+ single-shot add + connect.
            connect_cmd = f"cmd wifi connect-network {q_essid} wpa2 {q_psk}"
            result = subprocess.run(
                ["su", "-c", connect_cmd],
                capture_output=True,
                text=True,
                timeout=30,
            )
            out = (result.stdout + result.stderr).lower()
            if result.returncode == 0 and "fail" not in out and "error" not in out:
                print(f"[+] Network '{essid}' added to device.")
                return

            # Fallback: wpa_cli sequence for devices without `cmd wifi`.
            wpa_cmd = (
                "id=$(wpa_cli add_network | tail -n1); "
                f"wpa_cli set_network $id ssid '\"'{q_essid}'\"'; "
                f"wpa_cli set_network $id psk '\"'{q_psk}'\"'; "
                "wpa_cli enable_network $id; "
                "wpa_cli save_config"
            )
            result = subprocess.run(
                ["su", "-c", wpa_cmd],
                capture_output=True,
                text=True,
                timeout=30,
            )
            if result.returncode == 0:
                print(f"[+] Network '{essid}' added to device (wpa_cli).")
            else:
                print(
                    "[!] Could not add network to device: "
                    f"{(result.stderr or result.stdout).strip()}"
                )
        except Exception as e:
            print(f"[!] Auto-add to device failed: {e}")
```

- [ ] **Step 2: Call it at the end of `__saveResult`**

At the very end of `__saveResult`, after the `.csv` try/except block from Task 1, add:

```python
        self.__addNetworkToDevice(essid, wpa_psk)
```

- [ ] **Step 3: Verify parse + lint**

Run:
```bash
python -c "import ast; ast.parse(open('oneshot.py').read())" && flake8 oneshot.py
```
Expected: no output.

- [ ] **Step 4: Verify shlex escaping guards injection (non-device unit check)**

Run:
```bash
python -c "import shlex; print(shlex.quote('evil\"; rm -rf / #'))"
```
Expected: output is single-quoted and inert, e.g. `'evil"; rm -rf / #'` — confirms ESSID/PSK with shell metacharacters cannot break out of the command string.

- [ ] **Step 5: Commit**

```bash
git add oneshot.py
git commit -m "feat: auto-add cracked WPA network to rooted device on save"
```

---

## Manual verification (rooted Android/Termux — no CI hardware)

Run after both tasks land, on a real device:

1. **No-root host:** trigger a save path; confirm `[!] Root not available` prints, `reports/stored.txt` + `.csv` still written, no crash.
2. **Read-only reports dir** (`chmod a-w reports`): confirm `[!] Failed to write ...` per file, flow continues to device-add step, no crash.
3. **Rooted device, real crack:** confirm the network appears in Android saved WiFi and connects.
4. **Adversarial ESSID** (contains `"`, space, `;`): confirm no shell breakage and the network name is stored intact.

## Self-Review notes

- Spec coverage: feature (Task 2), report hardening (Task 1), shlex import (Task 1 Step 1), shlex.quote guard (Task 2 Step 1/4), root gate (Task 2), fallback (Task 2), call site (Task 2 Step 2), manual tests (all 4 spec tests mapped). No gaps.
- No placeholders: all code blocks complete.
- Type consistency: `__addNetworkToDevice(self, essid, wpa_psk)` name used identically at definition and call site.
