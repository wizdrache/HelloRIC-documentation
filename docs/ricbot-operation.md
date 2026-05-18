# RICbot Operation

Docker-based ROS 2 setup for running the RICbot from a control machine. The
robot runs only the hardware drivers, while the control machine runs Zenoh
networking, RViz, mapping, map saving, and Nav2.

## Requirements

- Docker and Docker Compose.
- `tmux` on the control machine.
- X11 or Xwayland forwarding for RViz. The environment may need to be adjusted
  depending on the operating system and display setup.
- SSH access to the robot.
- The robot and control machine must be on the same network.

If Docker reports that `/tmp/.X11-unix` is not shared from the host, allow
`/tmp` in Docker Desktop file sharing settings or run the setup on a native
Linux Docker engine. The RViz container needs that socket to open its window.
The launcher scripts create `/tmp/.docker.xauth` automatically so RViz can
authenticate to the host X11/Xwayland display, and allow local root X11 access
with `xhost` when the command is available.

## Robot Startup

Start the control-machine workflow first, either [map recording](#create-a-map)
or [navigation](#navigate-with-a-saved-map), so the `zenoh_router` container is
already running before the robot driver tries to connect.

Find the control machine IP address with `ip a` or `ifconfig`. Then SSH into
the robot and start the robot driver container:

```bash
bash start_ricbot.bash <control-machine-ip>
```

The script sets the robot container's Zenoh endpoint to the Zenoh router on the
control machine. If you need to set the endpoint manually, use:

```bash
ZENOH_CONNECT_ENDPOINT=tcp/<control-machine-ip>:7447 docker compose up ricbot
```

## Create a Map

From the repository root on the control machine, run:

```bash
bash map_recorder.bash
```

This opens a `tmux` session with two panes:

- Left pane: starts `zenoh_router`, RViz, Cartographer, and keyboard teleop.
- Right pane: opens a shell inside the Cartographer/Nav2 container for saving
  the map.

Once the left pane has started `zenoh_router`, start the robot driver from the
robot using the steps in [Robot Startup](#robot-startup).

Use the teleop pane to drive the robot through the area. The teleop container
starts with conservative speeds, currently `0.2` linear and `0.8` angular.

When the map looks correct in RViz, save it from the right pane:

```bash
ros2 run nav2_map_server map_saver_cli -f /map_data/<map_name>
```

This writes `<map_name>.yaml` and the map image into
`ricbot_navigation/maps/` on the control machine.

## Navigate With a Saved Map

From the repository root on the control machine, run:

```bash
bash navigation.bash
```

By default this loads:

```text
/map_data/rh1_eg_new_map.yaml
```

To use another map in `ricbot_navigation/maps/`, pass either the map base name
or the full container path:

```bash
bash navigation.bash rh1_eg_map
bash navigation.bash /map_data/rh1_et1_map.yaml
```

You can also set the map with an environment variable:

```bash
NAV_MAP_YAML=/map_data/rh1_eg_map.yaml bash navigation.bash
```

The navigation script opens a `tmux` session named `navigation`:

- Left pane: `zenoh_router` and RViz.
- Right pane: Nav2 using the selected map.

Once the left pane has started `zenoh_router`, start the robot driver from the
robot using the steps in [Robot Startup](#robot-startup).

In RViz, set the robot's initial pose with `2D Pose Estimate`. The arrow should
point in the robot's forward direction. Once localization matches the map, send
a goal with `2D Goal Pose`.

## Useful Paths

- Maps: `ricbot_navigation/maps/`
- Nav2 parameters: `ricbot_navigation/config/nav2_params.yaml`
- Cartographer configuration: `ricbot_navigation/config/cartographer_2d.lua`
- Docker services: `compose.yml`

## Stopping

Detach from `tmux` with `Ctrl-b`, then `d`.

Stop the control-machine-side containers with:

```bash
docker compose down
```
