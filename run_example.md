# Go2 MJPC (iLQR) simulation — run notes

All commands run **inside the container** (`cd ~ && . docker_ros_humble_v1.bash`),
from a terminal on the VM desktop (GUI needs `DISPLAY=:1`).

## Run — pick ONE (both open the same Go2 GUI)

### A. Standalone GUI (simplest)
```bash
cd ~/mujoco_mpc_ws/build/bin && ./mjpc --task="Quadruped Flat"
```

### B. Deploy repo Python example (drives ui_agent_server over gRPC)
```bash
source ~/mujoco_mpc_ws/venv/bin/activate   # REQUIRED: venv has mujoco==3.2.3 to match the server
cd ~/mujoco_mpc_ws/mujoco_mpc_deploy && python examples/mjpc_gui.py   # Ctrl+C to stop
```
GUI first shows "Acrobot" for a moment, then switches to Quadruped Flat — that's normal.

## GUI: make the Go2 trot
1. Agent panel → **Planner = iLQG** (iLQR from the paper). Make sure **Action** is on.
2. Task panel → **Gait switch = Manual** (default is Automatic, which picks the gait from CoM speed).
3. Task panel → **Gait = Trot** (also sets duty ratio 0.45 / cadence 2 Hz / amplitude 0.03).
4. Make it move:
   - **Modes = Walk**, then raise **Walk speed** (e.g. 0.3–0.5) / **Walk turn** for turning; or
   - stay in **Modes = Quadruped** and drag the green goal sphere:
     double-click it to select, then **Ctrl + right-drag** to move it on the ground.
5. Tune **Cadence / Amplitude / Duty ratio** sliders to change the step.

## Layout
```
~/mujoco_mpc_ws/
  mujoco_mpc         -> ~/mac_src/mpc/MuJoCoMPC/mujoco_mpc         (fork, branch go2)
  mujoco_mpc_deploy  -> ~/mac_src/mpc/MuJoCoMPC/mujoco_mpc_deploy
  build/             clang-14 build (mujoco_mpc/build symlinks here); bin/mjpc, bin/ui_agent_server
  build_gcc/         old GCC build — quadruped segfaults, safe to delete
  deps_src/          downloaded deps (gRPC, MuJoCo 3.2.3, abseil...) reused by builds
  venv/              python venv: mujoco==3.2.3 + mujoco_mpc
  build_mjpc.sh      rebuild script (clang-14, ccache)
```

## Rebuild after editing the fork
```bash
~/mujoco_mpc_ws/build_mjpc.sh                         # full C++ build (log: ~/mujoco_mpc_ws/build_mjpc.log)
# faster: cd ~/mujoco_mpc_ws/build && cmake --build . --target mjpc ui_agent_server agent_server -j4
# if ui_agent_server changed, refresh the python package:
source ~/mujoco_mpc_ws/venv/bin/activate
cd ~/mujoco_mpc_ws/mujoco_mpc/python && pip install --no-deps --no-build-isolation .
```

## Known issues / gotchas
- Go2 lies flat / legs splay: upstream commit e00c47a removed `biastype="affine"` from
  `mjpc/tasks/quadruped/go2.xml`, so actuators became constant torque (60*ctrl) instead of PD
  (kp=60, kd=5). Fixed in the fork's working tree; build copy (`build/mjpc/tasks/...`) and venv
  copy (`venv/.../mujoco_mpc/mjpc/tasks/...`) patched too.
- GCC 11 (aarch64) build segfaults in `QuadrupedFlat` (garbage gait index) → build with clang-14.
- "Quadruped Hill" task still segfaults (A1 terrain task, not needed).
- System python has mujoco 3.8.0; using it with this server gives
  `Model and executable have different number of ints in mjModel` + crash → always use the venv.
- Container DNS is flaky for curl/git: prefix with `RES_OPTIONS=single-request-reopen`.
- System pip: setuptools pinned to 79.0.1 (colcon needs <80, grpcio-tools needs >=77).
- Hardware path (`examples/go2.py`) needs unitree_sdk2py + ROS1 rospy mocap → not usable as-is on Humble.
