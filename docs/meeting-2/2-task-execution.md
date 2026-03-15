# Task Execution

Everything in the pipeline overview answers "how do we get from A to B." Tasks answer "what are A and B?"

Each competition task (entry gates, follow the path, docking, etc.) is a separate Python class that:
1. Reads the obstacle map
2. Decides what to do (find the next buoy pair, pick a dock slot, etc.)
3. Sends goal positions to the planner

## TaskServerBase

Every task inherits from `TaskServerBase` (defined in `all_seaing_common`). You implement two methods:

- `init_setup()` — runs once when the task starts. Use this to find initial buoys, set up state, etc.
- `control_loop()` — runs repeatedly. This is where the task logic lives: track buoys, send navigation goals, check if done.

`TaskServerBase` gives you helpers:
- `move_to_point(point)` — send the boat to a position using the navigation stack
- `move_to_waypoint(waypoint)` — similar but with heading control
- `get_robot_pose()` — returns (x, y, heading) in global coordinates
- `mark_successful()` — signal that the task is done
- `send_vel_cmd(x, y, angular)` — send velocity commands directly (bypasses planning)

## Task manager (run_tasks.py)

`run_tasks.py` is the state machine that orchestrates everything:
- Decides which task to run next
- Sends start/stop signals to each task's action server
- Handles interrupts (e.g., harbor alert — drop the current task and respond to a signal, then return)

Each task runs as a ROS 2 action server. The task manager is the action client that controls them.

## How a task runs, end to end

1. `run_tasks.py` sends a goal to the task's action server
2. The task's `init_setup()` runs — finds buoys, computes first waypoint
3. `control_loop()` runs repeatedly:
   - Reads the obstacle map for updated buoy positions
   - Computes the next goal (e.g., midpoint between buoy pair)
   - Calls `move_to_point()` to navigate there
   - Checks if the goal is reached, transitions to next state
4. Task calls `mark_successful()` when done
5. `run_tasks.py` moves on to the next task
