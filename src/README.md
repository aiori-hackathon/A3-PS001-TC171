# A3-PS001-TC171: Android Ransomware Early-Warning Detector

A behaviour-based (UEBA-style) prototype that watches file activity inside an Android emulator, flags ransomware-like mass file renaming, and stops the offending app automatically.

---

## 1. Project Files

| File | Type | Role |
|------|------|------|
| `Red_Team_Simulator.apk` | Android app | Attack and benign workload generator (the "victim" app) |
| `ueba_hook.js` | Frida (JavaScript) | Sensor: intercepts file renames inside the app |
| `ueba_engine.py` | Frida (Python) | Brain: counts events, detects the burst, kills the app |

### 1.1 `Red_Team_Simulator.apk`

**Purpose:** Safely imitates ransomware and normal apps so the detector can be tested without real malware.

| Item | Detail |
|------|--------|
| App name / package | `RedTeam Simulator` / `com.redteam.simulator` |
| Minimum Android | API 26 |
| Setup | On launch it creates dummy files (`.txt`, `.csv`, `.pdf`, `.jpg`, `.tmp`) in its private `files` folder, including nested folders (`folder1-3`, `level1-3`) |
| Logging | Writes to Logcat under the tags `RedTeamSim` / `HackathonTests` |

**Buttons**

| Group | Button | Expected result |
|-------|--------|-----------------|
| Benign workloads (true negative) | Run Photo Backup Sync | Should **not** raise an alarm |
| | Run Document Auto-Save | Should **not** raise an alarm |
| Ransomware simulations (true positive) | [SIMULATE] Sequential Crypto-Locker | Should be detected and stopped |
| | [SIMULATE] Polymorphic Wiper | Should be detected and stopped |
| | [SIMULATE] Deep Tree Encryption | Should be detected and stopped |

### 1.2 `ueba_hook.js`

**Purpose:** Acts as the sensor. It is injected into the running app by Frida.

- Hooks `java.io.File.renameTo(File)` inside the Android runtime.
- Sends a **log** message for every rename (source path to destination path).
- Sends a **rename** message when the source path contains `Ransomware` or the destination ends in `.enc`.
- Always calls the original `renameTo`, so app behaviour is not changed.

### 1.3 `ueba_engine.py`

**Purpose:** Acts as the detection and response engine on the host machine.

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `THRESHOLD` | 10 | Number of events that counts as an attack |
| `TIME_WINDOW` | 2.0 s | Events must fall inside this sliding window |

**What it does**
1. Connects to the emulator over USB and attaches to `RedTeam Simulator`.
2. Loads `ueba_hook.js` into the app.
3. Receives each event, keeps a sliding 2-second window of timestamps.
4. When 10 or more events fall inside the window:
   - prints a **Metrics Report** (measured delay, files changed),
   - runs the **kill-switch**: `adb shell am force-stop com.redteam.simulator`,
   - exits the engine.

---

## 2. How It Works (Architecture)

```
Red Team Simulator app  --renameTo()-->  ueba_hook.js (Frida)
                                              |  send() events
                                              v
                                       ueba_engine.py
                          (10 events in 2 s?) --yes--> adb force-stop + report
```

---

## 3. Step-by-Step Demonstration (from the video)

### Phase A: Prepare the lab

| Step | Action | Command / Detail |
|------|--------|------------------|
| 1 | Start the emulator | `emulator -avd RansomwareLab -writable-system -no-snapshot-load -no-metrics -qemu -enable-kvm` |
| 2 | Wait for Android to boot | Home screen appears; `RedTeam Simulator` is installed and opens with the benign and ransomware buttons |
| 3 | Get root access | `adb root` |
| 4 | Relax SELinux | `adb shell setenforce 0` |
| 5 | Start the Frida server on the device | `adb shell "/data/local/tmp/frida-server &"` |

### Phase B: Review and launch the detector

| Step | Action | Command / Detail |
|------|--------|------------------|
| 6 | Show the sensor and engine code | `ueba_hook.js` and `ueba_engine.py` are opened and walked through |
| 7 | Activate the Python environment | `source ~/RansomwareEnv/bin/activate` |
| 8 | Open the app on the emulator | Keep `RedTeam Simulator` visible on screen (Frida attaches to the running app) |
| 9 | Start the engine | `python ueba_engine.py` prints `[*] Engine Active. Waiting for events...` |

### Phase C: Run the test

| Step | Action | Outcome |
|------|--------|---------|
| 10 | Tap **Run Photo Backup Sync** (benign) | Toast "Starting Photo Sync (Benign)"; no alarm |
| 11 | Tap a red **[SIMULATE]** ransomware button | App starts rapidly renaming files |
| 12 | Engine detects the burst | `[!!!] RANSOMWARE BEHAVIOR DETECTED [!!!]` appears |
| 13 | Metrics report is printed | Measured Delay: **63.306 s**, Files Changed: **10 files** |
| 14 | Kill-switch fires | `[+] Threat neutralized. Engine exiting.` |
| 15 | Result on the emulator | The app is force-stopped and the screen returns to the home screen |

---

## 4. Quick Start

```bash
# 1. Boot the emulator and install the app
emulator -avd RansomwareLab -writable-system -no-snapshot-load -no-metrics -qemu -enable-kvm
adb install Red_Team_Simulator.apk

# 2. Prepare the device
adb root
adb shell setenforce 0
adb shell "/data/local/tmp/frida-server &"

# 3. Open the app on the emulator, then run the engine
pip install frida frida-tools
python ueba_engine.py
```

**Requirements:** Android emulator (API 26+), `adb`, Frida server on the device (matching the host Frida version), Python 3 with `frida`.
Keep `ueba_engine.py` and `ueba_hook.js` in the same folder.

---

## 5. Troubleshooting

| Message | Fix |
|---------|-----|
| `Could not find 'com.redteam.simulator'` | Open the app and keep it visible before running the engine |
| `'ueba_hook.js' not found` | Place both files in the same directory |
| Frida cannot connect | Confirm `adb root` worked and `frida-server` is running |
