# Self-Setting Mechanical Watch (e-Crown-style)

Research notes and a build plan for a mechanical watch that sets itself electronically,
like the **Ressence Type 2 e-Crown**. ("e-Crown" is a registered Ressence trademark, so
this project uses a different working name.)

---

## 1. What the Ressence e-Crown actually is

- **Base:** a modified off-the-shelf automatic movement. Sources disagree on whether it's
  an ETA 2824 or a 2892 derivative. It powers the ROCS rotating-disc display
  (Ressence Orbital Convex System) through the minute axle.
- **e-Crown module:** an electro-mechanical layer **sandwiched between the base movement
  and the display**. It has about 87 parts on a 4-layer, 0.25 mm flex PCB:
  - energy storage
  - a **micro motor and gearbox**
  - a **hand-position reader** (a sensor that reads where the display is pointing)
  - a setting mechanism for minutes and seconds
  - a mode selector, sensors, and a management MCU with Bluetooth
- **Power:** a kinetic generator, plus triple-junction PV cells behind 10 micro-shutters on
  the dial that open when the energy balance is low. The claimed budget is **about 1.8 J/day**.
  The module sleeps when the movement stops and wakes when the watch goes back on the wrist.
- **Key design rule:** it **never touches the going train** (barrel → escapement).
  Timekeeping stays 100% mechanical. The electronics only act like a robotic hand turning
  the crown.

### How the correction works
1. You set the time by hand (caseback lever). The module stores that as the **reference
   time** in its own clock (RTC).
2. At least once a day, the sensor reads what the display shows and compares it with the
   reference time (or with the phone's time over BLE).
3. If they differ, the motor applies the right amount of "crown rotation" to the motion
   works. The **slipping friction joint** (the same one that lets you set a normal watch
   while it runs) keeps that torque from reaching the going train.
4. Modes: *full* (set to the second from the phone), *semi* (to the minute, manual
   reference), *mechanical* (electronics off). It also supports time zones.

---

## 2. Fundamentally new movement, or minor changes?

**Mostly minor changes to the movement. The hard part is the add-on module.**

Every normal movement already has what this needs. The cannon pinion (or the equivalent
friction clutch) is a deliberate slip joint between the going train and the hands. That's
why turning the crown moves the hands without wrecking the escapement. The e-Crown just
**replaces your fingers with a motor**, entering at the motion works (minute wheel) the
same way the keyless works does.

So the movement changes are things like:
- a dial-side mounting plate / module interface (like a Dubois Dépraz complication module
  mounted on a 2824)
- an extended or modified minute-wheel arbor, or an extra intermediate wheel for the motor
  to mesh with
- optionally a hacking (stop-seconds) actuator, if you want to set **seconds**. Seconds
  live in the going train on the 4th wheel, with no slip joint, so you can't "turn" them.
  You can only stop the balance and wait. Minute-accurate is much easier.

What *is* genuinely new engineering (all in the module):

| Problem | Why it's hard | Candidate approaches |
|---|---|---|
| **Coupling / decoupling the motor** | A geared motor can't be back-driven at 1 rev/h. Left meshed, it would jam the motion works or slip the cannon pinion all the time. Even light drag costs balance amplitude. | Swinging/rocking pinion (like chronograph or auto-winding reversers): it engages when the motor spins and swings out when idle. Or a small engagement actuator (SMA wire, micro solenoid), or a magnetic coupling. **This is the #1 design problem.** |
| **Torque vs. size** | The motor must beat the cannon-pinion slip torque, through a gearbox only a few mm thick. | Lavet-type stepping motors from quartz watches (tiny, µJ pulses) driving a high-ratio wheel train. Bench version: 6 mm micro gearmotor. |
| **Reading hand position** | Needs an absolute reference, low power, no contact. | Optical: an LED plus photodiode through aligned holes in the wheels. This is how **radio-controlled analog watches (Junghans, Citizen, Casio) have done hand detection for decades**. Or a magnet plus a TMR/Hall sensor at µA current. |
| **Energy budget** | About 2 J/day is tiny. BLE plus the motor have to fit inside it. | A BLE SoC with µA sleep (nRF52/nRF54 class), a supercap or tiny Li cell, a PV cell, and duty-cycled sensing. |
| **Two-direction correction** | A watch running slow needs advancing; a fast one needs retarding. | A bidirectional motor plus a coupling that works both ways, or a "hack and wait" mode for small gains. |
| **Packaging** | Everything has to fit in about 1–2 mm of extra height under the dial. | This is where Ressence spent its years: a custom flex PCB and custom micro parts. |

---

## 3. Is it patented?

**Short answer:** Ressence calls its ROCS display "patented", and the e-Crown name is a
registered trademark. I couldn't pull up a specific e-Crown patent number from this
environment (the patent databases were unreachable), so **check this yourself before you
build anything you might sell**:

- Search **Espacenet** (worldwide.espacenet.com) and **WIPO Patentscope** for applicant
  `Ressence`. Look at the CH, EP, WO, and US publications in classes **G04B** (mechanical),
  **G04C** (electromechanical clocks), **G04R** (radio-controlled), and **G04G**.
- Read the **independent claims**, not the abstract. A claim only covers the specific
  combination it recites.

Things that work in your favor:
- **The general concept isn't new.** Self-setting analog displays with hand-position
  sensing and motor correction have existed for decades: radio-controlled watches,
  GPS/BLE analog watches like the Seiko Astron and Citizen, and hybrid smartwatches. Any
  Ressence patent is therefore likely to cover a *specific implementation* (for example,
  the module architecture inside a mechanical watch), not the idea itself.
- **Patents are territorial and last 20 years.** Filings from around 2017–2018 would run
  until about 2037–2038, but only in the countries where they were filed and kept in force.
- **Building one for study or personal use is low risk.** Making, selling, or offering
  it for sale is where infringement matters. Many European countries have a statutory
  experimental-use exemption. The US research exemption is very narrow (*Madey v. Duke*),
  but nobody realistically pursues a student demonstrator.
- If this ever turns commercial, get a **freedom-to-operate** opinion from a patent
  attorney. Your own design choices (coupling mechanism, sensing method) could be
  patentable in their own right.

---

## 4. Proposed build path

Don't start at wrist scale. Prove the physics big, then shrink it.

### Phase 0 — Measure (1–2 weeks)
- Get a few cheap movements: a **Seagull ST3600** (Unitas 6497 clone, 36.6 mm, hand-wound,
  roomy) and an **SW200 / PT5000** (2824 clones, 25.6 mm).
- With your MechE friends, measure the **cannon-pinion slip torque**: a torque gauge, or a
  lever arm with known weights on the minute-wheel arbor. That number sets the motor and
  gearbox spec.
- Measure balance amplitude on a timegrapher with and without extra drag on the motion
  works. That tells you how much load an idle coupling is allowed to add.

### Phase 1 — Bench demonstrator (about one semester)
- An ST3600 in a 3D-printed or machined fixture, with a micro gearmotor or stepper meshing
  with the minute wheel through a **swinging-pinion clutch** you design.
- Sensing: a reflective optical sensor or a hole-and-photodiode on the hour/minute wheel
  for an absolute 12 h index.
- Electronics: an nRF52 dev board with an RTC, BLE time sync from a phone, and closed-loop
  correction to the minute.
- **Success criteria:** the watch keeps running normally, the timegrapher shows no
  amplitude loss when idle, and it auto-corrects drift and time-zone changes.

### Phase 2 — Pocket-watch-sized module
- A custom flex or rigid-flex PCB, a Lavet stepper taken from a quartz movement, and a
  dial-side module plate on the 6497.
- Energy harvesting: a PV cell plus a supercap; measure real joules per day.
- Add the hacking actuator for to-the-second mode.

### Phase 3 — Wrist scale (stretch)
- Port it to an SW200-class movement under a dial module, within a total height budget of
  about 1.5–2 mm. This is CNC/EDM and micro-part territory, and where most of the real
  difficulty lives.

---

## 5. Difficulty, honestly

| Target | Difficulty for a student team |
|---|---|
| Bench demonstrator on a large movement | **Very achievable.** Standard MechE plus embedded skills. |
| Pocket-size integrated module | **Hard but doable.** It needs careful micro-mechanism design and access to good machining. |
| Wrist-size, production-like (Ressence level) | **Multi-year professional effort.** Custom micro motors, flex PCBs, Swiss part suppliers. |

The concept is simple. Miniaturization, low-power design, and a reliable clutch are where
the effort goes.

---

## Sources
- [Ressence — e-Crown page](https://ressencewatches.com/pages/e-crown)
- [Ressence — ROCS](https://ressencewatches.com/pages/rocs)
- [Quill & Pad — Type 2 e-Crown Concept](https://quillandpad.com/2018/02/18/ressence-type-2-e-crown-right-combination-right-time-videos/)
- [Quill & Pad — Type 2A/2G electronic-mechanical hybrid](https://quillandpad.com/2020/10/24/ressence-type-2a-2g-is-this-electronic-mechanical-hybrid-timepiece-the-future-of-mechanical-watchmaking/)
- [Monochrome — Type 2 e-Crown Concept](https://monochrome-watches.com/ressence-type-2-e-crown-concept-first-self-setting-mechanical-watch-sihh-2018/)
- [Deployant — Hands-on Type 2 e-Crown Concept](https://deployant.com/review-hands-ressence-type-2-e-crown-concept-dawn-new-era/)
- [Europa Star — Ressence reinvents the crown](https://www.europastar.com/the-watch-files/those-who-innovate/1004113548-ressence-reinvents-the-crown.html)
- [Loupiosity — Ressence Type 2 e-Crown](https://loupiosity.com/2018/02/ressence-type-2-e-crown/)
- [Fratello — Ressence Type 2 with e-Crown](https://fratellowatches.com/the-new-ressence-type-2-with-e-crown)
