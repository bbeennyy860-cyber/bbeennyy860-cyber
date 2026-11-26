# AEOLUS development plan for Helios

**26 November 2026 · First release: Habitat V2 fault replay**

Build an offline, side-by-side viewer for two runs of AEOLUS’s actual Python simulator. Start with **the same habitat and operating conditions, with and without fan degradation**. A visitor should see where airflow changed, scrub back to when it changed, and trace each value to a recorded sample. Package the result for the team repository, then use the same build for Ben’s GitHub profile GIF and download.

## What the project is trying to show

AEOLUS studies decisions under incomplete observations. An operator sees sensor readings and equipment feedback. The simulator can also reveal what it knows internally, which may differ from those readings. The viewer should let someone ask whether the same command produced the same airflow in both runs, and then inspect the evidence. That makes the geometry useful. A later view can ask whether a bounded intervention actually improved the outcome.

This is the first part of the team’s [research interface and replay work package](https://github.com/arm-hackathon/arm-hackathon/blob/main/docs/plans/aeolus-development-programme.md#wp22--build-the-research-interface-and-replay-experience). Helios implements the viewer. Ben reviews the demo story and public wording. Coordinate the team-repo review with Yaroslav, the roadmap’s proposed integration lead, and ask the simulation reviewer to check the scenario pair.

## Starting point verified for this handoff

I ran AEOLUS 0.8.0 from [team `main` at `c74c155`](https://github.com/arm-hackathon/arm-hackathon/tree/c74c15592d60f36282f33a103ae0dbe7540a7a3a) with the locked Python 3.11 environment.

| What ran | Observed result |
| --- | --- |
| Habitat V2 actuator-feedback example | Four steps, five validated trace rows including setup. Eight zones. At 120 s, the recorded total flow was 0.294830 m³/s and lab branch feedback was 0.031232 m³/s. This example combines several faults and changes operating mode. |
| Habitat V2 forecast-development scenario through the plant CLI | 24 steps, 25 validated rows at 60 s intervals. Eight zones, one occupied-mode segment and an empty fault list. Candidate base for the matched pair. |
| Existing forecast report and independent verifier | Four distinct candidate predictions, 24 replayed HMC steps and `verified: true` on the supported demo. This is a separate forecast/replay path. |
| Existing browser fixture | Three Node tests passed. The current profile GIF uses this approximate JavaScript model and displays its airflow in arbitrary units. |

The two existing Python scenarios differ in their schedules and fault lists. Use the results above as smoke checks. A causal comparison needs the matched pair below.

To repeat the smoke checks from the repository root, choose a fresh `out/` path for each invocation of the plant CLI.

```bash
uv sync --locked --python 3.11 --extra dev
uv run --locked --python 3.11 --extra dev python -m aeolus.habitat_v2 \
  scenarios/habitat_v2_actuator_feedback.json out/replay-baseline/actuator-feedback.jsonl
uv run --locked --python 3.11 --extra dev python -m aeolus.habitat_v2 \
  scenarios/habitat_v2_forecast_development.json out/replay-baseline/forecast-development.jsonl
uv run --locked --python 3.11 --extra dev python scripts/run_habitat_v2_forecast_report.py
node --test demo/browser-simulator/simulator.test.mjs
```

## 1. Produce one trustworthy pair

1. **Refresh the base.** Clone the team repo, read `AGENTS.md` and `CONTRIBUTING.md`, record the current `main` SHA, and work on a focused branch such as `helios/habitat-v2-replay`. Check whether the team has already started WP22.
2. **Use one public scenario.** Use `scenarios/habitat_v2_forecast_development.json` as the nominal input. Create a clearly named replay-only variant from it with exactly one `fan_speed_degradation` profile. Start with the mechanism and multipliers in `scenarios/habitat_v2_actuator_feedback.json` and choose an explicit active range with a few samples before onset. Keep the 24 steps, 60 s cadence, topology, initial state, sensors, loads, and command timeline identical. The scenario inputs have no user-facing seed knob. Confirm the pair meets the repo’s development-fixture rules with the team before making it a checked-in example.
3. **Run the Python engine.** Reuse `load_scenario_file` or `Scenario.from_mapping`, `run_scenario`, and `validate_trace_bytes`. Have a small entry point such as `scripts/build_habitat_v2_replay.py` write both input snapshots, both canonical JSONL traces, a manifest and `index.html` into a fresh `out/` directory. Reject mismatched clocks, topology and unsupported schema versions. Keep the existing browser fixture as its own offline example.
4. **Define the proof before drawing it.** The nominal and fault runs should match on selected physical and observed fields before the first active fault row. The command schedule should match throughout. During degradation, show a measured difference in effective fan speed and branch or total airflow at one named step. Compare aligned trace rows by `step` and `time_s`. Locate fault onset from `fault_receipt.active_faults` in the emitted data and check the initial row’s null receipts. Use **faulted minus nominal** consistently for deltas. Derive the values, units and active interval from the run. A viewer has no command authority and runs no model inference.

**First checkpoint:** deliver the two scenario inputs, validated traces and a short table covering one pre-fault row and one fault-active row. Re-run both on the same commit, environment and platform in a second fresh directory and compare the JSONL hashes. If the pair fails to isolate an airflow change, verify the scenario and physics before building the view.

## 2. Build the comparison viewer

**Default view.** Use two aligned plan views labelled **Nominal** and **Fan degradation**, with one shared time scrubber, play/pause, step buttons and a single metric legend. Open on the first verified fault-active sample and give the reader an obvious way to step back. Highlight the lab branch and show its two airflow values and signed difference beside a paired time-series chart. Choosing another zone should update both views and the chart together.

**Geometry.** Draw zone footprints from `zones[].geometry.center_m` and `size_m`, ducts from `air_network.branches[].duct_polyline_m`, and the fan from `air_network.fan.position_m`. Use the same extent, projection and colour domain for both views. Use line weight or colour to encode the recorded branch flow. A clean top-down view is enough for the release. Try a fixed isometric view later if elevation clarifies the routing. Avoid invented room shapes, decorative machinery or particle motion that readers could mistake for resolved fluid dynamics.

| Display | Source in the V5 trace |
| --- | --- |
| Branch airflow reported by the simulated instrumentation | `operational_feedback.branch_airflow_m3_s[zone]` |
| Modelled branch flow and total flow | `air_network_receipt.zone_flow_m3_s[zone]` and `total_flow_m3_s` |
| Reported zone temperature and CO₂ | `telemetry[zone].temperature_k` and `co2_ppm`. Convert Kelvin to °C once at the display boundary. |
| Fan command and actuator response | `actuator_receipt.fan.requested_fraction`, `achieved_fraction` and `effective_fraction` |
| Fault timing | `fault_receipt.active_faults`, labelled **Simulator diagnostic** |

Show observation and simulator diagnostic data in distinct, plainly labelled views. Render unavailable values as an em dash with an explanation. Preserve real zero readings. Use precise numbers in the inspector without making the view unreadable. Stack the panels on narrow screens, keep the clock and zone selection shared, and make all controls keyboard operable. Provide a text readout for the selected zone, use a visible focus style, and start playback only when requested.

**Packaging.** Follow the existing forecast report’s offline HTML embedding pattern. Keep readable renderer sources in the repository and generate a single self-contained HTML file. The manifest should identify the generator commit, input and trace schemas, scenario and run IDs, SHA-256 hashes of the original input and trace bytes, and the common time axis. Keep timestamps and local paths out of the deterministic payload. Put the provenance and trace references behind a short **Run details** panel. [Existing report renderer](https://github.com/arm-hackathon/arm-hackathon/blob/main/src/aeolus/habitat_v2/forecast/live_demo_report.py)

## 3. Verify and put it on GitHub

- **Data checks.** Check displayed values and deltas against raw JSONL at initial, pre-fault and fault-active rows. Cover null receipts, genuine zeros, incompatible pairs and changed input/trace bytes. Re-run deterministic pair generation and validate the manifest. Keep scenario and trace contracts intact.
- **Browser checks.** Open the generated file directly from disk in Chromium and Firefox with networking disabled. Verify both panels follow one timeline through play, pause, seek and step. Check desktop and narrow layouts, keyboard controls and zone selection. Record the browser versions and exact sampled values in the PR.
- **Repo checks.** Run the locked pytest, Ruff, compile and lock checks from `AGENTS.md`, `git diff --check`, and `node --test demo/browser-simulator/simulator.test.mjs`. Re-run the existing forecast report wrapper if a shared report helper changes. Record any failure and distinguish it from the new replay tests. [Contributor guide](https://github.com/arm-hackathon/arm-hackathon/blob/main/AGENTS.md)
- **Team repo first.** Prepare one reviewable PR with the scenario pair, generator, viewer source, focused tests and a README with the exact generation command. Follow the team’s review and publishing rules. Use a public release asset for the single-file HTML and, if useful, a trace bundle. Generated local `out/` files stay out of Git. Confirm the link downloads a usable HTML file from a signed-out browser.
- **Profile second.** Record a short GIF from the exact release build. Show the shared time, units, both values and the delta at a stable fault-active frame. Hold that frame long enough to read. Link the preview to the public demo or release and put the offline download and reproduction links below it. Update the current caption and alt text with numbers from the new trace, credit the team, and describe Ben’s contribution using the work he actually did. Check the live profile at desktop and mobile widths. [Profile README](https://github.com/bbeennyy860-cyber/bbeennyy860-cyber/blob/main/README.md)

**Ready to ship when** a new visitor can download and open the replay offline, identify the time and zone where flow changed, and reproduce the displayed comparison from the documented source revision and inputs. Helios should give Ben one review package with the runnable HTML, GIF preview, trace manifest, test results and proposed profile wording. Ben can focus his review on one screen, one measured claim and one working download.

## The next engineering improvement

Once the viewer is reliable, add one question at a time. A sensor-fault replay should show the reported signal diverging from simulated truth. Then use the existing HMC-governed forecast/replay path to design a paired **hold versus permitted intervention** example on a matched public development condition. Record proposal, arbitration, final command, achieved response and outcome separately. Use a declared outcome and a conventional baseline before testing learned advice. The first release gets its technical value from a reproducible comparison and a clear explanation of the fault. Training or Spark work can follow the project’s research roadmap. [Research roadmap](https://github.com/arm-hackathon/arm-hackathon/blob/main/ROADMAP.md)
