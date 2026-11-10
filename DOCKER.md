# Docker Cheatsheet

Commands for working with the `steve_ros2` container and fixing the usual problems.
Run them **on the host** unless the heading says *inside the container*.

---

## Daily use

| Task | Command |
|---|---|
| Build the image (once, or after editing `.devcontainer/Dockerfile`) | `./scripts/build_docker.sh` |
| Start a **new** container | `./scripts/run_docker.sh` (only once!) |
| Find the running container's name | `docker ps --filter ancestor=steve_ros2:latest` |
| Open another shell in it | `docker exec -it <name> bash` |
| Leave a shell | `exit` (the container stops when the **first** shell exits) |

`run_docker.sh` always starts a **new** container with a random name (e.g. `sharp_nash`).
Run it once; for every extra terminal, find that container with `docker ps` and
`docker exec` into it. **Always use one container for everything** (sim, `ros2 topic echo`, RViz, ...).
See [Topics listed but no data](#topics-are-listed-but-ros2-topic-echo-shows-no-data).

Shortcut: store the name once per host terminal, then use `$C` in the commands below:

```bash
C=$(docker ps -q --filter ancestor=steve_ros2:latest | head -1)
docker exec -it $C bash
```

In the tables below, `<name>` is the container name or ID from `docker ps`.

### Inside the container, first time / after pulling changes

```bash
./scripts/repos.sh                                   # clone/update packages into src/
sudo apt-get update                                  # the image ships without apt package lists
rosdep install --from-paths src --ignore-src -y -r   # missing apt deps (-r: skip unresolvable keys)
colcon build --symlink-install
source install/setup.bash                            # in EVERY new shell
```

- Without `sudo apt-get update`, rosdep fails with `E: Unable to locate package ros-humble-...`.
- Without `-r`, rosdep stops at `Cannot locate rosdep definition for [ament_python]`
  (a wrong key in `steve_pan_tilt_controller/package.xml`) and installs nothing.
- Anything rosdep installs is lost when the container exits. If it installs something every time,
  add that package to `.devcontainer/Dockerfile` and rebuild the image.

Build a single package: `colcon build --symlink-install --packages-select steve_simulation`

### Run the sim

```bash
ros2 launch steve_simulation simulation.launch.py
ros2 launch steve_simulation simulation.launch.py --show-args    # all options
```

---

## Containers and images

| Task | Command |
|---|---|
| Running containers | `docker ps` |
| All containers, including stopped | `docker ps -a` |
| Stop a container | `docker stop <name>` |
| Kill a stuck container | `docker kill <name>` |
| Remove a stopped container | `docker rm <name>` |
| Stop **all** running containers | `docker stop $(docker ps -q)` |
| List images | `docker images` |
| Remove an image | `docker rmi steve_ros2:latest` |
| Clear the build cache (then `./scripts/build_docker.sh` rebuilds from scratch) | `docker builder prune` |
| Shell as root (e.g. to `apt install`) | `docker exec -it -u root <name> bash` |
| See what a container is running | `docker exec <name> ps aux` |
| Container resource usage | `docker stats` |
| Disk used by Docker | `docker system df` |
| Free space (removes stopped containers, dangling images, build cache) | `docker system prune` |

> `apt install` inside the container is lost when the container is removed.
> To keep a package, add it to `.devcontainer/Dockerfile` and rebuild the image.

What survives: everything in the workspace (`src/`, `build/`, `install/`) because
it is mounted from the host at `/home/steve_ros2_ws`. Everything else in the container
is thrown away when it exits (`--rm`).

---

## Troubleshooting

### Topics are listed but `ros2 topic echo` shows no data

Cause: the publisher and subscriber are in **different containers** (or one on the host).
Discovery works over the shared host network, so `ros2 topic list` looks fine, but Fast DDS
sends the data through shared memory (`/dev/shm`), which each container has privately.

```bash
docker ps                                   # more than one steve_ros2:latest? that's it
docker exec <name> pgrep -a gzserver        # which one runs the sim? (prints a line if it does)
docker stop <other-name>                    # close the extra container
docker exec -it <sim-name> bash             # open the extra shell in the sim's container
```

Quick check that data is flowing at all (inside the container):

```bash
ros2 topic echo --once /clock               # sim is running if this prints a time
ros2 topic hz /scan                         # message rate
ros2 topic info -v /scan                    # publishers/subscribers and their QoS
```

### `ros2 topic list` shows nothing / topics missing

```bash
echo $ROS_DOMAIN_ID $ROS_LOCALHOST_ONLY     # must match in every shell
ros2 daemon stop && ros2 daemon start       # stale discovery cache
ros2 node list
ros2 doctor --report                        # network / middleware summary
```

> The VS Code devcontainer (`.devcontainer/devcontainer.json`) sets `ROS_DOMAIN_ID=42`
> and `ROS_LOCALHOST_ONLY=1`; `run_docker.sh` does not. Shells from the two won't see
> each other. Use one or the other.

Forgot `source install/setup.bash`? Then `ros2 launch steve_...` says "package not found".

### Topic has data but it looks frozen / TF errors in RViz

Sim nodes use simulated time. Your own nodes need `use_sim_time:=true`:

```bash
ros2 run <pkg> <node> --ros-args -p use_sim_time:=true
ros2 param get /rviz2 use_sim_time
ros2 run tf2_ros tf2_echo map base_link     # is the transform there?
ros2 run tf2_tools view_frames              # writes frames.pdf of the TF tree
```

### Gazebo / RViz window doesn't open

```bash
echo $DISPLAY                               # inside the container; must not be empty
glxinfo -B | grep "renderer string"         # inside the container; tests the display
```

- `cannot open display` / `Authorization required`: on the host run `xhost +local:docker`
  (only needed if your host's X access list doesn't already allow local clients; check with `xhost`).
- `renderer string: llvmpipe`: no GPU, rendering on the CPU, Gazebo will be very slow.
  The container needs `--device /dev/dri` (already in `run_docker.sh`). Check the host has it: `ls /dev/dri`.

### Gazebo won't start or hangs on launch

```bash
pkill -9 gzserver; pkill -9 gzclient        # inside the container; leftover processes
```

The launch file already does this, but a crashed sim in another shell can still hold port 11345.

### `git clone` / `repos.sh` fails with `Permission denied (publickey)`

Your SSH key isn't registered with GitHub/GitLab. `run_docker.sh` mounts `~/.ssh` from the host,
so fix it on the host:

```bash
ssh -T git@github.com                       # test
ssh-keygen -t ed25519 && cat ~/.ssh/id_ed25519.pub   # then add the key in GitHub settings
```

### Files in the workspace are owned by root

Something was built with `docker exec -u root`. Fix from the host:

```bash
sudo chown -R $(id -u):$(id -g) build install log
```

### `docker: permission denied ... docker.sock`

Your user isn't in the `docker` group: `sudo usermod -aG docker $USER`, then log out and back in.
