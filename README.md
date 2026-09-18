# FOC Simulator

A Simulink simulation of Field Oriented Control (FOC) for a surface-mount permanent magnet synchronous motor (PMSM).

> **Status: Work in progress / not working yet.** I'm actively debugging this simulation — the control loop doesn't currently drive the motor correctly. This repo is a snapshot of where I'm at.

## Why I'm building this

I'm building this as prep for a future power electronics/motor control implementation on the **UBC Aero** design team. FOC is the standard technique for controlling brushless motors (e.g. in ESCs), and this project is me working through the theory and control structure in simulation before trying to implement it on real hardware.

## What is Field Oriented Control?

A brushless PMSM is driven by three sinusoidal phase currents (`Iu`, `Iv`, `Iw`), and the torque it produces depends on how those currents are aligned with the rotor's magnetic field. Controlling three interacting, constantly-rotating sine waves directly is hard.

FOC's trick is to change coordinate systems so the problem becomes simple:

1. **Measure** the three phase currents and the rotor angle (θ) from an encoder.
2. **Clarke transform**: convert the 3-phase currents (`Iu, Iv, Iw`) into a 2-axis stationary frame (`Iα, Iβ`).
3. **Park transform**: use the rotor angle θ to rotate that stationary frame into a frame that *spins with the rotor* — the **d-q frame** (`Id`, `Iq`).

In the d-q frame, the rotating sine waves become **DC quantities**:
- **Iq** (quadrature axis) — directly proportional to torque.
- **Id** (direct axis) — the flux-producing component, normally regulated to 0 for a surface-mount PMSM.

Because Id and Iq are just DC values, they can be controlled with simple **PI controllers**, exactly like a DC motor. The controller outputs (`Vd`, `Vq`) are then transformed back through the **inverse Park** and **inverse Clarke** transforms into three-phase voltage commands, which are converted into switching signals for the inverter (via PWM) that actually drives the motor.

<p align="center">
  <img src="Screenshot 2026-09-17 210320.png" alt="Rotor d-q reference frame vs stator alpha-beta frame" width="600">
</p>

*The stationary stator frame (α, β) vs. the rotating rotor frame (d, q). The Park transform is what converts between them using the rotor angle θ.*

## The control loop

At a high level, the full FOC loop looks like this — torque command in, PI current control, PWM generation, inverter, motor, and current/position feedback closing the loop:

<p align="center">
  <img src="Screenshot 2026-09-17 210003.png" alt="High-level FOC block diagram" width="700">
</p>

Expanded to show the actual Park/Clarke and inverse Park/Clarke transform stages on both the forward (voltage command) and feedback (current measurement) paths:

<p align="center">
  <img src="Screenshot 2026-09-17 210011.png" alt="Detailed FOC block diagram with Park/Clarke transforms" width="700">
</p>

## My implementation

This is built in Simulink, using a Surface Mount PMSM plant model. The structure follows the block diagrams above:

- Two PID controllers regulate `Id` (target 0) and `Iq` (target 5, as a torque reference) in the rotating d-q frame.
- Their outputs go through an **inverse Park transform** (d,q → α,β) and an **inverse Clarke transform** (α,β → a,b,c) to produce the three-phase voltage command fed to the `Surface Mount PMSM` block.
- Measured phase currents are fed back through a **Clarke transform** (a,b,c → α,β) and a **Park transform** (α,β → d,q, using `sin(θ)`/`cos(θ)` derived from the motor's electrical position) to close the current loop.

<p align="center">
  <img src="Screenshot 2026-09-17 205409.png" alt="Simulink implementation of the FOC control loop" width="900">
</p>

## Current state / debugging notes

This does **not work correctly yet**. I'm actively debugging the control loop — if you're looking at this repo, assume it's a work in progress rather than a working reference implementation.
