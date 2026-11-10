Degree level: Bachelor’s

# Acoustic Drone Localization Agent

**Author:** Ori Cohen
**Course:** 00560413, AI Fluency and Agentic Methods for Engineers (Winter 2026–27)

## Problem

Small drones are hard to detect with radar and cameras: they are small and slow, and they fly low and often out of line of sight. Their rotors, though, make a strong harmonic acoustic signature. The goal is a passive system that estimates a drone's **direction (azimuth/elevation), range, and velocity** from microphone-array recordings.

Getting this to work in real conditions is hard. Low SNR, wind, multipath reflections, changing rotor speed during maneuvers, and multiple drones all break simple pipelines. Today, designing a pipeline that stays robust across these conditions is manual, trial-and-error work.

## Approach (Approach C)

### Ground (deterministic, no LLM)

The ground is a Python acoustic simulator built on `pyroomacoustics` and NumPy, seeded for reproducibility.

- **Scenario generator:** drone trajectory, rotor harmonic signature (fundamental + harmonics, variable RPM), Doppler, propagation loss, background noise, wind noise, reflections, optional second drone.
- **Array model:** configurable microphone geometry, with one or more arrays.
- **Evaluator:** runs a candidate pipeline on a fixed scenario suite and returns:
  - direction error (deg), range error (m), velocity error (m/s)
  - detection rate and false-alarm rate
  - per-scenario breakdown, so failures can be located

Unit tests validate the simulator against analytic cases, such as a static source in free field and pure Doppler shift.

**Ground qualification:**
- Importable Python API (`simulate(scenario, pipeline) -> metrics`)
- Free / open source (`pyroomacoustics`, NumPy, SciPy)
- Deterministic (fixed seeds)
- Bounded and fast: a full scenario suite runs in seconds to a minute
- Testable with `pytest`
- Non-trivial: the failure regimes need diagnosis, not just parameter sweeps

### Wiring

- The simulator and evaluator are exposed to the agent through a custom **MCP server** with tools such as `run_suite`, `run_scenario`, `get_failures` and `plot_scenario`.
- A custom skill holds the pipeline building blocks and the conventions for editing them.
- Unit tests on the ground and on the tools run in CI (no secrets, no live agent).

### Agent loop

1. The agent proposes or edits the estimation pipeline. This can mean the DOA method (GCC-PHAT, SRP-PHAT, MUSIC), harmonic tracking, range estimation (multi-array triangulation, level-based), or tracking (Kalman / particle filter). Array geometry may also be in scope.
2. The ground runs the scenario suite.
3. The agent reads the per-scenario results, identifies failure regimes (e.g. "DOA collapses below 0 dB SNR with reflections"), and forms a hypothesis about the cause.
4. The agent makes a structural change and the loop repeats.

## Failure test

There are two baselines:

- **B1, fixed pipeline:** GCC-PHAT + triangulation + Kalman, with parameters tuned by grid search / Bayesian optimization.
- **B2, random structural search:** random selection among the same pipeline building blocks.

The agent must beat both on the held-out scenario suite, or match them with significantly fewer simulator runs.

## Success metrics

- Median and 90th-percentile direction / range / velocity error on a held-out scenario set
- Robustness: performance degradation from easy to hard scenarios
- Efficiency: number of simulator calls to reach the final result

## Scope

**In scope:** simulated environment, a single drone (stretch goal: two), outdoor scenes.

**Out of scope:** drone type classification, real-time embedded deployment.

**Stretch goal:** validation on a real recording made with a low-cost mic array (ESP32 + MEMS microphones).

## Known risks

- **Range from a single array is weakly observable.** Mitigation: two arrays and triangulation.
- **Doppler-based velocity is confounded by rotor RPM changes.** Mitigation: velocity from the track (direction + range over time).
- **Sim-to-real gap.** Mitigation: the stretch-goal validation recording.
