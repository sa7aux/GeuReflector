# Full-duplex talk groups (`DUPLEX_MODE`)

GeuReflector can run a talk group as a **full-duplex audio conference**
instead of the classic half-duplex, single-talker repeater behaviour.

- **Half-duplex (default, legacy):** one talker per TG at a time. The first
  station to key up "holds" the TG; everyone else's audio is dropped until
  it releases. Audio is relayed as raw Opus, never decoded.
- **Full-duplex (new):** several stations may talk at the same time. Their
  audio is decoded, **mixed**, and re-encoded, so every listener hears the
  sum — like a telephone conference where you press PTT to join the mix.
  Each active talker hears everyone **except themselves** (mix-minus-self,
  so a duplex/app node never hears its own echo).

Full-duplex is entirely **opt-in and config-driven**. With no configuration
the reflector behaves exactly as before.

> **Status:** this feature is new. Validate it on a test talk group with
> real nodes before enabling it widely — see [Caveats](#caveats).

## Quick start

Enable it for a single talk group:

```ini
[TG#240]
DUPLEX_MODE=full
```

Enable it for the whole reflector (every talk group):

```ini
[GLOBAL]
DUPLEX_MODE=full
```

Enable globally but keep one talk group on the legacy behaviour:

```ini
[GLOBAL]
DUPLEX_MODE=full

[TG#91]
DUPLEX_MODE=half
```

## How the mode is resolved

For each talk group the mode is resolved in this order:

1. `DUPLEX_MODE` in the per-TG section `[TG#<n>]`, if set.
2. otherwise `DUPLEX_MODE` in `[GLOBAL]`, if set.
3. otherwise **`half`** (legacy behaviour).

The value is case-insensitive; only `full` enables mixing, anything else
(or unset) means `half`. A per-TG value always overrides the global one.

## Settings reference

All keys may be placed in `[GLOBAL]` (applies to every TG) and/or in a
per-TG section `[TG#<n>]` (overrides the global value for that TG). The
`MIX_*` tuning keys only have an effect when the TG is in `full` mode.

| Key | Values | Default | Meaning |
|-----|--------|---------|---------|
| `DUPLEX_MODE` | `half` \| `full` | `half` | Selects legacy single-talker relay (`half`) or full-duplex mixing (`full`) for the talk group. |
| `MIX_FRAME_MS` | `5`–`60` | `20` | Audio frame length in milliseconds used by the mixer clock and the Opus encoders. |
| `MIX_JITTER_MS` | `0`–`500` | `60` | Per-source jitter-buffer depth in milliseconds. Larger absorbs more network jitter but adds latency; `0` disables prebuffering. |
| `MIX_MAX_TALKERS` | `1`–`32` | `4` | Maximum number of simultaneously mixed talkers that receive a personalised mix-minus-self stream. Beyond this, extra active talkers receive the full mix. |
| `MIX_SOURCE_TIMEOUT_MS` | `1000`–`600000` | `10000` | How long a source may be idle before it is dropped from the mixer. Dropping is harmless — a source re-registers on its next packet, and listeners keep receiving regardless. |

Out-of-range values are clamped to the stated limits.

Example with tuning:

```ini
[TG#240]
DUPLEX_MODE=full
MIX_JITTER_MS=80
MIX_MAX_TALKERS=6
```

## How it works (brief)

For a `full` talk group the reflector builds one conference mixer per TG:

```
each source (local client, trunk peer, satellite, twin):
    Opus packet -> Opus decoder -> per-source jitter buffer (MIX_JITTER_MS)

on a fixed frame clock (MIX_FRAME_MS):
    sum all active sources            -> full mix
    for each active local talker X    -> encode (full mix - X) -> send to X
    for passive listeners             -> encode (full mix)     -> send to all
    for trunk/satellite/twin          -> encode (local sources only)
```

The stream sent out onto the mesh is a **local-only sub-mix** (the sum of
this reflector's own local sources, never any mesh-received audio). Because
it contains no mesh-sourced audio it can never loop back through the mesh,
and a remote peer never hears its own audio.

## Caveats

Read these before enabling full-duplex in production.

- **Half-duplex is unchanged.** TGs left in (or defaulted to) `half` behave
  byte-for-byte as before. Full-duplex adds a parallel path; it does not
  modify the legacy one.
- **No client change is required.** A node sends its audio and plays the
  single Opus stream it receives, exactly as in half-duplex. The mix is just
  that one stream. This works with stock SvxLink nodes.
- **Simplex radios still can't hear while transmitting.** Talking over each
  other and hearing the mix works for **duplex** and **app/network** nodes.
  A simplex radio or hotspot physically cannot receive while it transmits —
  that is an RF limitation, not something the reflector can change. But such
  a node is no longer *blocked* from entering the mix.
- **Mesh is single-hop.** Mixed audio is correct across a direct trunk/
  satellite/twin peer. It is **not** transited across multiple prefix-routed
  hops. The design is loop-safe (it cannot feed back), so the failure mode on
  a multi-hop path is *dropped* audio between hops, never howling.
- **Monitors are not routed in full mode.** Only nodes that have *selected*
  the TG receive the mix; passive monitors/observers do not (yet).
- **No talker indication.** In full mode the reflector deliberately does not
  send `MsgTalkerStart`/`MsgTalkerStop` — several stations talk at once, so a
  single "talker" is meaningless and would make clients flap. Audio flows;
  clients simply do not show a single talker.
- **Latency adds up.** End-to-end delay is the server mixer jitter buffer
  (`MIX_JITTER_MS`) plus the client's own `JITTER_BUFFER_DELAY`. In full mode,
  keep the client's `JITTER_BUFFER_DELAY` low to avoid stacking both.
- **CPU cost.** Half mode is a byte copy. Full mode decodes every source and
  encodes `MIX_MAX_TALKERS + 1` streams per active TG per frame. Budget for
  it on busy reflectors.

## Tuning

- **Choppy / robotic audio:** increase `MIX_JITTER_MS` (e.g. 80–120). This
  trades latency for robustness against network jitter.
- **Too much delay:** decrease `MIX_JITTER_MS`, and lower the *client's*
  `JITTER_BUFFER_DELAY`.
- **More than four simultaneous talkers expected:** raise `MIX_MAX_TALKERS`.
