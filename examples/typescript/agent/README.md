# TypeScript reference agent

Run an example robot-use agent built with the
[TypeScript SDK](../../../typescript-client/README.md). Give it a task and
the LLM uses robot observations and tools to decide what to do.
The example supports Anthropic, OpenAI, and Gemini.

## Run the agent

Start a station with [your arm connected](../../../docs/getting-started.md).
For testing without hardware, use [MuJoCo](../../../docs/simulation.md). Use the
Node.js version in the repository's `.nvmrc` and the pnpm version in `package.json`.

Set `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `GEMINI_API_KEY` in the environment
or `.env`. From the repository root, enable the arm and try a state-only task.

```sh
cargo run --locked -p hebbian-robot-use-cli -- enable arm
pnpm install --frozen-lockfile
pnpm --filter @hebbian/agent start --max-turns 3 -- "Describe the robot state. Keep the arm still."
```

Run `pnpm --filter @hebbian/agent start --help` for arm, station, provider,
model, camera, and recording options.

## Use the loop in your own code

[`src/loop.ts`](src/loop.ts) exports `runAgent`. See
[Write your own loop](../../../docs/write-your-own-loop.md) for SDK patterns or
[Model output](../../../docs/model-output.md) to customize what the model sees.
