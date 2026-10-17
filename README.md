# bus-tracker-api-notes

Working notes from reverse-engineering an undocumented bus-tracking API, plus the
route-progress math that came out of it. Written while building a bus tracker
widget for a GNOME desktop extension.

The widget itself lives in a separate repo — this is the "how did we figure it
out" write-up.

## Why this exists

A bus operator publishes a tracking page at a `/track/<token>` URL. The page
itself is static and does nothing useful on its own: it polls a JSON endpoint
from inline JS and renders the result. Anyone wanting that data in their own UI
has to read the page's JavaScript and work out what it's calling.

That's what this is — the endpoint, the payload, the states, and the geometry
needed to turn a stream of GPS fixes into a meaningful "how far along is the
bus" number.

## What's in here

- [bus-tracker-api.md](bus-tracker-api.md) — endpoint shape, payload, state
  machine, and the progress-bar math

## Redactions

A working version of this document contained two things that shouldn't be
public, so they've been removed:

- **A live tracking token.** Anyone holding it can pull that specific
  passenger's journey.
- **The real hostname, API path, and a full sample payload.** The sample was a
  real person's trip — coach number, coordinates, stop names, times.

Everything technical is intact. The endpoint's structure still shows it's a
Frappe framework method call, the payload shape and state machine are complete,
and the geometry is unchanged.

## The short version

```js
// 1. stops -> one polyline, with cumulative distance
pts = [...boarding, ...dropping].filter(hasCoords)
// drop points closer than 30 m, then accumulate haversine

// 2. project the bus onto that polyline, return fraction along it
for each segment:  t = clamp(dot(bus - a, b - a) / |b - a|^2, 0, 1)
pick nearest segment ->  (cum[i-1] + t * segLen) / total

// 3. stop markers are just their own fraction along the same polyline
ticks = stops.map(s => cum[indexOf(s)] / total)
```

Projecting onto a local equirectangular approximation
(`kx = cos(midLat) * 111320`, `ky = 110540`) is accurate enough over a few
hundred kilometres and avoids the cost of a proper geodesic.

## Tooling

Written up as plain markdown rather than a test suite — the code it describes
lives in the extension repo.
