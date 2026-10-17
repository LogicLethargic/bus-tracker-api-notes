# Bus tracking API (reverse-engineered)

> Host, API path, tracking token and the sample payload below have been
> redacted. See [README.md](README.md#redactions).

The operator publishes a tracking page at `/track/<token>`. It's static and
renders nothing on its own — it polls a JSON endpoint from inline JS and paints
the result. Reverse-engineering it means reading that JS and working out what it
calls.

## Endpoint

```
GET https://<host>/api/method/<...>.tracking.status?token=<token>
Accept: application/json
```

- Frappe-style: the body is `{"message": {...}}`. Knowing the framework gets
  you the response shape for free.
- The page fetches a *relative* URL on purpose, so it works behind a masking
  proxy — that's the tell that the path isn't meant to be hit directly.
- **CORS**: the server reflects any `Origin` and sends
  `Access-Control-Allow-Credentials: true`, so browser-side embeds from other
  origins work with no proxy.
- Rate limiting and errors return HTTP 403 with a JSON body (and no
  `message.state`, so a missing state has to be treated as an error, not a
  valid response).
- Poll cadence comes back as `poll_seconds` in the payload — 30 while live, but
  back off to 60 in the `early` state.

## Payload shape

```jsonc
{
  "state": "live",              // live | early | cancelled | expired | (else = invalid link)
  "coach_number": "<redacted>",
  "bus": {"lat": 0.0, "lng": 0.0},
  "fix_age_minutes": 1,
  "fix_stale": false,           // true = GPS hasn't reported lately
  "boarding": {"name": "<redacted>", "time_hhmm": "<redacted>",
               "lat": 0.0, "lng": 0.0},
  "trip": {"origin": "<redacted>", "destination": "<redacted>"},
  "boarded": true,              // passenger on the bus vs waiting at the stop
  "eta_minutes": null,          // ETA to YOUR stop; null once boarded
  "next_stop": {"name": "<redacted>", "eta_minutes": 7},
  "your_stop": {"name": "<redacted>", "reached": true, "here": false},
  "drop": {"name": "<redacted>", "time_hhmm": "<redacted>",
           "reached": false, "here": false, "eta_minutes": 240},
  "stops": {
    "boarding": [{name, time_hhmm, point_type, yours, drop, reached, here,
                  next, done, lat, lng}, …],
    "dropping": [{…same, drop:true for your drop…}, …]
  },
  "poll_seconds": 30
}
```

- `stops.boarding` and `stops.dropping` are each in route order. Concatenating
  them gives the full route.
- Flag semantics: `yours` = your boarding stop, `drop` = your drop stop,
  `next` = next stop ahead, `here` = the bus is currently at it,
  `reached`/`done` = passed.
- Data quirks worth knowing: duplicate GPS pins exist in the source data (the
  same stop name appears twice, once as a real stop and once as a `BYPASS`
  variant, tens of metres apart). Dedupe by distance, not by name.
- The `early` state adds `activates_at_hhmm` and `starts_in_seconds` — a
  countdown to when tracking goes live.

## State handling

`live` and `early` render the trip, `cancelled` → "Booking cancelled",
`expired` → "Journey completed", anything else → "Link not found".

Failures use exponential backoff, `min(30 * 2^(n-1), 300)` seconds, and keep
showing the last good data on screen. Only a server-side "invalid" verdict
clears the display — a 403 is not the same as a dead link.

## Progress-bar math

Implemented in the bus tracker module of the extension (`bustracker.js`).

1. `buildRoute(status)` — concatenate `stops.boarding` and `stops.dropping`,
   drop anything without coordinates, collapse consecutive points closer than
   30 m, then build cumulative haversine distances → `{pts, cum, total}`.

2. `progressAlong(route, busPos)` — project the bus onto every segment using a
   local equirectangular approximation (`kx = cos(midLat) * 111320`,
   `ky = 110540`), take the nearest segment, and return
   `(cum[i-1] + t * segLen) / total`. Monotone in practice; clamp `t` to
   `[0, 1]`.

3. `stopFraction(route, stop)` — `cum[firstIndexByName] / total`, which gives
   the tick marks on the bar.

Validated against a live run: 36 points, 319.3 km total, bus reading 30.3%
along the route. The bus sat 4.9 km off the stop polyline — expected, since
stops are GPS-pinned and the road isn't, and a great-circle line between two
stops will not match the road. Self-projection error came out at 0.000%, so
the maths itself is sound; the data is just approximate.

### Rendering

`St.DrawingArea` + Cairo: rounded track at `text` colour with 0.12 alpha, accent
fill, vertical tick marks (your stop in accent, your drop muted), and a bus dot
made of an accent core plus a ring in the text colour. The whole thing dims when
`fix_stale` is set.

Progress eases toward its target (`shown += (target - shown) * 0.22` at 100 Hz)
so the dot slides between polls instead of jumping.

### Deliberately not built

"Halt in the middle" — showing a stop on the bar and prompting when to buzz the
driver — was specced and then cut. Worth revisiting if the widget ever gets an
ETA that's reliable enough to trust for a time-critical prompt.
