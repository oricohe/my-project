Degree level: Bachelor’s

# Project proposal

**Acoustic Drone Localization Agent** · Ori Cohen

![System overview](docs/overview.svg)

## 1. The problem and goal

(Explain the problem you want to solve and why it matters. Describe what you
want to improve, how you will measure improvement, and which limits or
requirements must be respected. A short example can help someone outside your
field understand the problem.)

**The problem and goal in my project:**

Small drones are hard to detect with radar and cameras. They are small and slow, and they fly low and often out of line of sight. Their rotors, though, produce a strong harmonic acoustic signature. The goal is a **single compact passive device** that estimates a drone's **direction (azimuth/elevation), range, and velocity** from sound alone. The device is one small microphone array: 4–8 microphones rigidly mounted together, with a 10–30 cm aperture, at a known height above the ground.

Real conditions break simple processing pipelines. Low signal-to-noise ratio (SNR), wind, reflections from the ground, rotor-speed changes during maneuvers, and a second drone all cause failures. Range is especially hard with a single small array: direction alone says nothing about distance, so range has to come from indirect cues. These cues are the delay between the direct sound and its ground reflection, how frequency (Doppler) and direction change over time, and received level. Today, designing a pipeline that stays robust across all these conditions is manual trial and error.

**What I want to improve:** the accuracy and robustness of the estimation pipeline across many conditions.

**How improvement is measured:**
- Median and 90th-percentile direction error (deg), range error (m), and velocity error (m/s)
- Detection rate and false-alarm rate
- Robustness: how much performance degrades from easy to hard scenarios
- Efficiency: number of simulator runs needed to reach the result

All metrics are scored on a **held-out test set** of scenarios that the agent never sees during development.

**Limits and requirements:**
- One compact array with fixed geometry; no second station
- Passive sensing only
- The pipeline must be causal, using only past and present audio, so it could later run in real time
- Range is expected to be the least accurate output, and targets will be set accordingly

**Example:** a drone flies 150 m away at 40 m height behind a tree line. The device hears it, but wind noise and the ground reflection blur the signal. A good pipeline reports "azimuth 72°, elevation 15°, range ~150 m, moving left at 8 m/s". A bad one reports the wrong direction or a range that is off by a factor of two.

## 2. The simulator

(Describe a program or model that could simulate your system and calculate what
happens under different conditions. Explain what inputs it takes and what
results it produces. State whether it can run on your computer and accept inputs
in a text file, such as JSON or YAML. If you have not chosen a simulator yet,
name an option you want to explore and explain what you need to check.)

**The simulator in my project:**

The simulator is a custom outdoor acoustic propagation simulator in Python (NumPy/SciPy) that I will write. Room simulators such as `pyroomacoustics` assume static sources inside closed rooms, so they don't fit a moving drone outdoors. The simulator is deterministic: fixed random seeds give identical results.

**What it models:**
- Free-field propagation with time-varying fractional delays, which produces Doppler naturally for a moving source
- Spherical spreading loss and atmospheric absorption
- Ground reflection through one image source (two-ray model)
- Rotor harmonic signature (fundamental plus harmonics, with RPM that varies over time)
- Wind and background noise
- An optional second drone

**Inputs (YAML/JSON files):**
- **Scenario file:** trajectory waypoints, speed, rotor RPM profile, noise and wind levels, ground reflection coefficient, random seed
- **Array file:** microphone positions and height
- **Pipeline config file:** which processing blocks to use and their parameters (see Section 3)

**Outputs:**
- Simulated multichannel audio
- The pipeline's estimates over time
- An **evaluator** report comparing the estimates to ground truth: direction, range and velocity errors, detection and false-alarm rates, and a per-scenario breakdown that shows where failures happen

**Running and testing:**
- It runs on my laptop. A full scenario suite takes seconds to about a minute.
- It is exposed to the agent through a custom MCP server, with tools such as `run_suite`, `run_scenario`, `get_failures` and `plot_scenario`.
- `pytest` unit tests check it against analytic cases: the delay and level of a static source, the Doppler shift of a constant-velocity source, and the two-ray interference pattern.

Scenarios are split into a **development suite**, which the agent can run and inspect, and a **held-out test suite**, which is used only for final scoring.

## 3. The agent's role

(Explain what the language-model-based agent will decide or change, and how it
will use the simulator's results to choose its next action. If known, list the
variables it may change and their allowed ranges. Explain what you expect the
language model to contribute: could a simple program that tries different
options and picks the best result achieve the same goal? Include one question
you still need to answer.)

**The agent in my project:**

The agent designs and improves the **estimation pipeline**. The array geometry stays fixed.

**The loop:**
1. Propose or edit the pipeline config.
2. Run the development suite through the simulator.
3. Read the per-scenario failures (for example, "elevation collapses below 0 dB SNR when the ground reflection is strong").
4. Form a hypothesis about the cause.
5. Make a structural change and run again.

**What it may change:**

| Element | Options / range |
|---|---|
| Direction-of-arrival method | GCC-PHAT, SRP-PHAT, MUSIC |
| Frequency band / harmonic selection | 80 Hz – 8 kHz; number of harmonics tracked: 1–10 |
| Frame length | 20–500 ms |
| Range cue | ground-bounce delay, Doppler–bearing motion analysis, received level, or a fusion of these |
| Tracker | none, Kalman, particle filter; process-noise scale 10⁻³–10 |
| Detection threshold | 0–20 dB above noise floor |
| New blocks | the agent may write a new processing block in Python |

A new block is accepted into the library only after it passes unit tests: interface checks plus a known-answer test on a simple scenario. This keeps broken code out of the loop.

**What the language model contributes:** a simple program that tries options and keeps the best could tune *parameters*, but it can't diagnose *why* a pipeline fails or invent a new block. The agent reads failure patterns and links them to physics. For example, "range errors grow when the drone is high, because the direct and reflected paths merge", which leads to the fix "use Doppler–bearing cues there instead". The agent then changes the pipeline's *structure*.

To test this claim honestly, I will compare against two baselines. Both get the same development suite and the same simulator-run budget, and all three are scored only on the held-out test suite:
- **B1:** a fixed pipeline (SRP-PHAT + level-based range + Kalman) with parameters tuned by grid search / Bayesian optimization.
- **B2:** random structural search over the same building blocks.

The agent must beat both baselines, or match them with significantly fewer simulator runs.

**Open question:** which cues can actually give range from a single compact array at realistic SNR, and how accurate can range get? If no cue works well enough, the problem may need to be reframed, for example as "approaching or receding" plus a coarse range class.

## Explain your project to someone outside your field

(Now describe your whole project again, as if you were explaining it to someone
from another field who has no background in your topic. For example, if you
study chemical engineering, imagine explaining it to a physicist; if you study
aeronautical engineering, imagine explaining it to someone in medicine; if you
study data science, imagine explaining it to a materials engineer. In one or two
short paragraphs, connect all three parts: the problem you want to solve, how
the simulator helps you test ideas, and what the agent will decide or change.
Use everyday language, explain any technical terms you need, and avoid
unexplained abbreviations. Do not simply copy your answers above.)

**My project in simple words:**

Small drones are hard to spot, but they are noisy: their propellers buzz at a characteristic pitch. I want to build one small box with a few microphones that can "listen" to a drone and tell where it is, how far away it is, and how fast it is moving. Sound reaches each microphone at a slightly different moment, and that tells you the direction. The pitch rises as the drone approaches and drops as it leaves, like a passing ambulance siren, and that tells you about its motion. Sound bouncing off the ground arrives a moment later than the direct sound, and that echo hints at the distance. In practice, wind, background noise and echoes make the system easy to fool.

Instead of flying real drones for every experiment, I will write a computer program, a simulator, that recreates how drone sound travels through open air to the microphones, including wind, noise and ground echoes. Then an AI assistant (a language model, like a chatbot that can also run programs) acts as a junior engineer. It picks a way to process the sound, runs it in the simulator on many test flights, looks at where it failed, guesses why ("it gets confused when the drone is high and the echo overlaps the direct sound"), and changes the method. Finally, I will check whether this "thinking" assistant actually finds better methods than a program that just tries random combinations, by testing all of them on flights none of them saw during development.
