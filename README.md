# ◈ Campus Twin AI

**Dual-Twin Counterfactual Engine · Zero IoT Sensors · Local-Only**

A campus energy, water, and e-waste sustainability platform where **every claimed saving is verified against a parallel "ghost twin" simulation** — not estimated, not asserted, not read off a spreadsheet. If the live twin applies an approved action and the ghost twin doesn't, the gap between them *is* the measured saving.

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [How it works](#how-it-works)
- [Features](#features)
- [Screenshots](#screenshots)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Architecture](#architecture)
- [Data sources](#data-sources)
- [Models](#models)
- [Project structure](#project-structure)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Limitations & honest disclosures](#limitations--honest-disclosures)
- [Roadmap](#roadmap)
- [License](#license)

---

## Why this exists

Most campus sustainability dashboards report savings like this:

> "We turned the AC off for 3 hours, so we saved 4.5 kWh."

That number is a **claim**, not a measurement. It's computed from nameplate wattages and occupancy guesses. There is no way for a facilities manager, a sustainability officer, or a judge to verify that the kWh was actually avoided.

Retrofitting a campus with IoT smart meters fixes this — at ₹15,000–₹40,000 per meter, plus gateway hardware, network provisioning, and months of deployment. Most institutions never do it.

**Campus Twin AI** demonstrates that counterfactual verification is achievable **without any new hardware**, using historical smart-meter data (BDG2) as the baseline and a webcam for one room's ground truth. The verification *mechanism* is the same one you'd use on real meters — swapping in live hardware requires **zero changes to the rest of the codebase**.

---

## How it works

The system runs two simulations in lockstep:
