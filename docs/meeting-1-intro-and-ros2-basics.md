# Meeting 1: Intro + ROS 2 Basics + First Autonomy

## Part 1: Environment Setup

### Docker installation

We develop inside a Docker container — think of it as a lightweight virtual machine with all our tools pre-installed, so everyone has the same environment.

1. Install [Git](https://git-scm.com/downloads) and [Docker](https://www.docker.com/products/docker-desktop/) for your OS
2. Clone the Docker environment:
   ```bash
   git clone --recurse-submodules git@github.com:ArcturusNavigation/onboarding_lab_docker.git
   ```
3. `cd` into the cloned directory and start the container:
   ```bash
   docker compose up
   ```
4. In a separate terminal, open a shell inside the container:
   ```bash
   docker compose exec arcturus bash
   ```

> Apple Silicon users: edit `docker-compose.yml` and change the `image` field to `popkarsd/arcturus_onboarding:aarch64`

Some things to know:
- Files in `/home/arcturus` persist between restarts. Changes elsewhere are lost.
- Always run `docker compose down` before restarting.
- Login credentials: username `arcturus`, password `arcturus`.

### Building the workspace

Inside the container:

```bash
cd ~/arcturus/dev_ws
colcon build --symlink-install
source install/setup.bash
```

You need to run `source install/setup.bash` every time you open a new terminal.

### Accessing the GUI

Open a browser and go to: `http://localhost:6080/vnc.html?resize=remote`

This gives you a desktop inside the container where you can see the visualization.

---

## Part 2: Running the Simulation

In a terminal inside the NoVNC desktop:

```bash
cd ~/arcturus/dev_ws
source install/setup.bash
ros2 launch all_seaing_lab sim.launch.py
```

You should see the boat in RViz. Drive it around with the keyboard:

| Key | Action |
|-----|--------|
| W | Forward |
| Q | Turn left |
| E | Turn right |

It has momentum and drag, so it doesn't stop instantly. Get a feel for how it moves.

To exit: close the RViz window and press `Ctrl+C` in the terminal.

---

## Part 3: Exploring the System

With the simulation still running, open a new terminal and try:

```bash
ros2 topic list
```

This shows you all the message channels currently active. You'll see things like `/cmd_vel`, `/boat_pose`, `/world_map`.

Now try:

```bash
ros2 topic echo /boat_pose
```

Drive the boat around while this is running. You'll see position and orientation values changing in real time. This is the boat's position being sent as a stream of messages.

Try also:

```bash
ros2 topic hz /cmd_vel
```

This shows how frequently velocity commands are being sent.

### What's actually going on

When you launched the sim, several programs started running at the same time:
- `sim_engine` — simulates the boat's physics
- `teleop_controller` — reads your keyboard input and sends velocity commands
- `buoy_course` — publishes buoy positions (not visible yet since we didn't load a course)

In ROS 2, each of these programs is called a *node*. Nodes communicate by sending messages through named channels called *topics*. When a node sends a message, that's called *publishing*. When a node listens for messages, that's called *subscribing*.

So what's happening is:
- The keyboard node publishes velocity commands to the `/cmd_vel` topic
- The simulation node subscribes to `/cmd_vel` to know how to move the boat
- The simulation node publishes the boat's position to `/boat_pose`

The key thing: the simulation doesn't know or care where the velocity commands come from. It just listens to the topic. So if we write our own program that publishes to the same topic, the simulation will follow our commands instead. That's how we'll go from keyboard control to autonomous control.

---

## Part 4: Writing Your First Node

Let's write a program that moves the boat forward automatically.

Create a new file `all_seaing_lab/all_seaing_lab/waypoint_follower.py`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class WaypointFollower(Node):
    def __init__(self):
        super().__init__("waypoint_follower")
        self.cmd_vel_pub = self.create_publisher(Twist, "cmd_vel", 10)
        self.timer = self.create_timer(0.5, self.timer_callback)

    def timer_callback(self):
        msg = Twist()
        msg.linear.x = 1.0  # move forward
        self.cmd_vel_pub.publish(msg)


def main(args=None):
    rclpy.init(args=args)
    node = WaypointFollower()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == "__main__":
    main()
```

Walking through this:
- `create_publisher(Twist, "cmd_vel", 10)` — creates a publisher that sends `Twist` messages (velocity commands) to the `cmd_vel` topic
- `create_timer(0.5, self.timer_callback)` — calls `timer_callback` every 0.5 seconds
- `rclpy.spin(node)` — keeps the program running and processing messages

### Register the node

Add the file to `CMakeLists.txt` so ROS 2 can find it:

```cmake
install(PROGRAMS
    ${PROJECT_NAME}/buoy_course.py
    ${PROJECT_NAME}/pid_controller.py
    ${PROJECT_NAME}/sim_engine.py
    ${PROJECT_NAME}/teleop_controller.py
    ${PROJECT_NAME}/waypoint_follower.py   # <-- add this line
    DESTINATION lib/${PROJECT_NAME}
)
```

Make the file executable, then build:

```bash
chmod +x all_seaing_lab/all_seaing_lab/waypoint_follower.py
cd ~/arcturus/dev_ws
colcon build --symlink-install
source install/setup.bash
```

Run the sim in one terminal, then in another:

```bash
ros2 run all_seaing_lab waypoint_follower.py
```

Press `Enter` first to disable teleop so it doesn't fight with your node. The boat should start moving forward on its own.

---

## Part 5: Waypoint Following with PID

Moving forward is a start, but we want the boat to go to a specific point. For that we need to:

1. Know where the boat is — subscribe to `/boat_pose`
2. Know where it should go — subscribe to `/goal_pose` (we'll set this by clicking in RViz)
3. Compute how to get there — use PID control

### What is PID?

PID is a control algorithm. The intuition is simple:
- If you're far from the target, steer hard toward it (P — proportional)
- If you've been slightly off for a while, nudge a bit more (I — integral)
- If you're approaching fast, ease off so you don't overshoot (D — derivative)

The lab provides a `PIDController` class in `pid_controller.py` — you don't need to implement PID from scratch.

### Task: Add subscriptions and PID control

Update your `waypoint_follower.py` to:

1. Subscribe to `/boat_pose` (type `PoseStamped`) — the boat's current position and orientation
2. Subscribe to `/goal_pose` (type `PoseStamped`) — set by clicking "2D Goal Pose" in RViz
3. Use PID controllers to compute velocity commands that drive the boat toward the goal

You'll need three PID controllers:
- x position — controls forward/backward speed
- y position — controls lateral speed
- heading (theta) — controls rotation to face the goal

Hints:
- Use `euler_from_quaternion` from `tf_transformations` to convert orientation to a heading angle
- Heading wraps around (359° is close to 1°), so use `CircularPID` for the heading controller
- Start with just P (set Ki=0, Kd=0), get it working, then tune

Declare PID parameters so you can tune them without restarting:

```python
self.declare_parameter("Kpid_x", [1.0, 0.0, 0.0])
self.declare_parameter("Kpid_y", [1.0, 0.0, 0.0])
self.declare_parameter("Kpid_theta", [1.0, 0.0, 0.0])
```

Then tune from the command line:
```bash
ros2 param set /waypoint_follower Kpid_x [2.0,0.0,0.1]
```

### Converting PID output to velocity

The PID controllers give you corrections in the world frame (north/east), but the boat moves in its own frame (forward/sideways). You need to rotate based on the boat's heading:

```python
cos_h = math.cos(heading)
sin_h = math.sin(heading)
cmd_vel.linear.x = cos_h * pid_x_output + sin_h * pid_y_output
cmd_vel.linear.y = -sin_h * pid_x_output + cos_h * pid_y_output
cmd_vel.angular.z = pid_theta_output
```

Don't worry if this doesn't make sense right away — just use the formula and see if the boat goes to the right place. If it goes in the wrong direction, try flipping signs.

### Testing

1. Launch the sim: `ros2 launch all_seaing_lab sim.launch.py`
2. Run your node: `ros2 run all_seaing_lab waypoint_follower.py`
3. Press `Enter` to disable teleop
4. In RViz, click "2D Goal Pose" and click on the map to set a waypoint
5. The boat should navigate to the waypoint

If the boat oscillates wildly, reduce Kp. If it never quite reaches the goal, increase Kp or add some Ki.

---

## Part 6: Navigating Through Buoys

Now let's combine waypoint following with buoy detection to navigate a course.

### Launch with a buoy course

```bash
ros2 launch all_seaing_lab sim.launch.py course:=course_1.yaml
```

You should see red and green buoys in RViz forming a path. The buoy positions are published on two topics:
- `/red_buoys` — red buoy positions
- `/green_buoys` — green buoy positions

Both are `Marker` messages with a `points` array. The points are in order, so `red_buoys.points[0]` pairs with `green_buoys.points[0]`, etc.

### Task: Navigate through buoy pairs

Create a new node `follow_path.py` that:

1. Subscribes to the buoy topics to get positions
2. Computes the midpoint between each red-green buoy pair
3. Uses your waypoint follower to navigate to each midpoint in sequence
4. Detects when the boat has arrived (within some distance threshold), then moves on to the next pair
5. Stops when all pairs are passed

To check if a buoy pair is "ahead" of the boat, compare its position to the boat's current position and heading direction.

Don't forget to register the new node in `CMakeLists.txt` and add it to the launch file.

### Stretch goals

If you get the basic version working:
- Handle the case where the boat drifts off course and needs to re-acquire the next pair
- Visualize the target midpoint as a marker in RViz so you can see where the boat is aiming
- Modify the course or make your own YAML file

---

## Recap

What we covered:
- What Arcturus does and how RoboBoat tasks work
- ROS 2 basics: nodes, topics, publishing, subscribing
- PID control for waypoint following
- Buoy pair navigation — a simplified version of what the real boat does

Next meeting we'll look at how the actual codebase works — the perception pipeline, navigation stack, and task execution system that runs on the real boat.
