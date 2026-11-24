# PLAN.md — LiDAR Costmap Block Implementation

Work one phase at a time. After each phase, run every "Done when" check, paste the output in your
summary, tick the boxes below, and stop for review. Phases marked **[MANUAL]** need hardware and are
done by the student; Claude Code may only prepare files for them.

Status key: `[ ]` not started · `[x]` done

---

## Phase 0a — [MANUAL] Install ROS2 Jazzy and Nav2 (student does this; needs `sudo`)
Follow the official guide: https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html
- [ ] (Windows only) WSL2 with **Ubuntu 24.04**: `wsl --install -d Ubuntu-24.04`
- [ ] Set up the ROS2 apt sources exactly as the official guide says
- [ ] `sudo apt install ros-jazzy-desktop ros-jazzy-navigation2 ros-jazzy-nav2-bringup python3-colcon-common-extensions python3-rosdep python3-vcstool`
- [ ] `sudo rosdep init && rosdep update`
- [ ] (Optional) Add `source /opt/ros/jazzy/setup.bash` to `~/.bashrc`. If you skip this, source it manually in every terminal; Claude Code sources it in each command (see CLAUDE.md)
- [ ] `mkdir -p ~/ros_ws/src` and clone the `lidar_costmap` git repo to `~/ros_ws/src/lidar_costmap`

## Phase 0b — Verify environment and add the driver (Claude Code)
- [ ] Confirm Ubuntu 24.04 and ROS2 Jazzy: `lsb_release -a`, `printenv ROS_DISTRO` → `jazzy`.
      If anything from Phase 0a is missing, stop and list what to install. Do not run `sudo` commands
- [ ] Create `~/ros_ws/src/lidar_costmap/dependencies.repos` (vcstool format) listing
      `sllidar_ros2` → `https://github.com/Slamtec/sllidar_ros2.git`, branch `main`
- [ ] If `~/ros_ws/src/sllidar_ros2` is missing: `cd ~/ros_ws/src && vcs import < lidar_costmap/dependencies.repos`
      (the driver is a **sibling** of `lidar_costmap` in `src/`, never inside the `lidar_costmap` repo)
- [ ] Check dependencies without installing: `cd ~/ros_ws && rosdep check --from-paths src --ignore-src`.
      If anything is missing, **stop** and ask the student to run (needs `sudo`):
      `cd ~/ros_ws && rosdep install --from-paths src --ignore-src -y`, then re-run the check
- [ ] `cd ~/ros_ws && colcon build --packages-select sllidar_ros2`

**Done when:**
- `ros2 pkg list | grep -E "nav2_costmap_2d|nav2_lifecycle_manager|sllidar_ros2"` lists all three
- `ros2 pkg executables nav2_costmap_2d` output is recorded (needed in Phase 3)
- `rviz2` starts (close it after)

---

## Phase 1 — Package skeleton
- [ ] If `~/ros_ws/src/lidar_costmap` has no `package.xml` yet, create the package there without overwriting
      CLAUDE.md/PLAN.md: e.g. run `ros2 pkg create --build-type ament_python lidar_costmap` in a temp folder and move
      its files in. If a package already exists, check it against this phase instead of recreating it
- [ ] Create `launch/`, `config/`, `results/` folders
- [ ] After filling in `package.xml`, re-run `rosdep check --from-paths src --ignore-src` from `~/ros_ws`; if anything is missing, stop and ask the student to run `rosdep install --from-paths src --ignore-src -y`
- [ ] Add a `README.md` with setup steps (Phase 0a, `vcs import < dependencies.repos`, `rosdep install --from-paths src --ignore-src -y`, build, run) so a teammate can reproduce the setup from the repo alone
- [ ] Add a `.gitignore` for Python caches (`__pycache__/`, `*.pyc`); keep `results/` tracked (it is verification evidence)
- [ ] `package.xml` dependencies: `rclpy`, `sensor_msgs`, `nav_msgs`, `geometry_msgs`, `tf2_ros`, `nav2_costmap_2d`, `nav2_lifecycle_manager`, `sllidar_ros2`, `rviz2`
- [ ] `setup.py`: install `launch/*.launch.py` and `config/*.yaml` via `data_files`; add `entry_points` for `test_script = lidar_costmap.test_script:main` and `check_scan = lidar_costmap.check_scan:main` (stub `main()` functions for now)

**Done when:**
- `colcon build --packages-select lidar_costmap` succeeds with no warnings about missing files
- `ros2 pkg executables lidar_costmap` lists `test_script` and `check_scan`

---

## Phase 2 — Costmap configuration
Create `config/local_costmap_params.yaml` with these values (TBD values keep a `# TBD:` comment):

| Parameter | Value |
|---|---|
| `global_frame` | `odom` |
| `robot_base_frame` | `base_link` |
| `rolling_window` | `true` |
| `width`, `height` | `3` (m) |
| `resolution` | `0.05` |
| `update_frequency` | `5.0` |
| `publish_frequency` | `5.0` |
| `always_send_full_costmap` | `true` |
| `track_unknown_space` | `false` |
| `robot_radius` | `0.25` **# TBD: replace with real car dimensions (or use `footprint`)** |
| `plugins` | `["obstacle_layer", "inflation_layer"]` |
| `obstacle_layer.observation_sources` | `scan` |
| `scan.topic` | `/scan` |
| `scan.data_type` | `LaserScan` |
| `scan.marking`, `scan.clearing` | `true`, `true` |
| `scan.inf_is_valid` | `true` (needed so "no return" beams clear old obstacles) |
| `scan.obstacle_max_range` / `scan.raytrace_max_range` | `2.5` / `3.0` |
| `scan.min_obstacle_height` / `scan.max_obstacle_height` | `0.0` / `2.0` (**required**: Nav2's per-source default max is 0.0, which would drop every scan point above the ground, i.e. all of them) |
| `obstacle_layer.max_obstacle_height` | `2.0` |
| `inflation_layer.inflation_radius` | `0.55` (starting value, tuned in Phase 5) |
| `inflation_layer.cost_scaling_factor` | `3.0` (starting value, tuned in Phase 5) |

- [ ] Nest the parameters under the node name/namespace chosen in Phase 3 so the output topic is exactly `/local_costmap/costmap`
- [ ] Verify each parameter name against the Jazzy obstacle/inflation layer docs or `ros2 param list` once the node runs

**Done when:** the YAML loads without errors in Phase 3 and `ros2 param get` on the running costmap node returns the values above (check at least `publish_frequency`, `resolution`, `inflation_radius`, `scan.inf_is_valid`).

---

## Phase 3 — Launch files
### `launch/sim_testbench.launch.py`
Static transform conventions (apply to every static transform in both launch files):
- Use the Jazzy named-argument form of `tf2_ros` `static_transform_publisher`:
  `--x --y --z --roll --pitch --yaw --frame-id <parent> --child-frame-id <child>` (not the deprecated positional form)
- Units: metres and **radians**. Axes follow REP 103: x forward, y left, z up; positive yaw = counter-clockwise seen from above
- `roll` and `pitch` are always `0.0` (the LiDAR is mounted level; a tilted 2D LiDAR is out of scope)
- Frame names must match exactly everywhere: `odom` and `base_link` (costmap YAML), `laser` (driver `frame_id` and the fake `/scan` in `test_script`)
- All launch-argument values are passed as strings and converted with `LaunchConfiguration`

- [ ] `static_transform_publisher`: `odom → base_link`, all values `0.0` (placeholder for odometry)
- [ ] `static_transform_publisher`: `base_link → laser`, `x = 0.0`, `y = 0.0`, `z = 0.10` (**bench placeholder height**; keeps
      scan points safely inside the obstacle-height band instead of exactly on the 0.0 boundary), `yaw` from launch arg `laser_yaw` (default `0.0`)
- [ ] Costmap node (executable found in Phase 0) with `config/local_costmap_params.yaml`, named/namespaced so it publishes `/local_costmap/costmap`
- [ ] `nav2_lifecycle_manager` `lifecycle_manager` with `autostart: true` and `node_names` set to the costmap node
- [ ] Launch arg `use_rviz` (default `false`) that starts `rviz2`

### `launch/hardware.launch.py`
- [ ] Includes `sllidar_ros2`'s `sllidar_a1_launch.py` (args: `serial_port` default `/dev/ttyUSB0`, `frame_id: laser`)
- [ ] `base_link → laser` static transform with launch args `laser_x`, `laser_y`, `laser_z`, `laser_yaw`
      (defaults `0.0`, `0.0`, `0.10`, `0.0`, **# TBD: measured after mounting**; `laser_yaw` may need `3.14159` if the box check in Phase 6 shows the scan is reversed)
- [ ] `use_fake_odom` launch arg (default `true`) that adds the identity `odom → base_link`
- [ ] Same costmap node + lifecycle manager as the sim launch

**Done when** (sim launch running, no `/scan` publisher yet):
- `ros2 topic list` contains `/local_costmap/costmap`
- `ros2 topic info /local_costmap/costmap` → `nav_msgs/msg/OccupancyGrid`
- `ros2 run tf2_ros tf2_echo odom laser` prints translation `[0.000, 0.000, 0.100]` and zero rotation, without errors
- `ros2 launch lidar_costmap sim_testbench.launch.py laser_yaw:=3.14159` → `tf2_echo odom laser` shows a 180° yaw (launch arg is wired through)
- `ros2 run tf2_tools view_frames` shows `odom → base_link → laser`
- `ros2 launch lidar_costmap hardware.launch.py --show-args` lists all launch args

---

## Phase 4 — Test script (`lidar_costmap/test_script.py`)
Build and check each layer before the next.

### 4.1 Fake LiDAR
- [ ] Publishes `LaserScan` on `/scan` at 5.5 Hz: `frame_id: laser`, `angle_min = -π`, `angle_max = π`, 360 beams (`angle_increment = 2π/360`), `range_min = 0.15`, `range_max = 12.0`, `scan_time = 1/5.5`
- [ ] Beams that hit nothing report `inf` (matches the real driver, to be confirmed in Phase 6)
- [ ] Obstacles are circles (center, radius 0.10 m) and line segments (walls); each beam's range = nearest ray intersection
- [ ] Optional parameter `noise_pct` (default 0) adds Gaussian noise of that % of range

### 4.2 Scenarios (all positions in metres, `laser` frame; bench frames are identity so this equals `odom`)
| # | Name | Geometry | Expected costmap |
|---|---|---|---|
| 1 | `empty` | nothing | no cell equals 100 |
| 2 | `single_obstacle` | circle at (1.10, 0) → near face 1.00 m | 100 at the beam hit point (1.00, 0) ±1 cell; a cell 0.35 m from it has cost 1–98; cells > 0.60 m from it are 0 |
| 3 | `two_close` | circles at (1.10, 0.30) and (1.10, −0.30) | 100 at each hit point; midpoint (1.00, 0) has cost ≥ 1 (inflation zones merge) |
| 4 | `wall` | segment x = 1.00 from y = −0.75 to 0.75 | 100 at (1.00, −0.50), (1.00, 0), (1.00, 0.50) ±1 cell |
| 5 | `four_directions` | circles at (1.10, 0), (0, 1.10), (−1.10, 0), (0, −1.10) | 100 at all four hit points (checks angle sign conventions) |
| 6 | `removal` | `single_obstacle` for 3 s, then `empty` | first 100 at (1.00, 0); then that cell is 0 within 3 s (checks clearing / `inf_is_valid`) |

- [ ] Expected hit points are computed from the same geometry code the fake LiDAR uses (not hard-coded separately)

### 4.3 Checker
- [ ] Subscribes to `/local_costmap/costmap`. After switching scenario, waits 2 s (≈10 updates) before checking
- [ ] World → cell: `col = floor((x − origin.x)/res)`, `row = floor((y − origin.y)/res)`, `index = row·width + col`; "±1 cell" = max over the 3×3 neighbourhood
- [ ] Interface checks (once, at start): message type received, rate 5 Hz ±10% over 10 messages, `frame_id == odom`, `resolution == 0.05`, `width == height == 60`, all values in {−1} ∪ [0, 100]
- [ ] Prints a results table (`# | check | expected | measured | PASS/FAIL`) and ends with `ALL TESTS PASSED` or `N TESTS FAILED`; also saves it to `results/results_<timestamp>.md`
- [ ] Exit code 0 if all pass, 1 otherwise

### 4.4 Interactive mode and threading
- [ ] Parameter `interactive` (default `false`): after each scenario, print `"<scenario>: take RViz screenshot, then press Enter..."` and wait for Enter
- [ ] Parameter `scenario` (default `all`): run a single named scenario
- [ ] ROS spinning (scan publishing timer + costmap subscription) runs in a background thread (`MultiThreadedExecutor`), so scans keep publishing while the main thread waits on `input()`

### 4.5 `check_scan.py` (for the real LiDAR)
- [ ] Reads one `/scan` message, ignores `inf`/NaN and ranges below `range_min`, prints `Nearest object: <d> m at <angle>°`, plus `frame_id`, `angle_min`, `angle_max`, `range_min`, `range_max`
- [ ] Optional params `expected_distance` (default 1.00) and `tolerance` (default 0.02) → prints PASS/FAIL

**Done when:**
- With `sim_testbench.launch.py` running, `ros2 run lidar_costmap test_script` runs all 6 scenarios and prints the full table (passing not yet required)
- `ros2 topic hz /scan` ≈ 5.5 Hz while the script runs
- `check_scan` runs against the fake `/scan` (e.g. `test_script --ros-args -p scenario:=single_obstacle -p interactive:=true`) and reports ≈1.00 m at ≈0°

---

## Phase 5 — Simulated verification and tuning
- [ ] Run `test_script`; fix failures one at a time. Likely causes: clearing (`inf_is_valid`, `raytrace_max_range`), inflation width, topic/namespace, QoS (check `ros2 topic info -v`)
- [ ] Record every parameter change and its reason in a "Tuning log" section at the bottom of this file
- [ ] Re-run until all rows pass
- [ ] [MANUAL] Run with `interactive:=true` and `use_rviz:=true`, take one RViz screenshot per scenario, screen-record one full run

**Done when:** `test_script` exits 0 and prints `ALL TESTS PASSED`; the results file is in `results/`.

---

## Phase 6 — [MANUAL] Hardware integration (when the LiDAR is available)
Claude Code may prepare a checklist; the student runs it.
- [ ] `ls /dev/ttyUSB*` shows the LiDAR; fix permissions if needed (`dialout` group)
- [ ] `ros2 launch sllidar_ros2 sllidar_a1_launch.py`: motor spins; log shows serial number, firmware, health OK, no errors
- [ ] `ros2 topic info /scan` → LaserScan, 1 publisher; `ros2 topic echo /scan --once --no-arr` → `frame_id: laser`, angles ≈ ±3.14; record `range_min`/`range_max`
- [ ] `ros2 topic hz /scan` ≈ 5.5 Hz
- [ ] Box 1.00 m in front (measured to the rotation axis): `ros2 run lidar_costmap check_scan` → 1.00 ± 0.02 m; record the angle (sets `laser_yaw`)
- [ ] Check how the real driver reports "no return" (`inf` vs 0 vs `range_max`). If it differs from Phase 4.1, update the fake LiDAR and re-run Phase 5
- [ ] `ros2 launch lidar_costmap hardware.launch.py use_rviz:=true`: the box appears in the costmap at ≈1.00 m

---

## Phase 7 — [MANUAL] Jetson Orin Nano deployment
- [ ] Build `lidar_costmap` in the Jetson's Jazzy environment (Docker or source — match the team's setup)
- [ ] Run `test_script` on the Jetson → `ALL TESTS PASSED`
- [ ] Run `hardware.launch.py` with the real LiDAR; `ros2 topic hz` on `/scan` (≈5.5 Hz) and `/local_costmap/costmap` (≈5 Hz)
- [ ] Record CPU load (`jtop`) with other blocks running

---

## Phase 8 — [MANUAL] Integration with the car and other blocks
- [ ] Mount the LiDAR level; measure `laser_x/y/z` from `base_link`; set `laser_yaw` from Phase 6; re-run the box check measured from `base_link`
- [ ] Replace `robot_radius` TBD with real car dimensions (or a `footprint`); re-run Phase 5
- [ ] Once odometry publishes `odom → base_link`, launch with `use_fake_odom:=false`; confirm with `view_frames`
- [ ] Agree with the collision decision block on topic, type, 5 Hz rate, and cost meanings
- [ ] Drive slowly past a fixed box: it stays fixed in `odom` and clears after passing

---

## Open questions / TBD
- Car dimensions → `robot_radius` or `footprint`
- LiDAR mounting offsets and yaw
- Is a 3 m × 3 m window enough for the car's top speed?
- How Jazzy runs on the Orin Nano (Docker vs source)
- How the real driver encodes "no return"

## Tuning log
| Date | Parameter | Old → New | Reason |
|---|---|---|---|
