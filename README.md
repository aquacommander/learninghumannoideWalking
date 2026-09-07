# LearningHumanoidWalking

## Code structure:
A rough outline for the repository that might be useful for adding your own robot:
```
LearningHumanoidWalking/
├── envs/                      <-- Environment implementations
│   ├── common/
│   │   ├── base_humanoid_env.py   <-- Base class for all humanoid environments
│   │   ├── mujoco_env.py          <-- MuJoCo simulation wrapper
│   │   └── robot_interface.py     <-- Robot state/control abstraction
│   ├── jvrc/                      <-- JVRC robot environments
│   └── h1/                        <-- Unitree H1 robot environment
├── tasks/                     <-- Task definitions (rewards, termination)
├── rl/                        <-- Reinforcement learning
├── robots/                    <-- Robot abstractions (PD control, stepping logic)
├── models/                    <-- MuJoCo model files
└── tests/                     <-- Test suite
```

### Key abstractions:
- **BaseHumanoidEnv**: Common functionality for humanoid environments (observation history, action smoothing, reset logic)
- **BaseTask**: Interface for task implementations (reset, step, calc_reward, done)
- **Reward functions**: Explicit parameter functions in `tasks/rewards.py` for testability

## Requirements:
- Python version: >= 3.10


```bash
$ uv sync
```

## Usage:

Environment names supported:

| Task Description      | Environment name |
| ----------- | ----------- |
| Basic Standing Task   | 'h1' |
| Basic Walking Task   | 'jvrc_walk' |
| Stepping Task (using footsteps)  | 'jvrc_step' |
| Cartpole swing-up  | 'cartpole' |


#### **To train:**

```
$ uv run run_experiment.py train --logdir <path_to_exp_dir> --num_procs <num_of_cpu_procs> --env <name_of_environment>
```

Note: Setting `RAY_ADDRESS=` ensures Ray starts a new local cluster instead of connecting to an existing one.

#### **To play:**

```
$ uv run run_experiment.py eval --logdir <path_to_actor_pt>
```

Or, we could write a rollout script specific to each environment.

#### **Cartpole**

A minimal swing-up task for testing the RL pipeline. The goal is to swing the pole from hanging down to balancing upright.

```
$ uv run run_experiment.py train --env cartpole --n-itr 500 --std-dev 0.15 --learn-std --entropy-coeff 0.01 --minibatch-size 256 --max-traj-len 500 --no-mirror
```

## Configuration

Environment behavior is configured via YAML files in `envs/<robot>/configs/`. Key parameters:

```yaml
# Simulation
sim_dt: 0.001              # Physics timestep
control_dt: 0.025          # Control loop period
obs_history_len: 1         # Observation history length
action_smoothing: 0.5      # Action filtering coefficient

# Task parameters
task:
  goal_height: 0.80        # Target standing height
  swing_duration: 0.75     # Gait swing phase duration
  stance_duration: 0.35    # Gait stance phase duration

# Reward weights (sum to 1.0)
reward_weights:
  foot_frc_score: 0.225
  foot_vel_score: 0.225
  # ... see configs for full list
```

## Adding a new robot

1. Create `envs/<robot>/` directory with:
   - `gen_xml.py` - MJCF generation from URDF
   - `configs/base.yaml` - Configuration
   - `<robot>_env.py` - Environment class

2. Inherit from `BaseHumanoidEnv` and implement:
   ```python
   class MyRobotEnv(BaseHumanoidEnv):
       def _get_default_config_path(self) -> str: ...
       def _build_xml(self) -> str: ...
       def _setup_robot(self) -> None: ...
       def _setup_spaces(self) -> None: ...
       def _get_robot_state(self) -> np.ndarray: ...
       def _get_external_state(self) -> np.ndarray: ...
   ```

3. Register in `envs/__init__.py`:
   ```python
   ENVIRONMENTS = {
       "my_robot": (MyRobotEnv, "my_robot"),
       # ...
   }
   ```

4. Run tests: `uv run pytest tests/ -v

