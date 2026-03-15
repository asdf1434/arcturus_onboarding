# Codebase Guide

The `all_seaing_vehicle` repository is organized into packages:

| Package | What it does |
|---------|-------------|
| `all_seaing_perception` | Camera, LiDAR, YOLO, point cloud fusion, object tracking |
| `all_seaing_navigation` | A* planner, grid map generation |
| `all_seaing_controller` | PID control, potential field obstacle avoidance |
| `all_seaing_autonomy` | Competition task implementations |
| `all_seaing_common` | Base classes (TaskServerBase, ActionServerBase) |
| `all_seaing_bringup` | Launch files and YAML config |
| `all_seaing_driver` | Hardware interfaces (GPS, thrusters, cameras) |
| `all_seaing_description` | Robot model (URDF), thruster config |
| `all_seaing_interfaces` | Custom message/action/service definitions |

## Key files to look at

### Perception
- `all_seaing_perception/all_seaing_perception/yolov8_node.py` — runs YOLO on camera frames
- `all_seaing_perception/src/bbox_project_pcloud.cpp` — fuses bounding boxes with LiDAR
- `all_seaing_perception/src/object_tracking_map.cpp` — tracks obstacles over time

### Navigation and control
- `all_seaing_navigation/all_seaing_navigation/navigation_server.py` — A* path planning
- `all_seaing_controller/all_seaing_controller/controller_server.py` — PID path following
- `all_seaing_controller/all_seaing_controller/potential_field.py` — obstacle avoidance

### Task execution
- `all_seaing_common/all_seaing_common/task_server_base.py` — base class for all tasks
- `all_seaing_autonomy/all_seaing_autonomy/roboboat/run_tasks.py` — task state machine
- `all_seaing_autonomy/all_seaing_autonomy/roboboat/follow_buoy_path.py` — follow the path task
- `all_seaing_autonomy/all_seaing_autonomy/roboboat/entry_gates.py` — entry gates task
- `all_seaing_autonomy/all_seaing_autonomy/roboboat/speed_challenge.py` — speed challenge task
- `all_seaing_autonomy/all_seaing_autonomy/roboboat/docking.py` — docking task

### Shared utilities
- `all_seaing_autonomy/all_seaing_autonomy/buoy_utils.py` — shared buoy detection/tracking functions
- `all_seaing_autonomy/all_seaing_autonomy/geometry_utils.py` — geometry helpers

### Config
- `all_seaing_bringup/config/` — YAML config files organized by subsystem
- `all_seaing_bringup/launch/vehicle.launch.py` — real robot launch
- `all_seaing_bringup/launch/sim.launch.py` — simulation launch

## Custom messages

Defined in `all_seaing_interfaces`:

- `Obstacle` — a single tracked obstacle with label, position, convex hull, bounding box
- `ObstacleMap` — a list of obstacles (this is what task code subscribes to)
- `ControlOption` — velocity command with a priority level
- `FollowPath.action` — action for path planning + following
- `Waypoint.action` — action for direct point navigation
- `Task.action` — action for running a competition task

## Explore the codebase

Try to find:

1. Which node runs YOLO object detection?
2. What message type does the obstacle map use?
3. Where is `TaskServerBase` defined? What methods does it provide?
4. Find the follow-the-path task. What are `init_setup()` and `control_loop()` doing?
5. How does `run_tasks.py` decide which task to run next?
6. Where are the PID gains configured?
