# Python examples

Start a station with [your arm connected](../../docs/getting-started.md).
For testing without hardware, use a [simulated station](../../docs/simulation.md).
From the repository root:

```sh
cd examples/python
uv sync --locked
```

## Read your arm's state

From this directory, run this check with the arm disabled.

```sh
uv run python - <<'PYTHON'
import asyncio
from hebbian_rua import StationClient

async def main():
    async with StationClient() as station:
        state = await station.state()
        for arm in state.arms:
            print(arm.arm, arm.mode, arm.measured_joints)

asyncio.run(main())
PYTHON
```

## Preview and move

[`hello_robot.py`](hello_robot.py) enables the arm, previews a target, moves,
and opens the gripper. Use the [simulation quickstart](../../docs/simulation.md#2-run-hello_robot)
to try it. Check the target and clearance before running on hardware.

## Run an agent

Set `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `GEMINI_API_KEY` in the environment
or `.env`. [`agent_quickstart.py`](agent_quickstart.py) enables the arm and gives
the model motion tools. Release any active control lease before starting.

```sh
uv run agent_quickstart.py "Describe the robot state. Keep the arm still."
```

Run `uv run agent_quickstart.py --help` for arm, camera, station, and model
options. See [Write your own loop](../../docs/write-your-own-loop.md) and
[Model output](../../docs/model-output.md) when adapting the example.
