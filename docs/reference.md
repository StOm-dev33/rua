# Station reference

Follow [getting started](getting-started.md) to set up a physical arm.
Use [simulation](simulation.md) to test without hardware. The
[OpenAPI contract](../schemas/openapi.json) describes the station API.

## Station server

The station serves state, commands, events, camera access and recordings.
`hebbian-robot-use` is a [separate client CLI](../cli/README.md). The
[selected profile](install-from-source.md#start-the-station) supplies the arms,
model, hardware mapping and limits. Use a separate work directory for each
running station. See [serve options](../station/src/commands/mod.rs) or run
`uv run cargo run -p hebbian-robot-use-server -- serve --help` for flags
and defaults.

The station supervises the controller and the media companion. A media
restart interrupts snapshots and live publishing while control and state
remain available. The viewer's media status endpoint reports service and camera
errors. After a controller failure, use explicit recovery before resuming control.

Support and disable the arm before restarting the station to apply code,
profile or environment changes. A graceful shutdown can attempt to return to
the recorded rest pose before releasing torque. See
[failure handling](control-invariants.md#failure-responses).

### HTTP and event access

Use the [OpenAPI contract](../schemas/openapi.json) for routes and payloads.
The [station route table](../station/src/openapi.rs) owns those descriptions.

- To retry a submission after a lost reply, reuse the operation ID and the
  original request body. A different body with that ID conflicts. An expired
  outcome returns HTTP 410 on retry. Inspect or cancel a running request by
  operation ID. See [request tracking](../server-core/src/station_api/submissions.rs).
- Send stream goal updates with increasing sequence numbers for the same
  operation ID. The stream keeps the speed set when it started.
- Resume the event stream with the last event ID. HTTP 409 means the cursor
  is no longer available. Open a fresh stream to get a snapshot. See
  [event replay](../server-core/src/station_api/events.rs).
- Request [observations](../station/src/openapi.rs) to read arm state and camera
  snapshots together. Required cameras must return an image. The returned arm
  revisions let you reject motion based on a stale observation.

### SDK recovery behavior

The SDKs resume telemetry with the event cursor and request a fresh snapshot
when replay is unavailable. Inspect the command outcome and reacquire control
before sending new motion after a lost lease. A cancellation over a failed
connection leaves the physical outcome uncertain.

Python and TypeScript can retry submissions with the same operation ID. Rust
returns a connection error for the application to handle. See the
[Python](../hebbian_rua/client/station_connection.py),
[TypeScript](../typescript-client/src/station-connection.ts), and
[Rust](../sdk/src/connection.rs) clients for retry and cancellation behavior.

## Control, tools and results

See the [control model](control-model.md) for leases, command outcomes, action
sequences and [explicit compilation](control-model.md#compile-before-running).
[Write your own loop](write-your-own-loop.md) covers direct commands and tool
selection. [Model output](model-output.md) describes the data shown to models.

## Pose preview — no motor commands

Preview checks one or more independent Cartesian targets against the current
arm configuration. Results stay in input order. See the [tool catalog](../schemas/robot-tools.json)
for model-facing inputs and the [planner](../server-core/src/planning/mod.rs) for
IK and path checks.

With no orientation supplied, preview lets IK choose one. A move preserves the
current orientation. To compare a specific pose, supply the same explicit
orientation in both calls. Quaternion components are ordered `[x, y, z, w]`.

A constrained preview reports a feasible result and how far it falls short.
Motion follows a checked joint-space path. Check physical clearance separately.
See [commands and outcomes](write-your-own-loop.md#commands-and-outcomes).

## Cameras

[Camera setup](cameras.md) covers USB, phone and LiveKit feeds. Local USB and
simulator snapshots can serve SDK clients without LiveKit. See
[simulator integration](simulators.md) for other engines.

## Guard process and stop behaviour

An independent guard watches the powered controller. Following controller
loss it can supervise a hold and release torque if the hold becomes unsafe
or times out. It depends on the host, power and serial link.

A software stop brakes and holds the arm. Disabling releases torque, and an
unsupported arm can fall. A physical power cutoff operates separately from the
controller. See [failure responses](control-invariants.md#failure-responses)
for protective stops and recovery.
