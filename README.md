# Pit Wall Strategy Console

The **Pit Wall Strategy Console** is a planned decision-support tool for **Apex Velocity Racing's pit-wall strategy team** at the **Aurora Grand Prix**. It aims to give strategists a clear, shared view of race-strategy information for a single car, helping them evaluate options and discuss decisions such as pit-stop timing and tyre choices.

The console is intended to support the people making strategy calls, **not to make decisions automatically**. This repository currently describes the proposed product and its boundaries; it does not yet contain a working application.

## Who It Is For

The primary users are the race strategists on the Apex Velocity Racing pit wall. They need a concise view of the information relevant to their car so they can make and communicate strategy decisions during the Aurora Grand Prix.

## Scope

This project has four explicit boundaries:

- **One car only:** The console focuses on a single Apex Velocity Racing car, not the full racing field or a multi-car team view.
- **Synthetic data only:** All example inputs are fabricated for development, demonstration, or testing. No real race data is required.
- **No live telemetry:** The console will not connect to live car sensors, timing feeds, or other real-time telemetry services.
- **No race simulation:** The project will not run a race simulation or model lap-by-lap race outcomes. This is a small change

## Repository Layout

The intended directory structure is:

- `docs/` — Product notes, requirements, and design documentation.
- `src/` — Application source code when implementation begins.
- `tests/` — Tests and test fixtures when implementation begins.

## Project Status

**Planning / initial repository setup.** There is **no application code yet**. This README defines the intended product, audience, and scope. The functionality described here is a goal, not an implemented feature.

## Delegation and human-owned controls

### Setup activities I may delegate

I may delegate routine setup work to Claude Code, including creating
project folders and starter files, preparing configuration and
documentation changes, and running tests, formatting, and setup checks.
Claude Code may prepare these changes for human review.

### Controls that remain human-owned

Only a human may:
- Create, change, or disable the ruleset protecting `main`.
- Decide who can bypass repository rules or change bypass permissions.
- Change pull-request approval requirements or decide who may approve.
- Provide the required human approval and authorize merging into `main`.

Claude Code must not weaken or bypass these controls to complete a task.

### My responsibility

I remain responsible for delegated work. Before accepting changes, I will
review the changed files, check the test and setup results, and confirm
that the work matches the intended task. Changes must follow the normal
pull-request process and the protections configured for `main`.