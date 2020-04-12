# Training with Curriculum Learning

## How-To

Each group of Agents under the same `Behavior Name` in an environment can have
a corresponding curriculum. These curricula are held in what we call a "metacurriculum".
A metacurriculum allows different groups of Agents to follow different curricula within
the same environment.

### Specifying Curricula

In order to define the curricula, the first step is to decide which parameters of
the environment will vary. In the case of the Wall Jump environment,
the height of the wall is what varies. We define this as a `Shared Float Property`
that can be accessed in `SideChannelUtils.GetSideChannel<FloatPropertiesChannel>()`, and by doing
so it becomes adjustable via the Python API.
Rather than adjusting it by hand, we will create a YAML file which
describes the structure of the curricula. Within it, we can specify which
points in the training process our wall height will change, either based on the
percentage of training steps which have taken place, or what the average reward
the agent has received in the recent past is. Below is an example config for the
curricula for the Wall Jump environment.

```yaml
BigWallJump:
  measure: progress
  thresholds: [0.1, 0.3, 0.5]
  min_lesson_length: 100
  signal_smoothing: true
  parameters:
    big_wall_min_height: [0.0, 4.0, 6.0, 8.0]
    big_wall_max_height: [4.0, 7.0, 8.0, 8.0]
SmallWallJump:
  measure: progress
  thresholds: [0.1, 0.3, 0.5]
  min_lesson_length: 100
  signal_smoothing: true
  parameters:
    small_wall_height: [1.5, 2.0, 2.5, 4.0]
```

At the top level of the config is the behavior name. Note that this must be the
same as the Behavior Name in the [Agent's Behavior Parameters](Learning-Environment-Design-Agents.md#agent-properties).
 The curriculum for each
behavior has the following parameters:
* `measure` - What to measure learning progress, and advancement in lessons by.
  * `reward` - Uses a measure received reward.
  * `progress` - Uses ratio of steps/max_steps.
* `thresholds` (float array) - Points in value of `measure` where lesson should
  be increased.
* `min_lesson_length` (int) - The minimum number of episodes that should be
  completed before the lesson can change. If `measure` is set to `reward`, the
  average cumulative reward of the last `min_lesson_length` episodes will be
  used to determine if the lesson should change. Must be nonnegative.

  __Important__: the average reward that is compared to the thresholds is
  different than the mean reward that is logged to the console. For example,
  if `min_lesson_length` is `100`, the lesson will increment after the average
  cumulative reward of the last `100` episodes exceeds the current threshold.
  The mean reward logged to the console is dictated by the `summary_freq`
  parameter in the
  [trainer configuration file](Training-ML-Agents.md#training-config-file).
* `signal_smoothing` (true/false) - Whether to weight the current progress
  measure by previous values.
  * If `true`, weighting will be 0.75 (new) 0.25 (old).
* `parameters` (dictionary of key:string, value:float array) - Corresponds to
  Environment parameters to control. Length of each array should be one
  greater than number of thresholds.

Once our curriculum is defined, we have to use the environment parameters we defined
and modify the environment from the Agent's `OnEpisodeBegin()` function. See
[WallJumpAgent.cs](https://github.com/Unity-Technologies/ml-agents/blob/master/Project/Assets/ML-Agents/Examples/WallJump/Scripts/WallJumpAgent.cs)
for an example.


### Training with a Curriculum

Once we have specified our metacurriculum and curricula, we can launch
`mlagents-learn` using the `–curriculum` flag to point to the config file
for our curricula and PPO will train using Curriculum Learning. For example,
to train agents in the Wall Jump environment with curriculum learning, we can run:

```sh
mlagents-learn config/trainer_config.yaml --curriculum=config/curricula/wall_jump.yaml --run-id=wall-jump-curriculum
```

We can then keep track of the current lessons and progresses via TensorBoard.
