# Your First Node

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

## Register the node

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
