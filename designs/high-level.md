# Weather Assistant — High Level System Design

## Goals

Build a conversational weather assistant focused on:

* Hyperlocal weather awareness
* Near-term storm investigation
* Conversational meteorological reasoning
* Spatially-aware weather analysis
* Clear communication of evolving conditions

The system is **not** intended to:

* Replace operational meteorologists
* Issue official warnings
* Perform authoritative severe-weather detection
* Produce highly accurate long-range forecasts

The assistant should behave like:

> “A strong weather-aware investigative assistant that can reason about storms conversationally using high-quality atmospheric data.”

---

# Core Product Philosophy

## Primary Use Cases

Examples:

* “Will this storm hit me?”
* “How bad is it likely to get here?”
* “Am I in the warned area?”
* “Is the strongest part likely to miss north or south?”
* “When should the rain arrive?”
* “Should I delay biking home?”
* “What changed in the last 15 minutes?”
* “Can you explain what I’m seeing on radar?”

---

# System Architecture

```text
User
  ↓
Conversational Agent
  ↓
Planning / Reasoning Loop
  ↓
Meteorological Tool APIs
  ↓
Weather Data Sources
```

---

# Core Architectural Principles

## 1. Tools Provide Evidence, Not Conclusions

Tools should primarily expose:

* observations
* measurements
* timestamps
* spatial relationships
* derived physical quantities

Tools should avoid embedding:

* meteorological judgment
* conversational reasoning
* confidence interpretation
* safety recommendations
* synthesized conclusions

The agent is responsible for:

* interpreting evidence
* weighing conflicting information
* determining uncertainty
* communicating conclusions

---

## 2. MRMS Is the Primary Atmospheric State Source

MRMS should serve as the default near-term weather awareness layer.

Use MRMS for:

* precipitation intensity
* storm motion
* broad storm structure
* derived severe-weather products
* nearby storm analysis
* short-term weather evolution

The assistant should generally consult MRMS first for hyperlocal storm questions.

---

## 3. NEXRAD Level III Is Investigative Detail

Level III radar data is supplemental.

Use Level III when:

* additional radar detail is needed
* radar-specific investigation is useful
* velocity analysis is required
* MRMS appears ambiguous
* the user explicitly asks about radar detail

The system should avoid overusing station-specific radar data for general weather awareness.

---

## 4. NWS Alerts Provide Official Hazard Context

NWS alerts are:

* authoritative
* operationally important
* official hazard guidance

However:

* alerts are not perfectly hyperlocal
* the assistant may refine local impact estimates
* the assistant may discuss likely local storm tracks inside a larger warning polygon

Principle:

> NWS alerts establish official risk context. Radar and MRMS refine the local impact estimate.

---

## 5. Forecasting and Nowcasting Are Separate Concerns

### Near-Term (0–2 hours)

Primarily:

* MRMS
* NWS alerts
* targeted radar investigation

### Medium-Term (2–12 hours)

Primarily:

* forecast context
* broader atmospheric trends
* active weather analysis when relevant

### Longer-Term (12+ hours)

Primarily:

* forecast APIs
* lower-precision planning guidance
* broader summaries

The assistant should avoid pretending to provide storm-cell-scale predictions beyond reasonable nowcasting horizons.

---

# Data Sources

## Primary Sources

### MRMS

Purpose:

* primary near-term atmospheric state source
* storm structure and trends
* derived weather products

### NWS Alerts API

Purpose:

* official warnings/watches/advisories
* hazard context
* polygon geometry

### NEXRAD Level III

Purpose:

* radar investigation
* reflectivity and velocity detail
* radar-perspective analysis

### Forecast APIs (e.g. Open-Meteo)

Purpose:

* broader forecast context
* planning-oriented weather guidance
* non-hyperlocal future conditions

---

# Geospatial System

The assistant should be fundamentally spatial.

Expected capabilities include:

* point-based analysis
* polygon interaction
* map-driven workflows
* relationship analysis between weather features and user-defined regions

However, the geospatial architecture is intentionally not yet finalized.

Important unresolved areas include:

* canonical geometry representation
* coordinate systems and projections
* frontend/backend geometry responsibilities
* temporal geometry modeling
* moving storm object representation
* route modeling
* spatial indexing/query strategy
* extrapolation mechanics
* geometry simplification strategies

A dedicated geospatial architecture and interaction design phase is expected later in development.

---

# Agent Reasoning Model

## Planning Loop

High-level flow:

```text
1. User asks question
2. Agent identifies question type
3. Agent determines required evidence
4. Agent gathers evidence from tools
5. Agent evaluates whether additional information is needed
6. Agent synthesizes a response
```

The planning loop should remain iterative and adaptive rather than rigidly procedural.

---

# Question Classification

Example categories:

| Category    | Example                      |
| ----------- | ---------------------------- |
| Threat      | “Am I in danger?”            |
| Track       | “Will this hit me?”          |
| Timing      | “When will it arrive?”       |
| Severity    | “How bad will this get?”     |
| Planning    | “Should I bike home now?”    |
| Explanation | “What am I seeing on radar?” |
| Forecast    | “Will it rain tomorrow?”     |

---

# Reasoning and Uncertainty

The assistant should reason explicitly about:

* evolving weather conditions
* conflicting evidence
* rapidly changing storm structure
* extrapolation limits
* spatial ambiguity
* stale or incomplete data

Uncertainty communication should primarily emerge from the agent’s reasoning process rather than being embedded directly into backend tools.

---

# Tool Design Philosophy

The system should generally prefer:

* composable evidence-oriented APIs
* flexible planning behavior
* observable intermediate reasoning
* minimal hidden meteorological logic

The architecture should avoid prematurely embedding:

* hardcoded storm conclusions
* deterministic severe-weather classification
* opaque heuristic scoring systems

The goal is to preserve a clean separation between:

* evidence extraction
* meteorological reasoning
* conversational synthesis

---

# Important Non-Goals

The system should avoid:

* replacing operational severe weather systems
* overconfident severe-weather claims
* deterministic long-range storm extrapolation

---

# Recommended Development Phases

## Phase 1 — Core Conversational Weather Assistant

Implement:

* MRMS integration
* NWS alerts
* forecast integration
* geospatial foundations
* conversational planning loop

Avoid:

* excessive hidden meteorological logic
* overly opinionated backend synthesis

---

## Phase 2 — Improved Investigation and Spatial Reasoning

Focus on:

* stronger radar investigation workflows
* better motion and trend analysis
* improved spatial reasoning
* refinement of planning behavior
* improved user interaction patterns

---

# Core Product Identity

The assistant should feel like:

> “A calm, spatially-aware meteorological investigator.”

Not:

* an alarm system
* an authoritative forecaster
* a deterministic predictor
* a replacement for NWS warnings
