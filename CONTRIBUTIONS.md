# My contributions — Yasser Moufrad

This is my fork of [AravJuneja/umdloop-s2-autotyping](https://github.com/AravJuneja/umdloop-s2-autotyping), our team's submission for UMD Loop's Autonomous Typing Challenge. A simulated robot arm with a camera has to find a panel marked with four ArUco markers, locate a keyboard whose position on the panel is never given, and type a launch key exactly. Everything runs as ROS 2 nodes in Docker against the challenge simulator.

Team: Arav Juneja, Victor Casado, and me. This page covers the parts I personally built.

## 1. Panel pose and key projection — [PR #23](https://github.com/AravJuneja/umdloop-s2-autotyping/pull/23)

**What was broken:** the key projector waited for a `board` TF frame that never existed. The marker detector only outputs 2D pixel coordinates, so key projection was blocked.

**What I built** (commits `9504ba6`, `d3e73b4`):
- Bypassed TF. The projector subscribes to the raw 2D marker detections and solves the panel's pose itself with `cv2.solvePnP` (16 corner correspondences from the four markers).
- Located the undocumented keyboard. Using the pose, I warp the camera image into a top-down view of the panel with a homography, mask out the markers, and find the key grid with a local-variance threshold: the flat bezel has almost no texture, and the keys do.

**How it was tested:** a teammate checked the perception against three saved frames from different episodes, with the panel at 0.705 m, 0.853 m and 0.781 m. The keyboard position came back consistent to about 0.2 mm, max reprojection error was 0.86–1.13 px, and projected key centres landed on the caps. Teammates later refined the module further (see `member_ws/src/panel_detect/panel_detect/key_positions.py`).

## 2. Stabilizing the key grid estimate (Welford's algorithm)

A single frame's measurement is noisy: the pose is re-fit on every frame with about 1 px of noise. If the keymap were rebuilt from each frame, the arm could aim at the wrong key or a gap.

- I used Welford's online algorithm to keep a running mean (and variance) of the keyboard's position across frames, with no stored history.
- The keymap is only built after `MIN_MEASUREMENTS_TO_TRUST = 5` agreeing measurements.
- The estimate resets at the end of each episode, since the panel moves between episodes.

See `_update_key_area_estimate` in `key_positions.py`.

## 3. ROS 2 QoS and Docker communication fix

The team's camera pipeline was fully blocked: subscribers in other containers received no data. I traced it to ROS 2 QoS profile mismatches and Docker inter-container communication failures, fixed both, and restored the camera feed for the team.

## 4. Arm controller integration and final demo

When a teammate left days before the deadline, I took over the closed-loop joint-velocity arm controller and integrated it with the perception pipeline. The full system reached 100% typing accuracy in simulation, and I presented the whole system, my work and my teammate's, to the judges.

## Known limits

- **Simulation only.** None of this has run on a physical robot or camera.
- **All four markers have to be visible** for a pose fit; partial views are rejected.
- **The panel is assumed fixed within an episode.** If it moved mid-episode, the running average would adapt slowly. A recency-weighted average, or rejecting frames far from the mean using the tracked variance, would fix that.

## Running it

```
./start.sh   # bring up the simulator and dev containers
make e2e     # build, launch, and type 3 random episodes, checking each for an exact match
```

See the [main README](README.md) for full setup.
