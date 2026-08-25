# 2026 Hyundai Palisade Hybrid SEL Premium (LX3/HDA1) — sunnypilot Port Changes Summary
# This summary was produced by Claude (Anthropic)

**Platform:** Comma 3X with Hyundai N harness  
**Base fork:** sunnypilot kamdeva lx3-hda1 branch  
**GitHub:** https://github.com/Knhusker/sunnypilot (master branch) / https://github.com/Knhusker/opendbc (lx3-palisade branch)

---

## Architecture Notes

- Steering via **CCNC** (0x161, 0x162 on ECAN bus 0). Openpilot feeds lane line data to stock camera; camera drives MDPS via its own LKAS_ALT on bus 1. Openpilot does NOT send LKAS_ALT directly.
- Bus layout: ECAN=0, ACAN=1, CAM=2
- SCC_CONTROL (0x1A0): Real traffic on bus 2 (camera-side); CANFD_CAMERA_SCC flag required
- Cruise buttons: CRUISE_BUTTONS_ALT (0x1AA, 426 decimal) — CANFD_ALT_BUTTONS auto-detected
- LDA button: 0x10B (BCM_BUTTON), bit 87
- Blinkers: LEFT=0x35a byte5 bit0, RIGHT=0x35b byte5 bit0

---

## Required Param Setting
On the Comma UI, enable ICBM.  

**Note:** ICBM does NOT take over cruise control. It only suppresses cancel commands that cause accelerator hunting. Verified from CAN logs — zero panda TX button presses during normal cruise operation.

---

## File Changes

### 1. `opendbc_repo/opendbc/car/hyundai/fingerprints.py`

Added firmware string for 2026 Palisade Hybrid:

```python
b'\xf1\x00LX31.001.001.002591000HKP_LX325_50919099211P9030'
```

---

### 2. `opendbc_repo/opendbc/car/hyundai/values.py`

- Harness: `CarHarness.hyundai_n`
- Flags: `HyundaiFlags.CANFD_ANGLE_STEERING | HyundaiFlags.CCNC | HyundaiFlags.CANFD_CAMERA_SCC`
- `STEER_THRESHOLD = 75` for `CANFD_ANGLE_STEERING` cars (reduced from 175 to reduce effort to initiate lane change)
- `MAX_ANGLE_RATE = 5` (degrees per frame, for angle steering cars)

---

### 3. `opendbc_repo/opendbc/car/hyundai/carstate.py`

- `lda_button` reads from `BCM_BUTTON["LDA_BTN"]` (address 0x10B, bit 87)
- `leftBlinker` reads from `LEFT_BLINKER_LX3["LEFT_BLINKER_LX3"]` (address 0x35a, byte 5 bit 0)
- `rightBlinker` reads from `RIGHT_BLINKER_LX3["RIGHT_BLINKER_LX3"]` (address 0x35b, byte 5 bit 0)
- Parser subscriptions added for LX3:
  - `("BCM_BUTTON", 25)`
  - `("LEFT_BLINKER_LX3", 20)`
  - `("RIGHT_BLINKER_LX3", 20)`
- `cruiseState.available` reads from `SCC_CONTROL["MainMode_ACC"]` on cam bus (via `update_mads_canfd`) due to `CANFD_CAMERA_SCC`

---

### 4. `opendbc_repo/opendbc/dbc/generator/hyundai/hyundai_canfd.dbc`

Three new messages appended:

```
BO_ 267 BCM_BUTTON: 16 XXX
 SG_ LDA_BTN : 87|1@1+ (1,0) [0|1] "" XXX

BO_ 858 LEFT_BLINKER_LX3: 16 XXX
 SG_ LEFT_BLINKER_LX3 : 40|1@1+ (1,0) [0|1] "" XXX

BO_ 859 RIGHT_BLINKER_LX3: 16 XXX
 SG_ RIGHT_BLINKER_LX3 : 40|1@1+ (1,0) [0|1] "" XXX
```

---

### 5. `opendbc_repo/opendbc/car/hyundai/hyundaicanfd.py`

For `CANFD_ANGLE_STEERING` cars in `create_steering_messages`:

- `LKA_RcgSta = 0` always (stock camera always sends 0; value of 3 caused periodic MDPS faults)
- `ADAS_StrAnglReqVal = apply_angle if lat_active else steering_angle` (tracks actual angle when not active, prevents jerk on engagement)
- `LKAS_ANGLE_ACTIVE = 1` always
- `LKA_SysIndReq = 2 if lat_active else 1`
- `LKAS_ALT` only sent to `CAN.ACAN` when `lat_active=True`
- Added `steering_angle=0.0` parameter to function signature
- `create_ccnc` guard for empty dicts on first frame

---

### 6. `opendbc_repo/opendbc/car/hyundai/carcontroller.py`

- `self.prev_lat_active = False` initialized in `__init__`
- `self.lat_reengagement_frames = 0` initialized in `__init__`
- `apply_steer_req = False` and `apply_torque = 0` initialized before conditional assignment
- Palisade LX3 excluded from `ANGLE_SAFETY_BASELINE_MODEL` check
- `apply_angle_last` updated on `not lat_active OR (lat_active AND not prev_lat_active)`
- `create_steering_messages` called with `CS.out.steeringAngleDeg` as additional argument
- `self.prev_lat_active = CC.latActive` updated at end of `update()`

**Re-engagement ramp** (prevents MDPS ACI fault cascade on curve re-engagement):

```python
# kcn - On re-engagement, ramp slowly from actual angle to prevent ACI fault cascade
if CC.latActive and not self.prev_lat_active:
    self.lat_reengagement_frames = 20  # 20 frames = 200ms ramp
    apply_angle = CS.out.steeringAngleDeg
    apply_steer_req = False
elif CC.latActive and (self.lat_reengagement_frames > 0 or 
                       (apply_angle is not None and abs(apply_angle - CS.out.steeringAngleDeg) > 2.0)):
    actual = CS.out.steeringAngleDeg
    model_target = apply_angle
    max_gap = 2.0
    if abs(actual) < abs(self.apply_angle_last):  # actual unwinding
        apply_angle = float(actual)
    else:
        apply_angle = float(np.clip(model_target, actual - max_gap, actual + max_gap))
    if self.lat_reengagement_frames > 0:
        self.lat_reengagement_frames -= 1
    gap = abs(model_target - actual)
    if gap < 1.0 and abs(actual - self.apply_angle_last) < 0.5:
        self.lat_reengagement_frames = 0
```

**Blinker-aware torque reduction gain** (reduces effort for lane change nudge vs within-lane correction):

```python
def compute_torque_reduction_gain(steering_torque, v_ego, lat_active, last_gain, 
                                   angle_steering=False, blinker_active=False):
```

- `angle_steering=True`: uses tuned breakpoints for angle-steering cars
- `blinker_active=True`: uses very low breakpoints for minimal resistance during lane change nudge

Called with:
```python
apply_torque = compute_torque_reduction_gain(
    CS.out.steeringTorque, v_ego_raw, CC.latActive, self.apply_torque_last,
    bool(self.CP.flags & HyundaiFlags.CANFD_ANGLE_STEERING),
    CC.leftBlinker or CC.rightBlinker
)
```

---

### 7. `selfdrive/car/car_specific.py`

Prevents `steerTempUnavailable` (hard softDisable) from escalating for angle-steering cars — always uses `steerTempUnavailableSilent` (warning only):

```python
if self.silent_steer_warning or CS.standstill or \
        self.steering_unpressed < int(1.5 / DT_CTRL) or \
        self.CP.steerControlType == structs.CarParams.SteerControlType.angle:  # kcn
    self.silent_steer_warning = True
    events.add(EventName.steerTempUnavailableSilent)
else:
    events.add(EventName.steerTempUnavailable)
```

---

### 8. `sunnypilot/mads/mads.py`

`lateral_mismatch_counter` resets when panda recovers `controlsAllowedLateral`, preventing brief dropouts from accumulating to the 200-frame threshold:

```python
if not self.active or self.selfdrive.enabled:
    self.lateral_mismatch_counter = 0
elif any(not ps.controlsAllowedLateral for ps in self.selfdrive.sm['pandaStates']
         if ps.safetyModel not in IGNORED_SAFETY_MODES):
    self.lateral_mismatch_counter += 1
else:  # kcn - reset counter when panda recovers
    self.lateral_mismatch_counter = 0
```

---

### 9. `selfdrive/locationd/calibrationd.py`

Changed `put_nonblocking` to `put` (API change in this version of openpilot):

```python
self.params.put("CalibrationParams", self.get_msg(True).to_bytes())
```

---

## Known Limitations

### LDA Button Alone (without cruise)
The LDA button correctly reads from BCM_BUTTON and MADS engages in Python, but the panda firmware monitors `0x1aa` bit 39 for the MADS button signal. The LX3's LDA button is on `0x10b` which the panda firmware doesn't monitor for this purpose. As a result, `controlsAllowedLateral` never goes True in the panda, causing `controlsMismatchLateral` immediate disable after ~2 seconds.

**Workaround:** Use cruise control + lane centering (the primary use case). Press cruise, then lane centering activates automatically.

**Fix requires:** Panda firmware modification to also accept `0x10b` bit 87 as a MADS button source.

### Camera ACI Angle Limit at Highway Speed
The stock camera's ACI (Automated Correction Interface) has a speed-dependent maximum steering angle. At approximately 79 mph, this limit is ~11.4°. On curves requiring more than this angle at that speed, the camera holds at its limit and the car drifts toward the outside of the curve. The re-engagement ramp fix prevents fault cascades but cannot exceed the hardware limit.

**Workaround:** Reduce speed before curves that require significant steering at highway speed.

---

## Tunable Parameters (in carcontroller.py)

### Torque Reduction Gain — Angle Steering Cars

Controls resistance when driver pushes against the wheel during lane centering:

```python
# Within-lane correction (no blinker):
floor = np.interp(v_ego, [2, 25], [0.1, 0.12])
bp1 = np.interp(v_ego, [2, 25], [60, 150])
bp2 = np.interp(v_ego, [2, 25], [120, 250])
bp3 = np.interp(v_ego, [2, 35], [160, 300])
bp4 = np.interp(v_ego, [2, 35], [200, 350])

# Lane change nudge (blinker active):
floor = np.interp(v_ego, [2, 25], [0.02, 0.04])
bp1 = np.interp(v_ego, [2, 35], [30, 50])
bp2 = np.interp(v_ego, [2, 35], [50, 70])
bp3 = np.interp(v_ego, [2, 35], [70, 90])
bp4 = np.interp(v_ego, [2, 35], [90, 120])
```

- `bp1-bp4`: steering torque thresholds (higher = more resistance before system yields)
- `floor`: minimum gain at max driver input (lower = system yields more)
- Speed range `[2, 35]` covers ~4.5 mph to ~78 mph

### Re-engagement Ramp Parameters

```python
self.lat_reengagement_frames = 20  # frames (200ms at 100Hz)
max_gap = 2.0  # maximum degrees commanded can exceed actual
```

- Increase frames for smoother re-engagement on tight curves
- Decrease max_gap for stricter gap limiting (may slow curve recovery)

---

## Install Instructions


*Generated by Claude: August 2026*  
*For 2026 Hyundai Palisade Hybrid SEL Premium (LX3/HDA1) on Comma 3X*
