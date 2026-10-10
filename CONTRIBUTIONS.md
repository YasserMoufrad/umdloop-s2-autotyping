# My contributions — Yasser Moufrad

This is my fork of [AravJuneja/umdloop-s2-autotyping](https://github.com/AravJuneja/umdloop-s2-autotyping), our team's submission for UMD Loop's Autonomous Typing Challenge. A simulated robot arm with a camera has to find a panel marked with four ArUco markers, locate a keyboard whose position on the panel is never given, and type a launch key exactly. Everything runs as ROS 2 nodes in Docker against the challenge simulator.

I owned the perception pipeline: turning the camera feed into the 3D position of every key. When our arm control lead stepped back days before the deadline, I also took over the arm controller and integrated it with perception.

## 1. Getting camera data at all: QoS and Docker

My `key_projector` node reported `frames=0 (0.0 hz)`: it received no data at all. There were two separate causes.

- **QoS mismatch.** ROS 2 subscribers request a RELIABLE connection by default, but the camera topic publishes BEST_EFFORT. ROS 2 doesn't connect the two and gives no error. I switched the subscriber to `qos_profile_sensor_data` (best-effort, newest frame first).
- **Docker shared memory.** The sim and dev containers talk through shared `/dev/shm`. Restarting only the sim container destroyed it, so the dev container couldn't find any topics. A full `docker compose down` / `docker compose up -d` rebuilds the network correctly.

Before both were fixed, partial frames came through with the bottom of the keyboard replaced by a gray bar, from dropped packets. After both fixes, the feed ran cleanly at 15 Hz.

## 2. Panel pose with solvePnP — [PR #23](https://github.com/AravJuneja/umdloop-s2-autotyping/pull/23)

There is no `board` TF frame (the interface spec leaves it out on purpose), so I computed the panel's pose myself from the 2D marker detections with `cv2.solvePnP`.

- **First attempt:** I averaged each marker's corners into 4 center points. It wasn't accurate enough, because center points can't show which way a marker is tilted, so the rotation was off. A small rotation error moves keys a long way across the board.
- **Fix:** I used all 16 corners (4 per marker). Both the position and the orientation became stable.

## 3. Finding the undocumented keyboard

The keyboard's offset inside the panel (`key_area_origin`) is never provided.

- **First attempt:** I hardcoded an estimated origin. The grid was tilted correctly but shifted. The lesson: hardcoded positions break the moment anything moves, so the system has to measure.
- **Fix:** I built it in four functions:
  1. `_panel_pose_homography` / `_rectify_to_board_frame` warp the camera image into a top-down, metric view of the panel.
  2. `_mask_markers` paints over the ArUco markers in bezel gray so their texture isn't mistaken for keys.
  3. `_find_key_grid_rectangle` computes local variance (`cv2.boxFilter`), then thresholds it: the flat bezel goes black and the textured keys go white. It then boxes the white region.
  4. `_update_key_area_estimate` uses **Welford's algorithm** to average the origin across frames, keeping a running mean and variance without storing past frames. The keymap is only built after `MIN_MEASUREMENTS_TO_TRUST = 5` agreeing measurements, so one jittery frame can't send the arm to the wrong key.

## 4. How I tested it

- **Visual check (`_on_image`):** I project the computed 3D key positions back through the camera model onto the raw image and draw a green dot and label on each key. The output, `key_projection_validation.png`, updates live, so I can see the dots lock onto the key centers.
- **Teammate review:** a teammate checked the result against three saved frames from different episodes (panel at 0.705 m, 0.853 m and 0.781 m). The keyboard position was consistent to about 0.2 mm, max reprojection error was 0.86–1.13 px, and projected key centers landed on the caps.

## 5. Arm controller and final demo

When our arm control lead stepped back days before the deadline, I reverse-engineered his closed-loop joint-velocity controller, which builds directly on my perception output, and integrated the two. The full system reached 100% typing accuracy in simulation. I presented the whole system, including my teammate's work, to the judges.

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
