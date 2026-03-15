# Waypoint Following with PID

Moving forward is a start, but we want the boat to go to a specific point.

## Step 1: Know where the boat is

First, let's add a subscriber so our node can read the boat's position. Add these to your `waypoint_follower.py`:

```python
from geometry_msgs.msg import Twist, PoseStamped
from tf_transformations import euler_from_quaternion
```

In `__init__`, add a subscriber and some variables to store the boat's state:

```python
self.boat_x = 0.0
self.boat_y = 0.0
self.boat_heading = 0.0

self.create_subscription(PoseStamped, "boat_pose", self.boat_callback, 10)
```

Then add the callback — this runs every time a new position message arrives:

```python
def boat_callback(self, msg):
    self.boat_x = msg.pose.position.x
    self.boat_y = msg.pose.position.y
    orientation = msg.pose.orientation
    _, _, self.boat_heading = euler_from_quaternion(
        [orientation.x, orientation.y, orientation.z, orientation.w]
    )
```

`euler_from_quaternion` converts the orientation (stored as a quaternion — don't worry about what that means) into a heading angle in radians.

Rebuild, run, and add a print statement in `timer_callback` to verify you're getting position data:

```python
def timer_callback(self):
    self.get_logger().info(f"Boat at ({self.boat_x:.2f}, {self.boat_y:.2f}), heading {self.boat_heading:.2f}")
    # ... rest of your code
```

## Step 2: Know where to go

Now add a subscriber for the goal position. In RViz, there's a "2D Goal Pose" button that publishes a `PoseStamped` to `/goal_pose` when you click on the map.

Add to `__init__`:

```python
self.goal_x = None
self.goal_y = None

self.create_subscription(PoseStamped, "goal_pose", self.goal_callback, 10)
```

```python
def goal_callback(self, msg):
    self.goal_x = msg.pose.position.x
    self.goal_y = msg.pose.position.y
    self.get_logger().info(f"New goal: ({self.goal_x:.2f}, {self.goal_y:.2f})")
```

Rebuild and test — click "2D Goal Pose" in RViz and you should see the goal position printed.

## Step 3: Add PID control

Now we need to actually drive toward the goal. PID control is the idea that:
- If you're far from the target, steer hard toward it (P)
- If you've been slightly off for a while, nudge a bit more (I)
- If you're approaching fast, ease off so you don't overshoot (D)

The lab provides `PIDController` and `CircularPID` classes. Add these imports:

```python
import math
from all_seaing_lab.pid_controller import PIDController, CircularPID
```

Set up three controllers in `__init__` — one for x, one for y, one for heading:

```python
self.declare_parameter("Kpid_x", [1.0, 0.0, 0.0])
self.declare_parameter("Kpid_y", [1.0, 0.0, 0.0])
self.declare_parameter("Kpid_theta", [1.0, 0.0, 0.0])

kx = self.get_parameter("Kpid_x").value
ky = self.get_parameter("Kpid_y").value
kt = self.get_parameter("Kpid_theta").value

self.pid_x = PIDController(*kx)
self.pid_y = PIDController(*ky)
self.pid_theta = CircularPID(*kt)  # CircularPID handles angle wrapping

self.prev_time = self.get_clock().now()
```

## Step 4: Wire it together

Now update `timer_callback` to compute velocity commands using PID. Replace the old constant-velocity code:

```python
def timer_callback(self):
    if self.goal_x is None:
        return  # no goal set yet

    # Time since last update
    now = self.get_clock().now()
    dt = (now - self.prev_time).nanoseconds / 1e9
    self.prev_time = now

    # Set PID targets to the goal position
    self.pid_x.set_setpoint(self.goal_x)
    self.pid_y.set_setpoint(self.goal_y)

    # Point toward the goal
    goal_heading = math.atan2(
        self.goal_y - self.boat_y,
        self.goal_x - self.boat_x
    )
    self.pid_theta.set_setpoint(goal_heading)

    # Update PIDs with current position
    self.pid_x.update(self.boat_x, dt)
    self.pid_y.update(self.boat_y, dt)
    self.pid_theta.update(self.boat_heading, dt)

    # Get PID outputs (these are in the world frame)
    pid_x_output = self.pid_x.get_effort()
    pid_y_output = self.pid_y.get_effort()
    pid_theta_output = self.pid_theta.get_effort()

    # Convert from world frame to boat frame
    cos_h = math.cos(self.boat_heading)
    sin_h = math.sin(self.boat_heading)

    msg = Twist()
    msg.linear.x = cos_h * pid_x_output + sin_h * pid_y_output
    msg.linear.y = -sin_h * pid_x_output + cos_h * pid_y_output
    msg.angular.z = pid_theta_output
    self.cmd_vel_pub.publish(msg)
```

The world-to-boat frame conversion might look confusing — the boat thinks in terms of "forward" and "sideways" but the PID computes corrections in terms of "north" and "east." The cos/sin rotation translates between them.

## Step 5: Test and tune

1. Launch the sim: `ros2 launch all_seaing_lab sim.launch.py`
2. Run your node: `ros2 run all_seaing_lab waypoint_follower.py`
3. Press `Enter` to disable teleop
4. In RViz, click "2D Goal Pose" and click on the map
5. The boat should navigate to the waypoint

It probably won't work perfectly the first time. Tune the PID gains without restarting:

```bash
ros2 param set /waypoint_follower Kpid_x [2.0,0.0,0.1]
```

If the boat oscillates wildly, reduce the first number (Kp). If it never quite reaches the goal, increase Kp or add some Ki (the second number).
