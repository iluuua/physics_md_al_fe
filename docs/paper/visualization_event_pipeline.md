# Visualization Event Pipeline

Status: prepared for existing Stage B run artifacts. The pipeline is safe by
default: detection/window planning writes small artifacts, while event-window
reruns, OVITO rendering, and ffmpeg encoding require separate operator action.

## Goal

Provide one reproducible pipeline for paper and presentation assets:

1. detect dislocation nucleation or deformation-only evidence;
2. write an event timeline;
3. choose an event window for a high-frequency rerun plan;
4. render OVITO frames and static figures with fixed presets;
5. assemble a 30 FPS MP4 through a reproducible ffmpeg command;
6. preserve traceability from every frame/figure/video back to case, timestep,
   dump, restart, and analysis output.

## Current Integration

The implementation lives in `analysis/python/event_pipeline/` and uses CLI
wrappers in `scripts/`:

- `scripts/build_event_timeline.py`
- `scripts/plan_event_window.py`
- `scripts/render_geometry_panels.py`
- `scripts/render_deformation_frames.py`
- `scripts/render_dxa_frames.py`
- `scripts/render_event_frames.py`
- `scripts/export_event_figures.py`
- `scripts/encode_animation_30fps.py`
- `scripts/run_event_pipeline_dry_run.py`

The default Stage B configuration template is
`configs/event_pipeline.stageB.template.yaml`. Thresholds are also kept as code
defaults in `event_pipeline.schema.EventThresholds` so dry-runs work without
loading a config file.

The focused 5-6 transition helper lives in
`scripts/prepare_focus_5_6_transition.py`, with safety logic in
`analysis/python/stage_runner/focus_transition.py`. It writes preservation,
safe-stop, focused-run, and preflight artifacts only; it does not launch MD,
OVITO, or ffmpeg.

## Event Classes

`confirmed_DXA` means DXA found a line signal:

- `dislocation_segments >= 1`, or
- `dislocation_line_length_A > 0`, or
- `dislocation_density_per_m2 > 0`.

`weak_hcp` means structural or plastic-zone evidence exists without confirmed
DXA. The default thresholds include HCP atoms, HCP percentage, OTHER atoms, or
plastic-zone detection.

`deformation_only` means displacement, atomic strain, or Dmin2 passes the
configured deformation thresholds, but there is no DXA or weak HCP evidence.

`no_event` means none of the configured thresholds crossed.

## Timeline

Build the event timeline from existing production `analysis.json` files:

```powershell
.venv\Scripts\python.exe scripts\build_event_timeline.py --run-root runs\stageB_realism_100k\20260613-222836
```

For an intentionally incomplete run, use `--allow-incomplete`; missing running
cases are skipped:

```powershell
.venv\Scripts\python.exe scripts\build_event_timeline.py --run-root runs\stageB_realism_100k\20260613-222836 --allow-incomplete
```

Outputs under `<run_root>/event_pipeline/`:

- `event_timeline.csv`
- `event_timeline.json`
- `event_detection_report.md`

The timeline schema includes the required manifest fields:

- `frame_id`, `case_id`, `timestep`, `time_ps`, `dump_file`, `restart_file`
- `camera_id`, `coloring_mode`, `visible_layers`
- thermodynamic fields: `temperature`, `pressure`, `pe`, `ke`, `etotal`,
  `pxx`, `pyy`, `pzz`
- science fields: `eps_z`, `dislocation_segments`,
  `dislocation_line_length_A`, `hcp_atoms`, `other_atoms`,
  `atomic_strain_p95`, `atomic_strain_p99`, `Dmin2_p95`, `Dmin2_p99`,
  `max_displacement`, `event_class`

Extra timeline fields record `stage`, `phase`, `analysis_file`, `event_score`,
and `event_reasons`.

## Event Window

Plan the high-frequency rerun window:

```powershell
.venv\Scripts\python.exe scripts\plan_event_window.py --run-root runs\stageB_realism_100k\20260613-222836
```

If `confirmed_DXA` exists, the first confirmed timestep is selected. The
nearest restart before the window start is used when available.

If no `confirmed_DXA` exists, the fallback branch selects the strongest
`weak_hcp` or `deformation_only` frame and writes a rerun plan without launching
anything.

Outputs:

- `event_window_plan.json`
- `event_window_plan.md`
- `event_window_rerun.template.yaml`

The generated command is a template only. A high-frequency rerun remains
operator-triggered and requires separate approval.

## Rendering

All render entry points write manifests by default and do not invoke OVITO
unless `--execute` is supplied.

```powershell
.venv\Scripts\python.exe scripts\render_geometry_panels.py --run-root runs\stageB_realism_100k\20260613-222836
.venv\Scripts\python.exe scripts\render_deformation_frames.py --run-root runs\stageB_realism_100k\20260613-222836
.venv\Scripts\python.exe scripts\render_dxa_frames.py --run-root runs\stageB_realism_100k\20260613-222836
.venv\Scripts\python.exe scripts\render_event_frames.py --run-root runs\stageB_realism_100k\20260613-222836
.venv\Scripts\python.exe scripts\export_event_figures.py --run-root runs\stageB_realism_100k\20260613-222836
```

Preset strategy:

- geometry: overview context, particle type coloring;
- deformation: atomic strain or Dmin2 coloring, fixed camera;
- DXA: atoms plus DXA lines, fixed camera;
- event: structure type plus DXA overlay;
- figures: before/at/after when enough frames exist, otherwise the available
  traceable frame(s).

Each manifest records `camera_id`, `coloring_mode`, `visible_layers`, output
PNG path, source dump, timestep, and analysis path.

## Video

Create the 30 FPS ffmpeg plan after event frame manifests exist:

```powershell
.venv\Scripts\python.exe scripts\encode_animation_30fps.py --run-root runs\stageB_realism_100k\20260613-222836
```

Default behavior writes:

- `videos/event_animation_30fps.manifest.json`
- `videos/event_animation_30fps.ffmpeg.txt`

It uses `-framerate 30`, H.264, `yuv420p`, and `-n` by default, so important
artifacts are not overwritten. Use `--execute` only after frames exist and the
operator approves encoding.

## One-Step Dry-Run

For a safe end-to-end preparation pass:

```powershell
.venv\Scripts\python.exe scripts\run_event_pipeline_dry_run.py --run-root runs\stageB_realism_100k\20260613-222836
```

This writes timeline, event-window plan, render manifests, figure manifest, and
video command manifest. It does not run MD, OVITO, or ffmpeg.

For the completed-case partial handoff, the output directory is:

```powershell
.venv\Scripts\python.exe scripts\run_event_pipeline_dry_run.py --run-root runs\stageB_realism_100k\20260613-222836 --output-dir runs\stageB_realism_100k\20260613-222836\event_pipeline_partial_completed --allow-incomplete
```

The transition helper also writes alias files expected by the handoff:

- `event_timeline_completed_cases.csv`
- `event_timeline_completed_cases.json`
- `completed_cases_detection_report.md`

## Focused 5-6 Transition

The focused run template is
`configs/stageB_nearGB_vacancies_focus_100k.template.yaml`. It contains exactly
two production cases:

- `B3_nearGB_vacancies_medium_eps0025`
- `B3_nearGB_vacancies_medium_eps0100`

The template keeps Stage B realism geometry and uses:

- `production_chunk_steps: 10000`
- `dump_every.production: 1000`
- `restart_every: 10000`
- `thermo_every.production: 1000`
- `dump_modify sort id` through the existing runner input rewrite path
- event-pipeline metadata for timeline, event window, frame manifests, render
  presets, and 30 FPS video planning.

Prepare the safe transition artifacts without launching MD:

```powershell
.venv\Scripts\python.exe scripts\prepare_focus_5_6_transition.py --old-run-root runs\stageB_realism_100k\20260613-222836 --focus-run-root runs\stageB_nearGB_vacancies_focus_100k\20260615-215533 --partial-event-output-dir runs\stageB_realism_100k\20260613-222836\event_pipeline_partial_completed
```

Current focused run root:

- `runs\stageB_nearGB_vacancies_focus_100k\20260615-215533`
- effective config: `effective_config.yaml`
- command file: `focus_run_command.txt`
- preflight: `focus_run_preflight.json` and `focus_run_preflight.md`

The preflight blocks launch when `lmp_kokkos_cuda.exe` is active, the GPU is not
free, the completed-case manifest is missing, the focused config is not exactly
the two target cases, or the old and new run roots match.

## Branch A: confirmed_DXA Found

1. Review `event_timeline.json` for the first `confirmed_DXA` frame.
2. Review `event_window_plan.md` and generated restart choice.
3. If higher cadence is needed, approve and adapt
   `event_window_rerun.template.yaml`.
4. After the high-frequency rerun completes, rebuild the timeline against the
   rerun output.
5. Render DXA/event frames with `--execute` only after OVITO is available.
6. Encode the 30 FPS video only after frames are verified.

## Branch B: No confirmed_DXA

1. Treat the highest-scoring `weak_hcp` or `deformation_only` frame as the
   fallback focus.
2. Use the event-window plan as a deformation-localization rerun template, not
   as proof of dislocation nucleation.
3. Render deformation and geometry panels first.
4. Record the absence of confirmed DXA as a result, linked to the exact
   analysis files and dumps.
5. Keep 500k confirmation blocked unless the Stage B post-run gate later
   produces `A_500k_confirmation`.

## Current Stage B Observation

The partial completed-case dry-run on
`runs/stageB_realism_100k/20260613-222836` found three completed production
analysis frames as of 2026-06-15 21:54:

- `B3_interior_vacancies_medium_eps0025`
- `B3_nearGB_perfect_eps0025`
- `B3_nearGB_perfect_eps0100`
- timestep `100000`
- event class `weak_hcp` for all three frames
- `confirmed_DXA` not found in the completed frames
- fallback event window currently selects `B3_nearGB_perfect_eps0025`,
  `90000..110000`
- restart source `restart.B3_nearGB_perfect_eps0025_production.90000`

Operator update on 2026-06-15 22:16:

- old queue LAMMPS PID `2024` was stopped with normal `Stop-Process`;
- no force stop was used;
- old runner PIDs `21884` and `2812` were already absent after the LAMMPS stop;
- latest old restart remained
  `restart.B3_interior_vacancies_medium_eps0100_production.80000`;
- latest captured old log step was `88000`;
- old-root LAMMPS/runner process count after stop was `0`.

The focused nearGB vacancies run was then launched from:

`runs\stageB_nearGB_vacancies_focus_100k\20260615-215533`

Initial focused runtime state:

- focused parent runner PID: `11716`
- focused child runner PID: `15260`
- focused LAMMPS PID: `19080`
- current case at initial check: `B3_nearGB_vacancies_medium_eps0025_prep`
- exactly one LAMMPS was active, and no old-root process was active.

The initial focused event-pipeline dry-run with `--allow-incomplete` returned
`external_execution: not_run`, `frame_count: 0`, and `blocked_no_frames`. This
is expected until focused production analysis frames exist.

Follow-up validation on 2026-06-16 00:27:

- no old-root runner/LAMMPS process was active;
- no focused runner/LAMMPS process was active;
- focused prep/smoke phases completed for both target cases;
- the first focused production chunk failed before writing a restart;
- focused stage status: `failed_production`;
- failed chunk:
  `B3_nearGB_vacancies_medium_eps0025_production.chunk0000000_0010000`;
- production log file was missing.

Failure evidence from the production stdout:

```text
ERROR on proc 0: Cannot open universe log file log.B3_nearGB_vacancies_medium_eps0025_production.chunk0000000_0010000.lammps: No such file or directory
```

The refreshed focused event-pipeline dry-run still returned
`external_execution: not_run`, `frame_count: 0`, and `blocked_no_frames`. This
is a runtime/path failure before production analysis, not a no-event scientific
result.

## Stage C 1M Queue Plan

Operator update on 2026-06-16 17:31:

- prepared Stage C root:
  `runs\stageC_1M_nearGB_vacancies_eps0100_100k\20260616-173123`;
- selected case:
  `C1_1M_nearGB_vacancies_medium_eps0100`;
- target class: requested `1,000,000` atoms, configured geometry target
  `950,000`, estimated `944,812` atoms;
- expected runtime for the first `100000` steps:
  optimistic `~4 days`, expected `~5 days`, pessimistic `~6.5 days`;
- production dump cadence: `5000` steps plus final dump;
- restart cadence: `10000` steps;
- estimated total storage for dumps/restarts/overhead: `3.98 GB`;
- preflight result: `queue_ready: true`,
  `blocked_by: active_focused_100k_lammps`;
- Stage C was not launched.

The launch-after-focus guard is:

```powershell
.venv\Scripts\python.exe scripts\launch_stageC_1M_after_focus.py --focus-run-root runs\stageB_nearGB_vacancies_focus_100k\20260615-215533 --stageC-run-root runs\stageC_1M_nearGB_vacancies_eps0100_100k\20260616-173123
```

The guard re-runs preflight and refuses launch if any `lmp_kokkos_cuda.exe`,
focused runner, OVITO, or ffmpeg process is active.

The queued root was dry-run through the event pipeline with
`--allow-incomplete`. The result was `external_execution: not_run`,
`frame_count: 0`, `event_window: blocked_no_frames`, and
`video: blocked_no_frames`, which is expected before Stage C writes production
analysis frames.

After the 100k checkpoint:

- if `confirmed_DXA` is present, prepare an event-window high-frequency rerun;
- if no DXA but a strong precursor exists, review continuation from
  `restart.100000` to 200k/250k using the command templates in the Stage C root;
- if no DXA and no precursor exists, stop scaling and prepare
  positive-control/seeded/cyclic branches.

## Assumptions And Limits

- Existing final `analysis.json` files are currently frame-level inputs. A real
  nucleation onset needs higher-cadence dumps from an approved event-window
  rerun.
- OVITO is imported only in execute mode. Dry-runs and tests do not require
  scriptable OVITO.
- ffmpeg is not invoked unless `--execute` is passed.
- Thresholds are conservative defaults and should be reviewed before final
  publication figures.
