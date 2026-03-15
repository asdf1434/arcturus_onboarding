# Running the Simulation

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
