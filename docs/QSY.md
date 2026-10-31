# QSY across the mesh (`AUTO_QSY`, `QSY_ENABLE`, `QSY_FOLLOW`)

A **QSY** moves the nodes on one talk group to another talk group. SvxReflector
has always supported this *locally* (to the reflector's own clients). GeuReflector
now also **propagates a QSY across trunk, twin and satellite links**, so meshed
reflectors move their own local nodes on the same TG along with it — and every
knob is config driven, with a global default and a per-TG override.

Two kinds of QSY exist and they are kept separate:

- **Automatic QSY** — the reflector moves an *idle* TG to a free random TG after
  `AUTO_QSY_AFTER` seconds of silence (needs `RANDOM_QSY_RANGE`).
- **Manual / DTMF-92 QSY** — a node operator requests a QSY (the node sends the
  request; the stock SvxLink node handles the "press PTT to follow" prompt via
  its own `QSY_PENDING_TIMEOUT`). The reflector only relays the request.

> The "follow the QSY by keying up within X seconds" behaviour lives in the
> **node** (stock SvxLink `QSY_PENDING_TIMEOUT`), not in the reflector. A group
> already talking (PTT held) follows immediately; an idle node follows only if
> it keys up within its own pending window. The reflector just delivers the
> `MsgRequestQsy`; this is unchanged and works the same on every meshed peer.

## Settings

All keys live in `[GLOBAL]` (default for every TG) and/or in a per-TG section
`[TG#<n>]` (overrides the global value for that TG).

| Key | Values | Default | Meaning |
|-----|--------|---------|---------|
| `AUTO_QSY` | `1` \| `0` | *legacy* | Enable automatic idle QSY. When **unset**, the legacy rule applies: auto QSY is active whenever `AUTO_QSY_AFTER > 0`. Set it explicitly to force on/off independent of the timer. |
| `AUTO_QSY_AFTER` | seconds | `0` | Idle seconds before an automatic QSY fires. `0` disables it. Requires `RANDOM_QSY_RANGE` in `[GLOBAL]`. |
| `QSY_ENABLE` | `1` \| `0` | `1` | Allow a client/DTMF-92 requested QSY on this TG. `0` makes the reflector ignore QSY requests from nodes on this TG. |
| `QSY_FOLLOW` | `1` \| `0` | `1` | Honour a QSY **received from a mesh peer**. `0` means an incoming trunk/twin/satellite QSY for this TG is ignored and local nodes stay put. |

### Examples

Global defaults, one TG opted out of following mesh QSYs:

```ini
[GLOBAL]
RANDOM_QSY_RANGE=12399:100
AUTO_QSY_AFTER=300      # auto QSY after 5 min idle (auto-enabled by the timer)

[TG#91]
QSY_FOLLOW=0            # TG 91 never follows a QSY pushed from a peer
```

Force auto QSY off on a busy TG while keeping the global timer:

```ini
[GLOBAL]
AUTO_QSY_AFTER=300

[TG#240]
AUTO_QSY=0             # no automatic QSY on 240, regardless of the timer
```

## How mesh propagation works

```
Reflector A: QSY fires on from_tg -> to_tg (auto, or client/DTMF-92)
  1. A sends MsgRequestQsy(to_tg) to its own local clients on from_tg   (as before)
  2. A sends MsgPeerRequestQsy(from_tg, to_tg) to each peer that carries from_tg

Reflector B (peer) receives MsgPeerRequestQsy:
  - if QSY_FOLLOW is enabled for from_tg:
        B sends MsgRequestQsy(to_tg) to B's own local clients on from_tg
  - B never re-propagates it onto the mesh
```

- **Loop-safe.** A received peer QSY is applied to local clients only and is
  never forwarded on. In a cyclic trunk topology it therefore cannot loop; each
  reflector gets its own copy directly from the originator (single application).
- **TG mapping / routing respected.** On a trunk the QSY obeys the same
  blacklist / allow / shared / cluster / interest / `TG_MAP` rules as audio and
  talker-start, and both talk group numbers are mapped through the link's
  `TG_MAP` just like audio. Satellites apply their TG filter.
- **Backward compatible on the wire.** The propagation uses a new peer message
  type (`MsgPeerRequestQsy`, type 130). An **old (half-only) reflector** on the
  same mesh simply logs "Unknown message type" and ignores it — so old and new
  software interoperate with no change to existing behaviour. The local QSY and
  all existing messages are byte-for-byte unchanged.

## Interaction with full-duplex

In a full-duplex conference (see [FULL_DUPLEX.md](FULL_DUPLEX.md)) the nodes that
"joined by PTT" are all on the same TG, so a QSY now moves the whole conference —
local nodes and nodes on meshed peers alike — to the new TG, with each node's own
PTT-follow rule deciding whether it comes along.
