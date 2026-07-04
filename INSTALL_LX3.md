# Installing sunnypilot on 2026 Hyundai Palisade Hybrid (LX3)

This branch (`lx3-hda1`) adds support for the **2026 Hyundai Palisade Hybrid SEL Premium and similar trims** without HDA II (LFA2 only). It's based on sunnypilot's `hkg-angle-steering-2025` development branch with LX3-specific patches.

**Status:** Lateral (steering) ENGAGED and tested on real hardware. Longitudinal not yet validated.

---

## What's working

- ✅ Vehicle fingerprint: `HYUNDAI_PALISADE_HEV_LX3`
- ✅ Lateral control (steering) — engages and steers the car
- ✅ MADS (Modular Automated Driving System)
- ✅ Panda safety accepting all CAN frames (canValid=True)
- ✅ Vehicle parameter learning (steerRatio, etc.)
- ✅ GPS, calibration, all locationd services

## What's NOT working yet

- ⚠️ Longitudinal control (Alpha Long) — code path exists but SCC_CONTROL bytes 24-25 mismatch the camera's format; needs reverse engineering. Use stock Hyundai ACC instead.
- ⚠️ NNLC model — falls back to MOCK (no trained neural-net lateral controller for LX3 yet)

---

## Quick install (on comma device)

```bash
# SSH into the comma device
ssh comma@<comma-ip> -i ~/.ssh/your_key

# Make root filesystem writable
sudo mount -o remount,rw /

# Remove any existing openpilot install
cd /data
sudo rm -rf openpilot

# Clone this branch with all submodules
git clone --recursive --branch lx3-hda1 https://github.com/kamdeva/openpilot.git

# Build everything
cd /data/openpilot
scons -j4

# Reboot
sudo reboot
```

After reboot, the comma device will run with the LX3-patched code. **Panda firmware will be flashed automatically by pandad on first boot** if the firmware on the panda is older than what's compiled in this branch.

---

## What this branch changes (the 5 patches)

1. **`msgq_repo/msgq/msgq.h`** — `NUM_READERS` bumped from 15 to 64.  
   *Root-cause fix for sunnypilot's `commIssue` cascade. sunnypilot has 36+ subscribers per high-frequency service (carState publishes at 95Hz with 36 subscribers); the upstream limit of 15 caused subscribers beyond the cap to receive stale data, which made every daemon's `sm.all_checks()` fail.*

2. **`opendbc_repo/opendbc/safety/modes/hyundai_canfd.h`** — On 0x105 ACCELERATOR_ALT, set `max_counter=0` and `ignore_counter=true`.  
   *LX3 transmits 0x105 with a counter that increments by 2 per frame instead of 1, which the panda's standard counter check rejects.*

3. **`opendbc_repo/opendbc/car/hyundai/carstate.py`** — Pre-register pt+cam parser messages and guard absent message accesses.  
   *LX3 doesn't have some messages that other Hyundai canfd platforms have (DOORS_SEATBELTS, BLINKERS, ADAS_CMD, HOD_FD). Without guards, carstate crashes on missing fields.*

4. **`selfdrive/locationd/calibrationd.py`** — Send `valid=True` when `calStatus == calibrated`.  
   *Workaround for `sm.all_checks()` failing in calibrationd due to msgq buffer overflow before the NUM_READERS fix landed. Kept for redundancy.*

5. **`selfdrive/locationd/locationd.py`** — Use `sm.all_alive()` instead of `sm.all_checks()` for filter init.  
   *Same workaround as above for locationd.*

---

## Vehicle setup checklist

Before first drive:

1. **Pair the device** at https://connect.comma.ai
2. **Fingerprint** should detect as `Hyundai Palisade Hybrid (without HDA II, LFA2) 2026`
3. **DO NOT enable Alpha Longitudinal** — it will silence the cruise buttons. Use stock ACC.
4. **Calibrate** by driving 5–10 minutes on highway above 25 mph

## Recommended sunnypilot settings for LX3

| Setting | Value | Why |
|---|---|---|
| Enable sunnypilot | ON | Master toggle |
| MADS Enabled | ON | Decoupled lateral steering works well |
| MADS Main Cruise Allowed | ON | Use cruise button to engage MADS |
| MADS Unified Engagement Mode | ON | Both lateral and longitudinal with one button |
| Alpha Longitudinal | **OFF** | Not validated for LX3, breaks cruise buttons |
| Experimental Mode | **OFF** | Requires Alpha Long |
| Dynamic Experimental Control | **OFF** | Requires Alpha Long |
| Driving Personality | Standard | Aggressive may cause sharp behavior on a freshly-ported car |

---

## Known issues

- **Stop-and-go**: When stock SCC drops out after ~3 seconds at full stop, you must tap Resume. This is stock Hyundai SCC behavior, not openpilot. Only openpilot longitudinal (which isn't working on LX3) can solve this.

- **NNLC MOCK**: Steering uses fallback controller, may feel less refined than cars with trained NNLC models. Will improve as paramsd learns vehicle dynamics over time.

- **`controlsAllowed: False` until cruise main pressed**: Normal — press main cruise button in your car to engage.

---

## Troubleshooting

### "openpilot does not engage, cruise buttons don't work"
You've probably enabled Alpha Longitudinal. Disable it:
```bash
ssh comma@<ip>
echo "0" > /data/params/d/AlphaLongitudinalEnabled
echo "0" > /data/params/d/ExperimentalMode
echo "0" > /data/params/d/DynamicExperimentalControl
sudo reboot
```

### "commIssue" alerts
Check `/data/openpilot/msgq_repo/msgq/msgq.h` for `NUM_READERS`. Should be 64. If 15, the msgq submodule isn't on the right commit:
```bash
cd /data/openpilot/msgq_repo
git fetch origin
git checkout lx3-hda1
cd /data/openpilot
scons -j4
sudo reboot
```

### "Unknown Vehicle Variant" / canError
Panda firmware needs reflashing. Manager should do this automatically; if not:
```bash
cd /data/openpilot/panda
# manual flash steps
```

---

## Repository layout

This is one of three linked repos:

- **kamdeva/openpilot** (this repo, branch `lx3-hda1`) — main openpilot fork with LX3 platform support
- **kamdeva/opendbc** (branch `lx3-hda1`) — opendbc fork with LX3 carstate + panda safety patches
- **kamdeva/msgq** (branch `lx3-hda1`) — msgq fork with NUM_READERS=64 fix

The submodules are linked via `.gitmodules` to point at the kamdeva forks.

---

## Tag

The known-good commit is tagged `lx3-working` in all three repos. To reset to it:

```bash
cd /data/openpilot
git fetch origin
git checkout lx3-working
git submodule update --recursive
scons -j4
```

---

## Credits

- **Upstream:** [sunnypilot/sunnypilot](https://github.com/sunnypilot/sunnypilot) — `hkg-angle-steering-2025-hda1` branch
- **Comma:** [commaai/openpilot](https://github.com/commaai/openpilot)
- **Port:** kam (kamdeva) for the LX3-specific patches and msgq buffer fix

---

## License

Same as upstream openpilot (MIT).
