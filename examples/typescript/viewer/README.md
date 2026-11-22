# Example viewer

The station serves this React viewer at `/viewer/`. Follow the
[viewer build instructions](../../../docs/install-from-source.md#optional-viewer)
and start a station with [your arm connected](../../../docs/getting-started.md).
Use [MuJoCo](../../../docs/simulation.md) to test the viewer without hardware.
The viewer build uses the TypeScript SDK included in the checkout. The SDK
requires no separate build or installation from npm.

## Use the viewer

- **Workspace** shows the robot and cameras. Choose **Manual** or **Agent**
  while the arm is disabled. Enable the arm, then adjust a target in **Manual**
  or enter a task in **Agent**. See [arm controls](../../../docs/so101-manual-control.md#controls).
- **Run options** under **Agent** configures the model and cameras for that task.
- **Add remote camera** pairs a phone. See [Cameras](../../../docs/cameras.md)
  for phone and USB camera setup.
- **Recording** starts or stops capture and configures automatic recording.
  **Recordings** opens saved sessions. See [Recording](../../../docs/recording.md).
- **Settings** controls appearance, pose overlays, camera labels, and the agent
  server address.

## Develop the viewer

From the repository root, run `pnpm dev` and open the Vite viewer URL. Start the
station first and add `--manual-control` to exercise manual motion in simulation.
Use `pnpm check` and `pnpm test:viewer` to check changes.

- [`src/app/`](src/app/) contains the page entry points and workspace.
- [`src/features/`](src/features/) contains the controls, cameras, scene,
  recordings, and settings.
- [`server/src/`](server/src/) hosts the [reference agent](../agent/README.md).
- The [TypeScript SDK](../../../typescript-client/README.md) owns station access
  and shared schemas. See the [station reference](../../../docs/reference.md#station-server)
  to build a different viewer.
