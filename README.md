# Cleaning up a Realme Android TV (Changhong "AI PONT") with CleanYourTV

A write-up of what I did, plus a **revert guide** in case I (or my dad) want the stock TV back.

- **TV:** Realme Android TV, Changhong "AI PONT" platform, model shown as "AI Point"
- **OS:** Android TV 11, low-RAM device (MediaTek)
- **Tool:** [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv) (open source, no root, nothing uninstalled)
- **Computer:** Windows laptop, Git Bash, on the same network as the TV (TV on Ethernet)

> Redact your TV's IP address and any device names before sharing this publicly.

---

## TL;DR

1. Turned on USB debugging on the TV.
2. Connected over ADB from the laptop (`tools/tv.py --probe`).
3. Backed up the package list, then **disabled** (not uninstalled) 13 background/ad/preload packages in two small batches.
4. Tested after each batch (YouTube, HDMI switching, reboot).
5. Built and installed the Clean Your TV launcher, set it as Home, and disabled the stock Google TV launcher.
6. Result: a plain, Apple TV-style home screen. HDMI switching, streaming apps and playback all still work.

---

## What I did, step by step

### 1. Prepare the TV
Settings > Device Preferences > About > press OK on **Build** 7 times > back > Developer options > **USB debugging** on. The first connection shows an "Allow USB debugging?" prompt on the TV (tick *Always allow*).

### 2. Connect from the laptop
```bash
python -m venv tools/venv
tools/venv/Scripts/python.exe -m pip install -r tools/requirements.txt
export TV_HOST=<TV_IP>
tools/venv/Scripts/python.exe -I tools/tv.py --probe
```

### 3. Back up before touching anything
```bash
mkdir -p ~/tv-backup
tools/venv/Scripts/python.exe -I tools/tv.py 'pm list packages -f' > ~/tv-backup/packages.txt
tools/venv/Scripts/python.exe -I tools/tv.py 'pm list packages -d' > ~/tv-backup/already-disabled.txt
tools/venv/Scripts/python.exe -I tools/tv.py 'getprop' > ~/tv-backup/getprop.txt
```
`tools/disable.sh` also saves its own restore record in `~/.cleanyourtv/` before every run.

### 4. Debloat in two small batches
Lists use one package per line, `#` for comments. I based them on the repo's closest examples (`examples/chiq-ai-pont/`) and checked unfamiliar packages with `dumpsys package` first.

**`my-batch1.txt`** (ACR, ads, store and help apps)
```
tv.anoki.acr.controller                 # ACR, identifies what is on screen
com.cltv.fast                           # FAST ad channels
com.homwee.aipont                       # AI PONT brand portal
android.autoinstalls.config.homwee      # partner-app auto-installs
com.homwee.store                        # Changhong app store
com.homwee.help
com.homwee.howtocast
com.google.android.play.games
```

**`my-batch2.txt`** (after batch 1 tested fine)
```
com.google.android.katniss              # Assistant (the remote mic did not work anyway)
com.homwee.mmp.screenshare              # wireless casting; I use HDMI
com.homwee.multiscreenshare
com.google.android.backdrop             # ambient screensaver
```

Applied with:
```bash
export TV_HOST=<TV_IP>
tools/disable.sh my-batch1.txt
# test: YouTube plays, HDMI switches, reboot
tools/disable.sh my-batch2.txt
```

Kept on purpose: Netflix, Prime Video, JioHotstar, YouTube, YouTube Music, TV Browser, the firmware updater, Chromecast built-in, the MediaTek TV/input services, Play Store/Services, keyboard, Bluetooth and remote pairing.

### 5. Install the launcher
Needs JDK 17 and Android SDK platform 34 + build-tools 34.0.0 (via Android Studio's SDK Manager).

Dock set to **YouTube, Prime Video, JioHotstar, YouTube Music** by editing `launcher/app/src/main/res/values/config.xml` (`dock_apps`). Live TV stays hidden from the grid because the Inputs menu covers it.

```bash
export JAVA_HOME="/c/Program Files/Java/jdk-17"
echo "sdk.dir=C:/Users/<you>/AppData/Local/Android/Sdk" > launcher/local.properties
TV_HOST=<TV_IP> launcher/build-and-install.sh
```
The script installs the launcher, makes it Home, and only then disables the stock `com.google.android.tvlauncher`, so the TV is never left without a home screen.

### 6. Verified
- New home screen with the four-app dock
- Inputs menu switches between HDMI sources
- Streaming apps open and play
- Remote's Input button still works

*(Add your own before/after notes here: how the TV feels, boot time, anything you measured.)*

---

## REVERT GUIDE

Keep these files safe: `my-batch1.txt`, `my-batch2.txt`, and the folder `~/.cleanyourtv/` (the restore records, the only copy).

### Option A: Full revert from the laptop
```bash
export TV_HOST=<TV_IP>
tools/restore.sh my-batch2.txt
tools/restore.sh my-batch1.txt
```
This re-enables everything the batches disabled and gives Home back to the stock launcher (`com.google.android.tvlauncher`).

> **IP gotcha:** the restore records are saved per TV address. My TV's IP changed once (DHCP), so a record may be filed under the old IP. If `restore.sh` says it found no record, it falls back to re-enabling every package in the list, which is fine. Then run the manual commands below to be sure Home is back.

**Manual version (works even if the script misbehaves):**
```bash
PY=tools/venv/Scripts/python.exe
$PY -I tools/tv.py 'pm enable --user 0 com.google.android.tvlauncher'
$PY -I tools/tv.py 'cmd package set-home-activity --user 0 com.google.android.tvlauncher/.MainActivity'
```
Then re-enable the rest one by one if needed:
```bash
$PY -I tools/tv.py 'pm enable --user 0 <package>'
```

### Option B: Revert only the home screen (keep the debloat)
Run the two manual commands above. The Clean Your TV launcher stays installed but is no longer Home.

### Option C: No laptop, using only the TV remote
1. **Home screen:** on the Clean Your TV home, hold OK on **Settings** > **Launcher settings** > **Choose default home**, and pick the original launcher. (The original must be enabled first, see below.)
2. **Re-enable packages:** TV Settings > Apps > See all apps > show system apps > open any app marked *Disabled* > **Enable**. Start with *Android TV Home*.

### Option D: Remove the launcher app completely
After the stock home is back: TV Settings > Apps > **Clean Your TV** > Uninstall, or:
```bash
tools/venv/Scripts/python.exe -I tools/tv.py 'pm uninstall com.gathin.tvhome'
```

### Packages and how to bring them back individually
| Package | What it is |
|---|---|
| `tv.anoki.acr.controller` | ACR (content recognition) |
| `com.cltv.fast` | FAST ad channels (was already disabled on my TV) |
| `com.homwee.aipont` | AI PONT portal |
| `android.autoinstalls.config.homwee` | Preload installer |
| `com.homwee.store`, `com.homwee.help`, `com.homwee.howtocast` | Changhong store / help / cast tutorial |
| `com.google.android.play.games` | Play Games |
| `com.google.android.katniss` | Google Assistant |
| `com.homwee.mmp.screenshare`, `com.homwee.multiscreenshare` | Wireless screen sharing |
| `com.google.android.backdrop` | Ambient screensaver |
| `com.google.android.tvlauncher` | Stock home (disabled by the launcher install) |

---

## After a firmware update
An update can bring disabled packages back and reset Home. Nothing breaks permanently. To reapply only what drifted:
```bash
cat my-batch1.txt my-batch2.txt > my-debloat-all.txt
TV_HOST=<TV_IP> tools/venv/Scripts/python.exe -I tools/check.py my-debloat-all.txt        # report drift
TV_HOST=<TV_IP> tools/venv/Scripts/python.exe -I tools/check.py my-debloat-all.txt --fix  # reapply
```
I left the firmware updater enabled, so updates still arrive normally.

---

## Security: close ADB when you're done
Network ADB (port 5555) is unencrypted; anyone on the LAN could reach it while it's on.
1. TV: Developer options > turn off **USB debugging** (and any Network/ADB debugging), revoke USB debugging authorisations.
2. Reboot the TV.
3. Optional check: `TV_HOST=<TV_IP> tools/venv/Scripts/python.exe -I tools/adb-closed.py` (exit code 0 means closed).

**Note:** with debugging off, the laptop revert (Options A, B, D) needs it switched back on first. Options C works without it.

---

## Lessons learned (the useful bits)
- **An "unknown" TV model isn't a blocker.** The repo discovers bloat per device; related AI PONT lists were close enough to guide me.
- **Disable, don't uninstall**, in small batches, testing after each. Everything is reversible.
- **Phone app showed nothing for my model**; the CLI route worked fine.
- **Free-tier Gemini CLI** hit a usage limit before doing any real work. The repo's scripts need no AI, so I ran them by hand.
- **Git Bash vs PowerShell** syntax differences cost time (`$env:` vs `export`).
- **Android builds want JDK 17**, not the newest Java. Gradle on Windows needs a `sdk.dir` with a `C:/...` style path in `launcher/local.properties`.
- **DHCP changed my TV's IP** between steps. A router DHCP reservation avoids a confusing "connection timed out".
- `pm disable-user` can fail with "Shell cannot change component state" on packages that were already disabled by the vendor. Check `pm list packages -d` before worrying.

---

## Post draft for X
> Cleaned up my family's Realme Android TV (Changhong "AI PONT", Android TV 11) with the open-source CleanYourTV project.
> No root. Disabled ACR/ad/preload packages in 2 small batches, installed a lightweight Apple TV-style launcher. HDMI, YouTube and Prime all still work.
> Everything is reversible with one script. Full write-up and revert guide in the thread.
> Tool: github.com/Gathin23/cleanyourtv

---

*Credit: all tooling and the launcher are from [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv). I only ran it and documented my TV.*