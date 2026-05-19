# Geospatial Library Design

## Purpose

The geospatial library provides shared spatial primitives and deterministic geometry operations for the Weather Assistant system.

It exists to keep MCP tools, weather data adapters, and future UI workflows aligned on a common representation of locations, areas, routes, and spatial relationships.

The library should answer questions such as:

* Is this point inside this polygon?
* How far is this storm feature from the user?
* What weather data should be sampled around this location?
* Does this projected weather feature intersect a user-defined area?
* What bounding box should be requested for a map or data query?

The library should not decide what the weather means conversationally. It provides spatial evidence for the agent and tool layer.

---

## Design Principles

### 0. Prefer TypeScript-Native Libraries

The Weather Assistant should stay in TypeScript when practical.

The geospatial library should be implemented as a TypeScript package backed by established JavaScript/TypeScript geospatial libraries rather than custom geometry algorithms.

Recommended Phase 1 backing libraries:

* Turf.js for GeoJSON geometry operations, spatial relationships, distance, buffering, simplification, and bounding boxes
* Proj4js later if coordinate transformation requirements grow beyond WGS84 and simple local measurements

The project-owned library should wrap these dependencies behind Weather Assistant domain types so MCP tools do not depend directly on third-party geometry APIs.

### 1. Canonical Geometry Comes First

All MCP tools should accept and return geometry using a small set of canonical shapes.

Initial canonical geometry types:

* point
* bounding box
* polygon
* line string
* route

These shapes should be enough for Phase 1 use cases without committing the system to a full GIS domain model too early.

### 2. WGS84 Is the External Contract

External APIs, MCP tools, persisted user locations, and agent-facing responses should use WGS84 latitude and longitude.

Rules:

* latitude and longitude are decimal degrees
* latitude range is `-90` to `90`
* longitude range is `-180` to `180`
* coordinate order is always explicit in schemas
* API field names should prefer `latitude` and `longitude` over positional arrays

Internal operations may project coordinates when needed for distance, buffering, interpolation, or area calculations.

Turf.js uses GeoJSON coordinate arrays in `[longitude, latitude]` order. The Weather Assistant API should still expose explicit `latitude` and `longitude` fields to reduce agent and caller mistakes. Conversion to and from GeoJSON should happen inside the geospatial library.

### 3. Geometry Operations Should Be Deterministic

Given the same inputs, the library should return the same geometry and measurement results.

The library should avoid hidden weather-specific interpretation. For example, it may return that a point is inside a polygon, but it should not decide what that means for a weather workflow.

### 4. Spatial Results Should Be Minimal and Reproducible

Geospatial operations should return minimal deterministic results by default.

The caller already knows which function it called and which geometry it passed in, so the library should avoid echoing inputs or adding service-specific provenance.

Results should include extra metadata only when it affects interpretation or reproducibility.

Examples:

* units for numeric measurements
* tolerance used for simplification
* whether geometry was simplified
* whether the result is approximate
* whether the result was clipped to requested bounds

The geospatial library should not include weather-service identifiers such as alert IDs, MRMS product IDs, or forecast provider references. Weather-specific provenance belongs in the calling MCP tool or data adapter.

---

## Scope

Phase 1 should include:

* geometry validation
* coordinate normalization
* bounding box creation
* point-in-polygon checks
* point-to-point distance
* point-to-polygon distance
* polygon intersection checks
* simple buffering around points
* geometry simplification for API and UI payloads
* common unit conversion

Phase 1 should support:

* NWS alert polygon checks
* MRMS point and area sampling
* forecast lookup by point
* nearby precipitation searches
* map viewport queries

---

## Library Boundary

The geospatial package should expose Weather Assistant types and functions, not raw Turf.js calls.

Example package responsibilities:

* convert canonical Weather Assistant geometry to GeoJSON
* convert GeoJSON source data into canonical Weather Assistant geometry
* validate coordinates before calling GIS operations
* normalize third-party library errors into project error codes
* attach approximation metadata when an operation is lossy or bounded

MCP tools should import the project geospatial package. They should not import Turf.js directly.

This boundary keeps the system free to replace or supplement Turf.js later without changing every weather tool contract.

---

## Non-Goals

The initial library should not include:

* meteorological hazard classification
* severe-weather detection
* storm-cell identity tracking
* probabilistic impact modeling
* long-range storm extrapolation
* user-facing safety recommendations
* rendering or map display logic

---

## Canonical Geometry Types

Canonical Weather Assistant geometry should be explicit and agent-friendly.

Internally, the geospatial package may convert these types to GeoJSON for Turf.js operations.

### Point

Represents a single location.

Required fields:

* `type`: `point`
* `latitude`: decimal degrees
* `longitude`: decimal degrees

Optional fields:

* `label`
* `source`
* `accuracy_meters`

Example:

```json
{
  "type": "point",
  "latitude": 41.8781,
  "longitude": -87.6298,
  "label": "Chicago, IL"
}
```

### Bounding Box

Represents an axis-aligned WGS84 envelope.

Required fields:

* `type`: `bbox`
* `south`
* `west`
* `north`
* `east`

Example:

```json
{
  "type": "bbox",
  "south": 41.7,
  "west": -87.9,
  "north": 42.0,
  "east": -87.5
}
```

### Polygon

Represents a closed area.

Required fields:

* `type`: `polygon`
* `rings`

Rules:

* the first ring is the exterior boundary
* additional rings are holes
* rings contain points in WGS84 coordinates
* rings must be closed or normalized to closed form

Example:

```json
{
  "type": "polygon",
  "rings": [
    [
      { "latitude": 41.7, "longitude": -87.9 },
      { "latitude": 42.0, "longitude": -87.9 },
      { "latitude": 42.0, "longitude": -87.5 },
      { "latitude": 41.7, "longitude": -87.5 },
      { "latitude": 41.7, "longitude": -87.9 }
    ]
  ]
}
```

### Line String

Represents an ordered path.

Required fields:

* `type`: `line_string`
* `points`

Example:

```json
{
  "type": "line_string",
  "points": [
    { "latitude": 41.8781, "longitude": -87.6298 },
    { "latitude": 41.8818, "longitude": -87.6231 }
  ]
}
```

### Route

Represents a user-relevant path with optional timing context.

Required fields:

* `type`: `route`
* `path`

Optional fields:

* `departure_time`
* `arrival_time`
* `mode`

The route type should be treated as a higher-level wrapper around a line string. Phase 1 can store route shape and metadata without implementing full route/weather intersection analysis.

---

## Measurement Units

Internal calculations should use metric units unless a source product requires otherwise.

Preferred API units:

* distance: meters
* speed: meters per second
* area: square meters
* time: ISO 8601 timestamps and durations
* angles: degrees

Agent-facing tools may include converted display units when useful, but the canonical measurement should remain explicit.

---

## Core Operations

### Validate Geometry

Validates shape, coordinates, ring closure, and supported geometry type.

Returns:

* normalized geometry when valid
* structured validation errors when invalid

Implementation note:

* validation should happen before converting to GeoJSON
* GeoJSON conversion should use `[longitude, latitude]` coordinate order internally

### Create Bounding Box

Creates a bounding box from:

* point plus radius
* polygon
* line string
* route
* collection of geometries

This operation supports area queries, map viewport requests, and polygon pre-filtering.

Bounding boxes should also be accepted as input parameters for operations that need constrained spatial work.

Examples:

* clipping a polygon to a map viewport
* limiting an intersection check to a requested area
* requesting a weather data window around a user location

### Check Point In Polygon

Determines whether a point is inside, outside, or on the boundary of a polygon.

Returns:

* relation: `inside`, `outside`, or `boundary`
* distance to boundary when practical

Primary use:

* determining whether a point is inside a supplied polygon

Likely backing operation:

* Turf.js boolean point-in-polygon behavior, wrapped with boundary handling and metadata

### Measure Distance

Measures distance between:

* point and point
* point and polygon
* point and line string

Returns:

* distance in meters
* nearest point when applicable

Likely backing operations:

* Turf.js distance and nearest-point operations for Phase 1
* later projection-aware calculations if Phase 1 accuracy is insufficient for local buffering or area operations

### Buffer Point

Creates an approximate polygon around a point for a requested radius.

Primary use:

* localized MRMS sampling
* nearby precipitation searches
* small-area user context

Likely backing operation:

* Turf.js buffer operation, with explicit units and approximate-result metadata

### Intersect Geometries

Determines whether two geometries overlap.

Phase 1 should support:

* polygon and polygon
* bbox and polygon
* bbox and point

Returns:

* relation
* intersection bbox or geometry when practical
* approximate flag when simplification is involved

Likely backing operations:

* Turf.js boolean intersection checks for simple relationship queries
* Turf.js intersection operations when an intersection geometry is needed

### Simplify Geometry

Reduces geometry complexity for API responses or UI display while preserving broad spatial meaning.

Returns:

* simplified geometry
* original point count
* simplified point count
* tolerance used

Simplification should never be used silently for safety-sensitive containment checks unless the response clearly marks the result as approximate.

Likely backing operation:

* Turf.js simplify operation, with tolerance recorded in the response

---

## Error Model

The library should return structured errors rather than ambiguous booleans.

Common errors:

* `invalid_geometry`
* `unsupported_geometry_type`
* `coordinate_out_of_range`
* `empty_geometry`
* `invalid_polygon_ring`
* `projection_failed`
* `operation_not_supported`
* `precision_limit_exceeded`

Errors should include:

* code
* message
* field path when applicable
* whether the caller can retry with adjusted input

Errors should not include service-specific identifiers. Callers can attach their own source context when wrapping geospatial errors in MCP/API responses.

---

## MCP/API Usage

MCP tools should use this library for all spatial validation and relationship checks.

Examples:

* `get_alerts_for_point` should validate the point and use point-in-polygon checks against alert geometry.
* `sample_mrms_at_point` should validate the point and derive the relevant MRMS grid lookup.
* `find_nearby_precipitation` should create a point buffer or bbox before querying MRMS products.
* `get_forecast_summary` should validate the point before calling forecast providers.

MCP tools should not each implement separate geometry parsing, coordinate validation, or containment logic.

---

## Open Questions

Areas that need later design:

* How to represent antimeridian-crossing geometries
* Which projection strategy to use for local distance and buffering
* Whether the frontend should share this library directly
* How to model moving storm features over time
* How to represent route timing and exposure
* How much geometry simplification should happen server-side
* Whether spatial indexes are needed in Phase 1

---

## Recommended Phase 1 Decision

Start with a small TypeScript geospatial library that owns validation, normalization, distance, containment, bounding boxes, buffering, simplification, and conversion to/from GeoJSON.

Use explicit WGS84 point objects in MCP/API contracts. Avoid positional coordinate arrays in agent-facing schemas.

Use Turf.js as the Phase 1 GIS engine behind the project-owned geospatial package.

Defer storm objects, temporal geometries, route-weather intersections, and advanced spatial indexing until after MRMS and NWS alert workflows are designed.
