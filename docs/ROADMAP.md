# Roadmap: Kimera-Multi on simulated TurtleBot3 (ROS 2 Jazzy)

Goal: run Kimera-Multi (distributed visual-inertial SLAM) on several simulated
TurtleBot3 robots in Gazebo Harmonic, alongside Cartographer and Nav2, with
frontier exploration driving the robots.

Starting point (2026-11-19): one simulated TB3 (`waffle_pi`) in Docker builds a
map with Cartographer and navigates with Nav2. The Docker setup is in
`docker/Dockerfile.sim_setup` on the `simulation_testing` branch.

Phases 1–2 and Phase 3 can run in parallel. Phase 7 can start at any time.

## Dependency notes

These findings shape Phases 0–2. They were checked against the pinned submodules.

- **GTSAM 4.2a9.** This is what the Kimera-Multi `.repos` file uses, and it matches the current pin.
  Keep it.
- **Kimera-RPGO is needed in two versions:**
  - Kimera-PGMO (ROS 2) needs RPGO `develop`: `find_package(kimera_rpgo)`,
    `kimera_rpgo::` namespace, and it needs `nlohmann_json`.
  - Kimera-VIO `ros2/base` needs RPGO `master`: `find_package(KimeraRPGO)`,
    `KimeraRPGO::` namespace.
  - The package names, headers, namespaces and libraries all differ, so both
    versions can be installed side by side. Fallback: fork Kimera-VIO and switch it
    to `kimera_rpgo`.
- **Missing PGMO dependencies.** Kimera-PGMO also needs `config_utilities` (`main`) and
  `Spatial-Hash` (`main`), as listed in `Kimera-PGMO/install/packages.yaml`.
- **Messages already ported.** `pose_graph_tools` (`ros2`) already defines the Kimera-Multi messages
  (`VLCFrames`, `BowQueries`, `LoopClosures`, ...) for ROS 2.
- **Fork state.** The `ros2-jazzy-port` branches of the forks (Kimera-VIO-ROS,
  Kimera-Distributed, Kimera-Multi-LCD, dpgo_ros) are identical to `main`, so no
  porting has been done yet.
- **Dense mesh.** PGMO's mesh input (`kimera_pgmo_msgs/Mesh`) came from Kimera-Semantics
  (voxblox) in ROS 1. That is not a build dependency, but without a mesh source
  there is no dense mesh.

## Phase 0: Housekeeping (**COMPLETED**)

- [x] Fork submodules on `ros2-jazzy-port`
- [x] GTSAM pinned at 4.2a9
- [x] Docker, `.dockerignore` and RViz config committed (`simulation_testing`)
- [x] Reference dependency lists: `kimera_multi.repos`, `Kimera-PGMO/install/packages.yaml`
- [x] Add submodules `src/kimera/config_utilities` (`main`) and `src/kimera/Spatial-Hash` (`main`)
- [x] Set `branch = develop` for Kimera-RPGO in `.gitmodules` and move the submodule to `develop`

## Phase 1: Core C++ libraries (no ROS), in a Docker `kimera` stage

- [ ] Install from apt: glog, gflags, Eigen, OpenCV, `nlohmann-json3-dev`, Boost, TBB
- [ ] GTSAM 4.2a9 with `GTSAM_USE_SYSTEM_EIGEN=ON`, `GTSAM_POSE3_EXPMAP=ON`,
      `GTSAM_ROT3_EXPMAP=ON`, `GTSAM_TANGENT_PREINTEGRATION=OFF`, `GTSAM_BUILD_UNSTABLE=ON`
- [ ] opengv, using the same Eigen as GTSAM and matching `-march=native` settings
- [ ] DBoW2 (dorian3d)
- [ ] Kimera-RPGO `master` (d28b4df), installed as `KimeraRPGO` for Kimera-VIO
- [ ] Kimera-RPGO `develop`, installed as `kimera_rpgo` for PGMO (check that it builds against GTSAM 4.2a9)
- [ ] Kimera-VIO (`ros2/base`)
- [ ] dpgo
- [ ] Run Kimera-VIO's standalone executable on a EuRoC sequence

Done when: all libraries install in one image and Kimera-VIO tracks EuRoC.

## Phase 2: Single-robot ROS 2 layer, tested on a dataset

- [ ] colcon build: `config_utilities`, `spatial_hash`, `pose_graph_tools`, `mesh_rviz_plugins`
- [ ] colcon build: Kimera-PGMO
- [ ] Port Kimera-VIO-ROS (`ros2-jazzy-port`):
  - catkin → ament
  - nodelets → components
  - ROS 1 messages → `pose_graph_tools_msgs`
  - Python launch files
  - drop the `*_catkin` dependencies
- [ ] Convert EuRoC to a ROS 2 bag (`rosbags-convert`)
- [ ] Run Kimera-VIO-ROS with PGMO on the bag (no mesh input yet)
- [ ] Remove the leftover ROS 1 `.launch` files from each fork as it is ported

Done when: RViz shows the VIO trajectory and PGMO's optimized pose graph from a ROS 2 bag.

## Phase 3: Kimera-ready TB3 in `src/custom_tb3`

- [ ] TB3 variant with a stereo pair (about 10 cm baseline, 20–30 Hz) plus the 200 Hz IMU
- [ ] Bridge stereo images and `camera_info` with ros_gz
- [ ] Kimera parameter files: camera intrinsics, `T_BS` extrinsics, IMU noise matching the SDF
- [ ] Textured world (`turtlebot3_house`)
- [ ] Generate the URDF with xacro (the raw `.urdf` files contain `${namespace}`)
- [ ] Record a teleop bag of stereo images and IMU data, and check that TF matches the parameter files

Done when: a stereo+IMU bag exists and TF matches the Kimera parameter files.

## Phase 4: Kimera-VIO on one simulated TB3

- [ ] Run Kimera-VIO-ROS on the live simulation
- [ ] Compare against Gazebo ground truth with `evo`
- [ ] Tune parameters; consider wheel odometry as an extra input (planar motion is hard for VIO)
- [ ] Separate frames: Kimera must not publish `map`/`odom` while Cartographer owns `map→odom`
- [ ] Check that Cartographer and Nav2 still work alongside Kimera

Done when: drift is acceptable over a lap and Cartographer and Nav2 are unaffected.

## Phase 5: Port the multi-robot packages

- [ ] Port dpgo_ros
- [ ] Port Kimera-Multi-LCD
- [ ] Port Kimera-Distributed
- [ ] Reuse the `pose_graph_tools_msgs` messages instead of redefining them
- [ ] Replace one ROS master per robot with one DDS domain and namespaces (`/tb3_0`, `/tb3_1`)
- [ ] Test with two robots' worth of bags before running live

Done when: bags produce loop closures between robots and a jointly optimized trajectory.

## Phase 5b: Dense mesh (optional, after Phase 6)

- [ ] Choose a mesh source that publishes `kimera_pgmo_msgs/Mesh`: port Kimera-Semantics
      and voxblox, use a ROS 2 TSDF mapper with a converter node, or use Hydra
- [ ] Connect it to PGMO's mesh frontend

Done when: PGMO deforms the mesh after loop closures.

## Phase 6: Multi-robot TB3 simulation

- [ ] Spawn 2–3 namespaced robots in Gazebo
- [ ] Per robot, namespaced: robot_state_publisher, Cartographer, Nav2, Kimera-VIO
- [ ] Run Kimera-Distributed across the robots

Done when: the robots' trajectories merge into one frame after they meet.

## Phase 7: Exploration

- [ ] Single-robot frontier explorer on Cartographer's `/map` and Nav2 (`NavigateToPose`)
- [ ] Several robots dividing the frontiers between them

Done when: several robots explore on their own and Kimera-Multi produces the merged result.

## Main risks

1. RPGO `develop` against GTSAM 4.2a9 (Phase 1).
2. VIO performance on a planar, low-texture ground robot (Phase 4).
3. The size of the Kimera-Distributed port (Phase 5).
