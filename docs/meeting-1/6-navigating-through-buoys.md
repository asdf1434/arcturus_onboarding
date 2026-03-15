# Navigating Through Buoys

Now let's combine waypoint following with buoy detection to navigate a course.

## Launch with a buoy course

```bash
ros2 launch all_seaing_lab sim.launch.py course:=course_1.yaml
```

You should see red and green buoys in RViz forming a path. The buoy positions are published on two topics:
- `/red_buoys` — red buoy positions
- `/green_buoys` — green buoy positions

Both are `Marker` messages with a `points` array. The points are in order, so `red_buoys.points[0]` pairs with `green_buoys.points[0]`, etc.

## Task: Navigate through buoy pairs

Create a new node `follow_path.py` that:

1. Subscribes to the buoy topics to get positions
2. Computes the midpoint between each red-green buoy pair
3. Uses your waypoint follower to navigate to each midpoint in sequence
4. Detects when the boat has arrived (within some distance threshold), then moves on to the next pair
5. Stops when all pairs are passed

To check if a buoy pair is "ahead" of the boat, compare its position to the boat's current position and heading direction.

Don't forget to register the new node in `CMakeLists.txt` and add it to the launch file.
