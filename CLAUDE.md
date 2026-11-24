# CLAUDE.md — LiDAR Costmap Block

## What this project is
A class-project block for an autonomous car. It reads a Slamtec RPLIDAR A1 (A1M8) over USB serial,
publishes `sensor_msgs/LaserScan` on `/scan` via Slamtec's `sllidar_ros2` driver, and turns those scans
into a Nav2 local costmap published on `/local_costmap/costmap` (`nav_msgs/OccupancyGrid`).
A separate team's collision decision block subscribes to that costmap. This repo contains only
configuration, launch files, and test tooling: the costmap itself is Nav2's `nav2_costmap_2d`, not custom code.

The step-by-step task list is in `PLAN.md`. Read it at the start of every session.

## Environment
- Ubuntu 24.04 (native or WSL2), ROS2 **Jazzy**. Deployment target: Jetson Orin Nano running Jazzy.
- Workspace: `~/ros_ws` is a **shared** ROS2 workspace that may hold unrelated packages. This project is only
  `~/ros_ws/src/lidar_costmap` (this folder, where Claude Code runs). The driver is at `~/ros_ws/src/sllidar_ros2`.
- `lidar_costmap` is its own git repository. The driver is **not** part of it: it is listed in
  `dependencies.repos` and imported next to it with `vcs import`.
- Do not run `sudo` or install system packages; if something is missing, stop and tell the student what to install.
- Only create or edit files inside `~/ros_ws/src/lidar_costmap`. Do not touch other packages in `~/ros_ws/src`,
  and do not delete or clean `~/ros_ws/build`, `install`, or `log`.
- ROS is **not** sourced in `~/.bashrc`. Prefix every ROS command with
  `source /opt/ros/jazzy/setup.bash && source ~/ros_ws/install/setup.bash && ...`
  (only the first part before the workspace has been built once). Do not edit `~/.bashrc`.
- Build (always from the workspace root, never from this folder):
  `cd ~/ros_ws && colcon build --packages-select lidar_costmap` (add `sllidar_ros2` the first time:
  `--packages-select sllidar_ros2 lidar_costmap`)
- Use ROS2 (not ROS1) commands and docs: `ros2 launch`, `rviz2`, `sensor_msgs/msg/LaserScan`, `*.launch.py`.

## Package layout (`~/ros_ws/src/lidar_costmap`, ament_python)
```
lidar_costmap/
├── launch/sim_testbench.launch.py      # static TFs + costmap + lifecycle manager (no driver)
├── launch/hardware.launch.py           # same + sllidar_ros2 driver
├── config/local_costmap_params.yaml    # all costmap parameters
├── lidar_costmap/test_script.py        # fake /scan publisher + costmap checker (simulation)
├── lidar_costmap/check_scan.py         # nearest-object check for the real LiDAR
├── results/                            # test output (results tables, saved by test_script)
├── CLAUDE.md, PLAN.md                  # this file and the task list
├── README.md                           # setup and run instructions for teammates
├── dependencies.repos                  # vcstool list of external repos (sllidar_ros2)
├── package.xml
└── setup.py                            # data_files for launch/ and config/; entry_points for both scripts
```

## Interface contract (do not change without asking)
| | Property | Value |
|---|---|---|
| data_in | Physical | USB serial, 115200 baud 8N1, Slamtec binary packets (datasheet Fig. 2-5) |
| data_in | ROS topic | `/scan`, `sensor_msgs/msg/LaserScan`, `frame_id: laser`, ~5.5 Hz, 360° |
| data_out | ROS topic | `/local_costmap/costmap`, `nav_msgs/msg/OccupancyGrid` |
| data_out | Rate | 5 Hz (`publish_frequency` = `update_frequency` = 5.0) |
| data_out | Frame / grid | `frame_id: odom`, rolling window 3 m × 3 m, resolution 0.05 m (60 × 60 cells) |
| data_out | Values | 100 lethal, 99 inscribed, 1–98 inflated, 0 free, −1 unknown (unused: `track_unknown_space: false`) |

## TF tree
`odom → base_link → laser`
- `base_link → laser`: static transform. Bench placeholder is x = y = 0, z = 0.10 m, roll = pitch = yaw = 0;
  real values are measured after mounting (TBD). Metres and radians, REP 103 axes (x forward, y left, z up).
- `odom → base_link`: will come from the odometry block. Until then, a static identity placeholder in `sim_testbench.launch.py`, and behind a launch argument (`use_fake_odom`) in `hardware.launch.py`.

## Rules for working in this repo
- Work **one phase of PLAN.md at a time**. When a phase's "Done when" checks pass, tick its boxes in PLAN.md and stop for review.
- Run the "Done when" commands yourself and report their actual output. Never mark a check as passed without running it.
- **Do not modify `~/ros_ws/src/sllidar_ros2`.** It is the vendor driver; configure it only through launch arguments.
- Phases marked **[MANUAL]** need physical hardware (LiDAR, Jetson, car). Do not attempt or simulate them; you may prepare files or commands for them.
- Values marked **TBD** (car dimensions, mounting offsets) are placeholders. Keep the placeholder, leave a `# TBD:` comment, and list it in your summary. Do not invent real-looking numbers.
- Do not change the interface contract above. If a check fails because of it, stop and ask.
- Keep Python simple and readable (rclpy, no extra dependencies beyond standard ROS2 Jazzy packages and numpy). This is a class project and others must be able to follow the code.
- If a Nav2 parameter name or behaviour is uncertain, check the installed package (`ros2 param list`, `ros2 param describe`, or the Jazzy docs) rather than guessing.

## Useful checks
```
ros2 topic list
ros2 topic info /local_costmap/costmap -v
ros2 topic hz /local_costmap/costmap
ros2 topic echo /local_costmap/costmap --once --no-arr
ros2 run tf2_ros tf2_echo odom laser
ros2 run tf2_tools view_frames
```
