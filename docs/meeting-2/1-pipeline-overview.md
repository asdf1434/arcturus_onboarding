# Pipeline Overview

The real boat follows the same pattern as your lab — detect obstacles, decide where to go, drive there — but each step is more involved.

```
Perception → Mapping → Planning → Control
```

On top of all this sits a task manager that decides what the boat should be doing at any given moment (e.g., "navigate through the gates" or "circle the buoy").

## Perception

Figures out what's around the boat using cameras and LiDAR.

1. **YOLO detection** — a neural network looks at camera frames and draws bounding boxes around objects it recognizes (red buoy, green buoy, dock, etc.)
2. **LiDAR fusion** — LiDAR gives a 3D point cloud (thousands of distance measurements). We project these points onto the camera image. Points inside a YOLO bounding box get that box's label. Now we have labeled 3D positions.
3. **Tracking** — raw detections are noisy, so we track obstacles over time, matching new detections to previously seen objects and smoothing their positions.

The output is an `ObstacleMap` message — a list of obstacles, each with a label (what it is) and a position (where it is).

### Key nodes
- `yolov8_node.py` / `yolov11_all_node.py` — YOLO object detection
- `bbox_project_pcloud` (C++) — fuses camera bounding boxes with LiDAR point clouds
- `object_tracking_map` (C++) — tracks obstacles over time, publishes `ObstacleMap`

## Localization

Figures out where the boat is in the world.

- GPS gives a global position (latitude/longitude, converted to x/y)
- SLAM (Simultaneous Localization and Mapping) refines the position by matching sensor data over time

Together they produce the boat's position and heading in a global coordinate frame.

## Mapping

Combines perception + localization to build a global picture.

- `object_tracking_map` takes local obstacle detections and the robot's position to produce a global obstacle map
- `grid_map_generator` creates an occupancy grid (bird's-eye view where cells are free or occupied) used for path planning

## Planning

Given a goal and a map of obstacles, finds a path.

- `navigation_server` uses A* search on the occupancy grid
- Returns a list of waypoints from the boat's position to the goal, avoiding obstacles

In your lab you drove directly to the midpoint with PID. On the real boat, A* finds a path around obstacles first.

## Control

Follows the planned path.

- `controller_server` uses PID control to follow each waypoint (same idea as your lab)
- Adds potential field obstacle avoidance — if something unexpected is close, steer away in real time
- Outputs velocity commands to the thrusters
