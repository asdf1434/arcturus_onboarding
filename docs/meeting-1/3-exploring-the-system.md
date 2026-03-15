# Exploring the System

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

## What's actually going on

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
