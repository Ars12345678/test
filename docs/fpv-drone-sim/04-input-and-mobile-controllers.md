# 4. Input Architecture — USB Radios & Mobile Controllers

Input is routed through UE5's **Enhanced Input System**, which normalizes every source
(USB radio, gamepad, touch, gyro) into the same abstract axes the flight controller
consumes: `Throttle`, `Roll`, `Pitch`, `Yaw`, plus buttons (`Arm`, `ModeSwitch`,
`Beeper`, `TrimUp/Down`). The flight model never knows *which* device produced a stick
value — this is what lets a phone and a $300 radio fly the same sim.

```
  Physical devices ──▶ Device Adapters ──▶ Enhanced Input mappings ──▶ Normalized Axes ──▶ UDroneFlightController
   RadioMaster/TBS         RawInput            per-device curves          [-1..1]              (Rates → PID)
   Gamepad                 XInput/Gm             + deadzone/expo
   Touch gimbals           Slate/UMG
   Gyro (tilt)             Motion sensor
```

Core principle: **calibration + curves per device**, one shared control target.

---

## 4.1 USB FPV Controllers (RadioMaster, TBS Tango, etc.)

Real radios present to the OS as **USB HID joysticks** (when in "USB Joystick"/"Game
controller" mode) or as a serial gamepad. UE5 reads them via the **RawInput plugin**,
which exposes arbitrary HID axes beyond the standard gamepad set.

### Implementation
1. **Enable `RawInput` plugin**; register the radio's VID/PID with an axis map in
   `DefaultInput.ini` (RadioMaster and TBS Tango have known descriptors).
2. **Axis mapping:** radios output AETR/TAER channel order depending on the pilot's
   config — so expose a **channel-mapping UI** (assign which HID axis → Throttle/Roll/
   Pitch/Yaw) rather than hard-coding. Pilots expect this.
3. **Per-channel calibration:** capture endpoints + center, store min/mid/max, and apply
   the same **RC Rate / Super Rate / Expo** curve family as the flight model so the
   in-sim feel matches their real quad's rates.
4. **Latency:** RawInput is polled; poll at the input thread's max rate and feed the
   fixed-substep controller. Show an **on-screen latency meter** during setup.
5. **Failsafe & arming:** honor a real arm switch channel; a mode switch selects
   Acro/Level; support a "throttle-low + yaw" arm gesture as fallback.

### Data
```cpp
// Per-radio profile, saved to a UDataAsset the pilot can export/share.
USTRUCT() struct FRadioProfile
{
    FString      DeviceName;         // "RadioMaster TX16S", "TBS Tango 2"
    uint16       VID, PID;
    TMap<EAxis, FHidAxisBinding> ChannelMap; // which HID axis → which control axis
    FRatesCurve  Rates;              // RC rate / super rate / expo per axis
    FVector2D    Deadzone;
    FName        ArmChannel, ModeChannel;
};
```

---

## 4.2 Mobile Controllers

Mobile is a **first-class input profile**, not an afterthought. Three tiers, so a phone
newcomer and a serious pilot on the go are both served:

### Tier A — On-screen virtual gimbals (touch)
- Two **virtual sticks** rendered as gimbals; left/right stick layout follows the
  chosen radio **mode (Mode 1 / Mode 2)** — Mode 2 (throttle+yaw left, pitch+roll right)
  is the default.
- **Self-centering vs. non-centering throttle:** throttle stick stays where released
  (like a real radio), the other stick self-centers. This detail is what makes touch
  Acro survivable.
- **Assist ramp:** offer a **Leveled/Assist mode** (angle mode with self-leveling) for
  touch newcomers, with a clear path to full Acro as they improve. Assist is a control
  *mode*, layered on the same PID controller (angle setpoint instead of rate setpoint).
- Adjustable stick size/position/opacity; haptic tick at stick center.

### Tier B — Gyro / motion tilt
- Optional **device tilt → roll/pitch** using the phone's gyroscope; throttle + yaw stay
  on touch. Intuitive for casual play; toggle + sensitivity + recenter button.
- Fuse gyro with a complementary filter to avoid drift; clamp to sane rate limits.

### Tier C — Physical controllers on mobile
- **Bluetooth/USB-C gamepads** (Xbox, DualSense, mobile clip controllers) via UE5's
  standard gamepad path — same Enhanced Input axes, per-device expo/deadzone.
- **Real radios on mobile:** many radios (RadioMaster, TBS) enumerate as **USB
  joysticks over USB-C** or via an **ELRS/USB-C gamepad dongle**; route through the same
  RawInput-style adapter where the mobile OS permits HID access. This lets a pilot fly
  the mobile build with their actual transmitter.

### Mobile UX & performance notes
- Auto-detect device tier and pick a sensible default profile; let the player override.
- Because mobile also runs a **scaled render profile** (see roadmap Phase 7), keep the
  HUD/feed-degradation FX GPU-cheap — RF post-process gets a lighter mobile variant.
- Onboarding: a short "hover school" tutorial in Assist mode before unlocking Acro, since
  touch Acro has a brutal learning curve.

### Data
```cpp
USTRUCT() struct FMobileInputProfile
{
    EStickMode   Mode;               // Mode1 / Mode2
    bool         bGyroTilt;          // tilt → roll/pitch
    float        GyroSensitivity;
    EAssistLevel Assist;             // FullAcro / Assisted / Leveled
    FVector2D    LeftStickPos, RightStickPos, StickSize;
    float        Opacity;
    bool         bThrottleSelfCenter;
};
```

---

## 4.3 Shared Input Contract

Regardless of device, the flight controller receives the same struct each substep:

```cpp
struct FControlInput
{
    float Throttle;  // 0..1
    float Roll;      // -1..1  (rate or angle setpoint depending on Assist mode)
    float Pitch;     // -1..1
    float Yaw;       // -1..1
    bool  bArmed;
    EFlightMode Mode; // Acro / Assisted / Leveled
};
```

This clean seam is what makes the input matrix tractable: add a new device by writing one
adapter that outputs `FControlInput` — nothing downstream changes.
