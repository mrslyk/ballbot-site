# Ballbot — Version 1 Design Document

**A solar-assisted robot that finds tennis balls on a court, collects them into a hopper, and returns to its dock.**

Prepared for: Tim Parsa & Kevin Eisenbacher
Date: July 17, 2026
Status: v1 concept — scoping draft

---

## 1. Product scope for v1

The temptation with Ballbot is to build the dream machine. Kevin's longer-term wishlist — an onboard serving gun, a squeegee for wet courts, a peat/clay floor conditioner, a leaf-blower to clear debris — is a great north star, but every one of those is a separate mechanism with its own failure modes. Version 1 does exactly one job well:

> **Autonomously find tennis balls scattered on a single court, pick them up into an onboard hopper, and return to a corner dock to recharge and dump.**

Everything else is explicitly deferred. Getting the "find → pick up → return" loop reliable is the whole ballgame for v1; it's also the foundation every later feature bolts onto.

**Deferred to later versions (intentionally):** serving gun, court squeegee, floor conditioning, debris blower, and court-to-court travel.

---

## 2. Reference dimensions (design constraints)

These fixed facts drive the whole chassis and intake design:

- **Tennis ball:** 6.54–6.86 cm (2.57–2.70 in) diameter, ITF regulation. Optic yellow. ~57 g, and it compresses slightly under load — which we exploit in the pickup mechanism.
- **Court:** 78 ft long × 27 ft wide (singles) / 36 ft wide (doubles). A known, flat, bounded rectangle with high-contrast white lines.
- **Net:** 3 ft 6 in high at the posts, dipping to 3 ft 0 in at the center strap. This is the single most important constraint on chassis height (see §5).
- **Typical loose-ball count in a drill/lesson basket:** 50–75 balls. That sets hopper capacity.

---

## 3. Subsystem design

### 3.1 Ball detection — "How does it know where the balls are?"

This is the question that kicked off the design, and the good news is that detection is the *easy* part. A tennis ball's optic-yellow color occupies a band that essentially nothing else on a court shares, so it segments cleanly with basic computer vision (simple HSV color thresholding — no machine learning strictly required for v1). We combine three layers:

1. **Overhead camera — the backbone.** A single wide-angle camera mounted on a net post or the fence, looking down at the whole court. From a bird's-eye view the system sees every ball at once, plans an efficient collection route, *and* tracks the robot's own position. This is dramatically simpler and more reliable than making the robot reason from ground level, and it's cheap. Make this the primary sensor for v1.
2. **Onboard forward camera — the last half-meter.** The overhead view loses precision right at the robot's feet, so a forward-facing camera lets the robot home in and line each ball up with the intake throat.
3. **Blind systematic coverage — the fallback.** Because the court is a bounded rectangle, even with zero detection the robot can mow the whole surface in a back-and-forth (boustrophedon) pattern like a robot vacuum and sweep everything up. Cheap insurance if a camera is occluded or fails.

### 3.2 Pickup mechanism — bowling-alley style, kept simple

This is the piece that was missing from the brief and where v1 should concentrate its effort. Recommended approach: **a pair of counter-rotating wheels** at the front — the same mechanism a tennis ball machine uses to *launch* a ball, run in reverse so it *lifts* one. It's the bowling-alley ball-return principle shrunk to tennis scale, and because ball machines already prove the hardware at this size, it's low-risk.

Front-to-back, how a ball gets from ground to hopper:

- **A low scoop lip** at the front, riding ~2–3 mm off the court on a smooth skid, so a resting ball rolls up onto it instead of being pushed away.
- **A funnel throat** narrowing from a wide mouth to the wheel gap, so the ball doesn't need perfect centering.
- **Two counter-rotating wheels** (~70 mm foam-over-rubber), set slightly *closer* than a ball is wide (~50–55 mm gap). The 65 mm ball is squeezed to ~52 mm, the wheels grip it, and their upward-moving faces fling it up a short chute into the hopper. One small motor + one belt spins both wheels.
- **Why wheels, not a passive scoop:** a scoop or ramp just shoves the ball ahead of it. Moving a ball *up and over* a lip requires *gripping* it — spinning wheels grip. Two wheels and one motor is about as simple as a reliable lift gets.
- **Simpler fallback — one brush roller:** a single powered bristle roller (like a range-ball collector) can flick balls up the ramp with one motor and no gap to tune, but grips a bouncy ball a little less surely. Plan: bench-test both cheaply, keep the more reliable one. Lead = twin wheels; brush = backup.

### 3.3 Navigation & localization

Localization is nearly free here because the court is a fixed, high-contrast rectangle:

- **Primary:** the overhead camera reports the robot's position on the court directly.
- **Secondary:** a downward-facing camera detects the white lines for on-board position correction, fused with wheel odometry and a cheap IMU.
- No GPS or LiDAR needed for v1 — save the cost and complexity.

Path planning is a coverage/routing problem over a known rectangle: pick up the nearest cluster first, minimize travel, end near the dock.

### 3.4 Chassis, drive, and the net

- **Differential drive** (two driven wheels + caster) — simple, turns in place, ideal for a flat court.
- **The net decision (needs an early call):** the robot can only cross to the other half by ducking under the center strap (3 ft clearance) — or we treat **each half-court as a separate job**. This drives chassis height and is the biggest single architecture fork in v1. Recommendation below in §6.
- Low profile, rounded, bright/high-visibility shell so it doesn't trip players and is easy to see.

### 3.4b Reaching balls at the fence and in corners

A front-mouth collector can't push its nose flush into a 90° corner, and a ball hugging the fence sits just outside the intake's path. Fix, borrowed from street sweepers and robot vacuums: **a rotating side-sweeper (gutter) brush** at the front corner of the robot, sticking out past the body edge.

- When the overhead camera sees balls near the fence, the robot runs a **wall-follow routine** along it.
- The angled, vertical-axis brush reaches into the corner and against the fence and **flicks the ball inward** — off the fence and into the robot's forward path, where the twin-wheel intake grabs it.
- Cost: one extra small motor, on a thoroughly proven mechanism (every Roomba and street sweeper uses it).
- For a true corner apex, the robot approaches diagonally so the brush rakes the ball out along one fence line first. The genuinely pinned 1–2% of balls in the exact apex are an accepted v1 limitation.

### 3.5 Power & docking

- **Charging dock is the primary power source** — a corner-mounted dock the robot returns to via IR/visual beacon homing when its battery runs low or its hopper is full.
- **Solar is a bonus, not the backbone.** Be honest about the physics: a low, ~2-ft-wide robot simply doesn't have much roof area, so a panel is a trickle top-up that extends runtime, not a primary charger. Design around the dock; treat solar as range extension. (Over-investing in solar for v1 is a classic way to blow the schedule.)
- **Hopper full-detection** (simple weight or optical sensor) tells the robot when to return and dump, whether or not the battery is low.

### 3.5b Hopper — sized to standard capacities

Rather than invent a size, anchor to what teaching hoppers already come in. Common commercial capacities: Gamma baskets ship in 50, 55, 75, 80, 90, 110, and 140; folding rolling coach carts are typically 150. A lesson/drill basket is 50–75; a coach's cart ~150. Recommendation: build the hopper cavity around a **standard 75–110 ball basket** — enough to clear a typical loose-ball session in one or two trips without making the robot tall and tippy. Using an off-the-shelf basket size means the removable bin can literally be a stock hopper we drop in and lift out to empty.

### 3.6 Safety & when it runs

The uncomfortable question: collecting balls *while people are actively playing* is a safety and annoyance problem — nobody wants a robot underfoot mid-rally, and person-detection/liability is a large body of work.

**Recommendation for v1:** the robot runs **on command, between sessions or drills** (or parks at the fence and deploys only when told). This sidesteps most of the person-detection burden. Regardless, include bump sensors, an obvious physical e-stop, and a "stop if something large is in the path" behavior.

---

## 4. Electronics & compute architecture

Ballbot splits its computing the standard robotics way: a Linux single-board computer (the "brain") handles heavy, non-real-time perception and planning, while a small microcontroller runs the hard-real-time motor-and-sensor loop. That split matters — a momentary hiccup in the vision pipeline can never stall a motor command or delay the e-stop. In v1 the overhead camera stays dumb: it just streams frames over Wi-Fi to the onboard computer, which does all the processing.

Three tiers:

- **Onboard computer (the brain):** a Linux SBC — a Raspberry Pi 5 is ample for the classic OpenCV color-vision pipeline; step up to a Jetson Orin Nano only if we later add a learned detector for tricky lighting or heavy occlusion. Runs frame capture, the optic-yellow mask, homography to court coordinates, the route planner, and the motion controller.
- **Real-time microcontroller:** a Teensy 4.1 or ESP32 handling motor PWM, quadrature encoder reads, sensor polling, and a hardware e-stop. Talks to the SBC over USB serial — taking velocity commands, returning odometry.
- **Overhead camera:** kept dumb in v1, streaming frames to the SBC over Wi-Fi. A later version could push detection onto a small edge board at the camera to cut bandwidth.

**Vision/autonomy pipeline:** capture → HSV optic-yellow mask → contour/blob detection → homography (pixel→court, calibrated once off the four court corners) → robot pose from an overhead marker fused with onboard odometry/IMU → greedy nearest-ball route with corner flags → velocity commands to the microcontroller.

### 4b. Bill of materials (v1 prototype, ballpark)

| Subsystem | Component | Suggested part | ~Cost |
|---|---|---|---|
| Compute | Onboard computer (brain) | Raspberry Pi 5 8GB *or* Jetson Orin Nano | $80–250 |
| Compute | Real-time microcontroller | Teensy 4.1 / ESP32 | $25 |
| Vision | Overhead camera | Wide-angle IP cam / Pi HQ cam on Pi Zero 2 W | $60 |
| Vision | Forward camera | Pi Camera Module 3 / USB webcam | $25 |
| Vision | Downward camera | Small USB camera | $15 |
| Drive | Motor drivers | Dual H-bridge (Cytron MDD10 / VNH5019) | $30 |
| Drive | Drive motors ×2 | 12V gearmotors w/ encoders | $40 |
| Drive | Intake motor | 12V brushed gearmotor | $20 |
| Drive | Side-brush motor | Small 12V gearmotor | $12 |
| Sensing | IMU | BNO085 / MPU-6050 | $15 |
| Sensing | Bump, e-stop, hopper-full | Microswitches + IR break-beam / load cell | $20 |
| Power | Battery | 12V LiFePO₄ ~10Ah | $60 |
| Power | BMS / charge controller | Off-the-shelf BMS + DC-DC | $20 |
| Power | Solar panel | 20–30W semi-flexible | $40 |
| Power | Dock | Charging contacts + IR homing beacon | $25 |
| **Total** | **Ballpark, one prototype** | | **≈ $490–660** |

*Rough prototype-quantity estimates; chassis, wheels, fasteners, and wiring extra. Not a production BOM. Plus the mechanical bits: 2× counter-rotating intake wheels + funnel/scoop lip, rotating side brush, molded hopper bin (75–110 balls), and a low weather-resistant high-visibility shell.*

---

## 5. Open decisions to resolve (for Tim & Kevin)

These are the forks worth settling before we cut metal:

1. **Net-crossing vs. half-court jobs.** Duck under the 3-ft center strap (low chassis, more mechanical constraint) *or* run each half-court as an independent task (simpler robot, requires repositioning)?
2. **Run-during-play vs. between-sessions.** Between-sessions is far cheaper and safer for v1 — agree to defer live-play operation?
3. **Solar scope.** Confirm solar is a bonus trickle-charger for v1, with the dock as primary — yes/no?
4. **Hopper capacity target.** 50, 75, or more? Drives size and dump frequency.
5. **Pickup mechanism prototype-off.** Build both a twin-roller and a brush intake as quick prototypes and bake-off, or commit to rollers now?

---

## 6. Recommended v1 baseline (my opinion)

If we want the fastest path to a working, demoable robot: **treat each half-court as a separate job** (skip net-crossing), **run between sessions only**, **dock-primary power with a token solar panel**, **twin-roller intake with a funnel**, **overhead camera as the backbone** with an onboard homing camera, and a **50–75 ball hopper**. That's the smallest machine that convincingly does the core loop — and it's the platform every one of Kevin's future features (serving gun, squeegee, blower) later bolts onto.
