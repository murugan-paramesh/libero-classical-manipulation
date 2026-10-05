# LIBERO-10 Classical Manipulation with Intrinsic Core

This project implements a classical, non-learned manipulation pipeline for the LIBERO-10 benchmark. Intrinsic Core is used for forward and inverse kinematics, collision checking, and motion planning, while the planned motions are executed in the LIBERO/MuJoCo environment.

## Results

I evaluated the final system on all 10 LIBERO-10 tasks using 10 evaluation initial states for each task, giving a total of 100 episodes. These evaluation states were kept separate from the states used during development.

The final system successfully completed **93 out of 100 episodes**.

| Task | Successful Episodes |
|---|---:|
| Task 0 | 10/10 |
| Task 1 | 8/10 |
| Task 2 | 10/10 |
| Task 3 | 10/10 |
| Task 4 | 10/10 |
| Task 5 | 9/10 |
| Task 6 | 10/10 |
| Task 7 | 9/10 |
| Task 8 | 10/10 |
| Task 9 | 7/10 |
| **Overall** | **93/100** |

The evaluation used a fixed seed of 0 and a maximum budget of 1,200 control steps per episode. Task success was determined using LIBERO's task-success predicates.

All robot motions were planned through Intrinsic Core. During the final evaluation, the system made **1,343 `PlanTrajectory` calls**, with an average planning time of approximately **22 ms** and no planning timeouts.

One of the more challenging tasks was Task 3, which requires placing a bowl inside the bottom drawer and then closing it. I handled this by first opening the drawer fully, placing the bowl inside, and then closing the drawer using a pitched push on the handle. IK re-seeding and a small set of retry variations were used when the initial motion was not feasible. This approach completed all 10 evaluation states for Task 3.

The complete evaluation records are available in `evaluations/final/`. I have also included one representative successful video for each of the 10 tasks in `evidence/videos/`.

### Limitation

The current implementation uses object and articulation poses directly from the simulator rather than estimating them from camera images. The focus of this project is therefore on the classical manipulation stack—kinematics, collision checking, motion planning, and execution—rather than visual perception.

## Project Structure

I kept the repository structure simple so that the implementation and evaluation can be followed easily.

```text
src/libero_intrinsic/
    env/            LIBERO environment setup and robot execution
    model/          Robot and scene model conversion
    intrinsic/      Communication with Intrinsic Core
    skills/         Pick, place, push and articulation skills
    eval/           Task execution and evaluation

intrinsic_stack/cc/
    C++ planner server used to connect the project with Intrinsic Core

scripts/
    Scripts to set up, validate and run the tasks

configs/
    Evaluation configuration

tests/
    Unit tests used to check the implementation

docs/
    Architecture and final results

evaluations/final/
    Episode records from the final 100-episode evaluation

evidence/videos/
    Example successful runs for each LIBERO-10 task
```

LIBERO and Intrinsic Core are external dependencies and are not included as full repositories here. The required versions and setup steps are given below.

## Setup

The project was tested on Ubuntu 24.04 (x86-64). Approximately 30 GB of free disk space is recommended for building Intrinsic Core with Bazel.

The exact versions of Intrinsic Core and LIBERO used for the final evaluation are provided below so that the setup can be reproduced.

### Installation

First, clone the versions of Intrinsic Core and LIBERO used for the final evaluation.

```bash
mkdir -p third_party
```


```bash
# Clone Intrinsic Core
git clone https://github.com/intrinsic-ai/intrinsic-core third_party/intrinsic-core
git -C third_party/intrinsic-core checkout c61bf075f2335371c6367b61117e8a62bb960c3b

# Clone LIBERO
git clone https://github.com/Lifelong-Robot-Learning/LIBERO third_party/LIBERO
git -C third_party/LIBERO checkout 8f1084e3132a39270c3a13ebe37270a43ece2a01

# Clone Google API proto definitions
git clone https://github.com/googleapis/googleapis.git third_party/googleapis
git -C third_party/googleapis checkout 03a91044136a014466d4293eb1fe91f2b02075d2
```

Create the Python environment and install the required dependencies.

Install `uv` first if it is not already available:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```


```bash
uv venv --python 3.10 .venv
source .venv/bin/activate
uv pip install -r requirements.txt

echo "$PWD/third_party/LIBERO" > .venv/lib/python3.10/site-packages/libero_src.pth

mkdir -p ~/.libero
python scripts/write_libero_config.py

sudo apt-get install -y libegl1 libegl-mesa0 libgl1-mesa-dri libosmesa6
```

Build the Intrinsic Core planner server.

```bash
ln -sfn "$PWD/intrinsic_stack/cc" third_party/intrinsic-core/libero_bridge

(cd third_party/intrinsic-core && \
bazel --output_base=$HOME/bazel_out build --jobs=2 \
    //libero_bridge:libero_planner_server)
```

Generate the Python interfaces for the Intrinsic APIs and run the unit tests.

```bash
python scripts/gen_intrinsic_protos.py
python -m pytest -q
```

## Running the Project

After completing the setup, I recommend first checking the connection between LIBERO and Intrinsic Core.

### Validate the Kinematics

```bash
python scripts/validate_kinematics.py --task 0
```

This checks the forward and inverse kinematics between Intrinsic Core and MuJoCo.

### Run a Simple Planned Motion

```bash
python scripts/demo_motion.py --task 0 --init 0
```

This executes a motion planned through Intrinsic Core inside the LIBERO environment.

### Run an Individual Task

```bash
python scripts/run_task.py --task 0 --inits 0
```

The task number can be changed from `0` to `9` to run any of the LIBERO-10 tasks.

Multiple initial states can also be provided:

```bash
python scripts/run_task.py --task 0 --inits 0 1 2
```

### Run the Full Evaluation

```bash
python scripts/evaluate.py --config configs/eval_frozen.yaml
```

This runs the evaluation using the settings defined in `configs/eval_frozen.yaml`.

## Evaluation and Evidence

The final evaluation records are included in:

```text
evaluations/final/
```

These records contain the episode-level results used to report the final **93/100** result.

For an easier visual check of the system, I have also included one representative successful execution for each LIBERO-10 task in:

```text
evidence/videos/
```

The videos are named:

```text
task_0.mp4
task_1.mp4
task_2.mp4
task_3.mp4
task_4.mp4
task_5.mp4
task_6.mp4
task_7.mp4
task_8.mp4
task_9.mp4
```

These videos are representative demonstrations of how the classical manipulation pipeline executes the different tasks. The reported **93/100** quantitative result is based on the complete evaluation records rather than only these example videos.

## Quick Verification

After completing the setup, the main parts of the implementation can be checked using:

```bash
# Run the unit tests
python -m pytest -q

# Check FK and IK between Intrinsic Core and MuJoCo
python scripts/validate_kinematics.py --task 0

# Run one Intrinsic-planned motion
python scripts/demo_motion.py --task 0 --init 0

# Run one complete LIBERO task
python scripts/run_task.py --task 0 --inits 0
```

## Dependencies

The main versions used for the final evaluation are listed below.

| Component | Version / Configuration |
|---|---|
| Intrinsic Core | Commit `c61bf075f2335371c6367b61117e8a62bb960c3b` |
| LIBERO | Commit `8f1084e3132a39270c3a13ebe37270a43ece2a01` |
| Google APIs | Commit `03a91044136a014466d4293eb1fe91f2b02075d2` |
| Bazel | 8.8.1 |
| Python | 3.10 |
| Robot controller | robosuite `OSC_POSE`, 20 Hz |

The remaining Python dependencies are listed in `requirements.txt`.

## Intrinsic Core Integration

The connection between LIBERO and Intrinsic Core is implemented through the C++ planner server in:

```text
intrinsic_stack/cc/libero_planner_server.cc
```

The server uses Intrinsic Core's `MotionPlannerService` and `ObjectWorldService` through gRPC.

The main Intrinsic operations used by the project are:

- `ComputeFk` for forward kinematics
- `ComputeIk` for inverse kinematics
- `CheckCollisions` for collision checking
- `PlanTrajectory` for robot motion planning

The LIBERO scene is converted into a representation that can be used by the Intrinsic planning stack. During execution, the required robot and object states are sent to the planner, and the resulting trajectories are converted into commands for LIBERO's `OSC_POSE` controller.

This keeps the manipulation pipeline classical: task execution is built from kinematics, collision checking, motion planning, and predefined manipulation skills rather than an end-to-end learned policy.

A more detailed view of the integration is available in `docs/architecture.md`.
