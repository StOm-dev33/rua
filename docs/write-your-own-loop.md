# Write your own loop

Build a loop that gives an LLM robot state and camera images, lets it choose
robot tools, then feeds the outcomes back into its next turn. Your application
owns the model and decides when to stop. RUA supplies the SDK and executes
commands and action sequences under a lease.

- Start with the [Python agent](../examples/python/README.md#run-an-agent) or
  [TypeScript agent](../examples/typescript/agent/README.md). The
  [Rust SDK](../sdk/README.md#model-facing-tools) provides the same model tools.
- Use the same SDK for direct control with
  [`hello_robot.py`](../examples/python/hello_robot.py) on a
  [physical arm](getting-started.md). You can test the example in
  [MuJoCo](simulation.md) without hardware.

The examples below use Python. See the [TypeScript SDK guide](../typescript-client/README.md#write-a-model-loop)
for its equivalent workflow and [model output](model-output.md) for units,
camera images and formatting across SDKs.

## Start with a working example

With the station running, start with the Python SDK's
[read-only state check](../examples/python/README.md#read-your-arms-state).
For motion, [hello_robot.py](../examples/python/hello_robot.py) claims control,
checks the preview, moves to a target and checks the outcome before opening the
gripper. Choose a target appropriate for your arm and verify clearance before
running it. Use the same steps in your loop, supplying your own target.

Use `station.arm("arm")` to read state, enable the arm or stop it.
Use `station.holding_lease(...)` to acquire a lease and send motion through
`lease.arm("arm")`. Check the enable result before moving. The
[control model](control-model.md) covers lease handoff and interruption.

## Commands and outcomes

| Call | Does |
| --- | --- |
| `lease.arm(name).move_to_pose(position, orientation=None, *, max_joint_speed=None)` | Tool position in metres, world frame. A `GraspOrientation(approach, jaw_yaw_degrees)` is an approach (`"down"`, `"forward"` or a vector) plus jaw yaw in degrees. Without one, the tool keeps its current orientation. |
| `station.arm(name).preview_pose(position, orientation=None)` | Moves nothing. Returns a `PoseProjection` whose outcome is `projected` with a pose, `within_tolerance`, pose errors and any constraint reason, or `unavailable` with a reason and no pose. It also includes the available jaw yaws. A request refusal returns `PreviewRefused`. Without an orientation the solver chooses one. |
| `lease.arm(name).move_to_joints(joints, *, max_joint_speed=None)` | Joint targets in radians. |
| `lease.arm(name).rehome()` | Return to the pose saved at enable time. Use `set_gripper` separately to change the jaw target. |
| `lease.arm(name).set_gripper(opening)` | 0 closed, 1 open. Stopping short can indicate contact or a blockage; it does not prove a grasp. |
| `station.arm(name).stop()` | Stop arm and gripper motion and hold with torque enabled. Needs no lease; does not end an agent run. |

Motion commands return a `CommandResult` with an outcome, a detail and,
when admitted, a command record. Check `result.completed` and
`result.was_limited` before sending another action. The
[control model](control-model.md#command-outcomes) explains what the results
mean. For pose targets, use the same explicit orientation in preview and move
to check the intended pose.

## Reading state

`arm.state()` returns an `ArmState`:

```python
snapshot = await arm.state()
snapshot.mode  # ArmMode: disabled, holding, moving, faulted
snapshot.measured_joints  # radians
snapshot.tool.position  # [x, y, z] metres; `tool` is None before a pose exists
snapshot.tool.quaternion  # [x, y, z, w]
snapshot.tool.orientation  # approach, degrees_from_down, jaw_yaw_degrees
snapshot.gripper_opening  # 0 closed to 1 open; None without a gripper
snapshot.fault  # str or None
```

Camera frames are separate: `await station.camera_names()` and
`(await station.camera_jpeg(name)).jpeg`. Frames and state are not time-synchronized.

## Run an action sequence

Use a sequence when the robot should complete several ordered steps before
your application makes its next decision. A normal submission compiles
automatically. To inspect a program before running, use the
[explicit compilation workflow](control-model.md#compile-before-running).

With the arm named `arm` enabled, submit two joint targets from Python:

```python
import asyncio

from hebbian_rua.client.contract.station_api import (
    ActionSequenceJointsMovement,
    ActionSequenceArmAction,
)
from hebbian_rua.client.station_client import StationClient


async def main():
    async with StationClient() as station:
        async with station.holding_lease("action sequence example") as lease:
            result = await lease.arm("arm").run_action_sequence(
                [
                    ActionSequenceArmAction(
                        movement=ActionSequenceJointsMovement(joints=[0.1, 0, 0, 0, 0]),
                        gripper_opening=1.0,
                    ),
                    ActionSequenceArmAction(
                        movement=ActionSequenceJointsMovement(joints=[0.0, 0, 0, 0, 0])
                    ),
                ]
            )
            print(result.phase, result.steps)


asyncio.run(main())
```

Use joint positions that are appropriate for your arm and workspace. A completed
action sequence reports `completed`; a stopped action sequence explains why it stopped.

## With a model: four pieces

`RobotTools(station, lease, arm_name, camera_names)` provides a default set of
model-facing tools. Use them as provided, filter or combine them with your own
tools, or build your own toolset using the direct SDK calls above. The station
applies the same motion checks to each path. For a structured observation with
selected arms, camera frames, and arm revisions, use the SDK's capture operation
over `POST /observations`. You can also read state and cameras separately.

| Piece | Call |
| --- | --- |
| Robot facts for your own prompt | `await robot.context()` |
| Observe: state as text and a JPEG per camera | `await robot.observe()` returns a `RobotObservation` (`.text`, `.cameras`) |
| Tools: `RobotToolSpec` with `name`, `description`, `parameters_schema` (plain JSON schema) | `robot.tools` |
| Execute: carry out one turn's calls | `await robot.execute([RobotToolCall(call_id, name, arguments)])` returns `RobotToolResult`s (`.data`, `.text`, `.cameras`, `.is_error`, `.halt`) |

For OpenAI, Anthropic and Gemini, use the adapters from `hebbian_rua.client.llm.openai`,
`hebbian_rua.client.llm.anthropic` and `hebbian_rua.client.llm.gemini`. They supply native `.tools`,
`.observe(wait_for_recovery=True)` returns a native message with its arm
revisions. Send `observation.message` to the model and pass
`expected_revisions=observation.revisions` to `.execute(response, ...)`.
Append `turn.messages` and stop on `turn.should_stop` or empty `turn.results`.
See the [three-provider example](../examples/python/agent_quickstart.py).
For other SDKs or mixed toolsets, use the provider-neutral types above and route
calls yourself. Stop on a result's `halt` or `interrupted` flag.
For interruptions, `capture_when_ready(robot)` waits for arm and required-camera
recovery and returns a fresh capture with the arm's revision. Pass that revision
as `expected_revision` when executing tools; a command whose arm was interrupted
since is refused as `stale_revision`. `station.ensure_recording()`
starts or joins a recording; use `lease.stop_recording()` when the task
ends so a handover cannot stop the new holder's recording. In TypeScript,
`robot.observeWhenReady()` returns the capture and each arm's revision, the recording
client provides `ensure()`, and `lease.stopRecording()` stops it only while the lease is current.

The API docstrings in [station_client.py](../hebbian_rua/client/station_client.py)
and [robot_tools.py](../hebbian_rua/client/llm/robot_tools.py)
describe behavior at the methods themselves.

The SDK’s default model tools are `action_sequence`, `preview_pose`, `stop`
and `observe`. The single-arm sequence scope supports pose, joint and gripper
actions. Reference agents expose pose and gripper actions and add application
tools such as `finish`.

Configure available actions in code; the same configuration constrains both the
schema and argument parsing:

```python
from hebbian_rua import RobotTools, filter_sequence_actions

robot = RobotTools(station, lease, "arm").map_tools(
    filter_sequence_actions(lambda action: action != "joints")
)
```

Mappings compose in order: each mapping receives the preceding mapping's
capabilities, and tool definitions preserve the returned order. `map_tools`
transforms immutable capability definitions and returns a view sharing
the connection, lease and output formatter. It composes with `with_output_formatter`; neither
changes the original view. TypeScript provides `mapTools` and
`filterSequenceActions`; Rust provides `map_tools` with an owned `FnOnce` mapping.

For multiple arms, replace the sequence capability's scope with
`NamedArmSequence(("left", "right"))`. Each step then has an `arms` map; every
participating target is planned before the step is admitted together. TypeScript
uses `{ kind: "named", arms: ["left", "right"] }`; Rust uses `SequenceScope`.

To replace the default mapping, select capabilities from
`all_robot_tool_capabilities()` (TypeScript: `allRobotToolCapabilities()`), which
also includes standalone movement, gripper and rehome tools. The default mapping
selects sequences, previews, observation and immediate stop. Individual SDK
methods remain available regardless of the model catalog. Model speeds use
degrees per second and are clamped to the profile maximum; SDK station arguments
use SI units.

## Host your application

Run your loop where it can reach the station, with its own model credentials
if needed. A custom planner submits targets through the API. The station and
controller own motion from admission through execution. See [control limits](control-model.md) and the
[simulator interface](simulators.md) for the plant boundary.

For a browser application, proxy station and recording routes through the
application's origin. The station API has no configurable browser CORS policy
and assumes a trusted network.
