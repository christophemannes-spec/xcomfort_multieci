# xComfort Multi-ECI

**Fork of [karloygard/xcomfortd-go](https://github.com/karloygard/xcomfortd-go)**
(v0.74) - Eaton xComfort to MQTT gateway for Home Assistant. All credit for
the original protocol implementation goes to that project; this fork changes
exactly one thing on top of an otherwise unmodified checkout, described
below. See `UPSTREAM.md` for the full origin reference.

Built because a real bug in multi-ECI (multi-floor) setups had an open
report upstream with no maintainer response for an extended period.

## The bug

In a setup with more than one ECI/gateway (e.g. one Eaton CCIA-ETH per
floor), every interface loads the **same** datapoint file via a single
shared `--file` option. MRF-exported datapoint files number their entries
locally (column 1, values roughly 1-60) starting at **1 for every floor's
export** - these numbers are only unique *within one physical ECI*, never
across floors.

`pkg/xc/readers.go` parses that file into a `map[byte]*Datapoint` keyed by
this small number. When one merged, multi-floor file is loaded, later
floors' entries silently overwrite earlier floors' entries wherever the
numbers collide - and since every interface loads the identical merged
file, **every** interface ends up with the identical, already-collided map,
regardless of which physical ECI it's actually connected to.

Net effect: incoming RF packets from devices on the "losing" floor(s)
arrive over a perfectly healthy TCP connection (handshake, serial number,
ongoing heartbeats all succeed normally - there is no connection-level
symptom at all), but `i.datapoints[data[0]]` resolves to the *wrong* device
(whichever floor's entry survived the overwrite for that number), so the
real device's state is never recognised or published. No error is logged
for this specific case - the packet is silently misrouted, not rejected.

`pkg/xc/collision_test.go` in this fork reproduces the exact scenario with
real device data (two real devices sharing datapoint number 1 - one
silently lost behind the other) and confirms the fix.

## The fix

`main.go`'s `--file`/`-f` flag now accepts **multiple** values (one per
`--host`, in the same order). Each interface loads only its own, correctly
scoped file - no collision possible. Passing `--file` exactly once keeps
the original shared-file behaviour unchanged, so single-ECI/single-USB-
stick setups are unaffected.

The App's `datapoints_files` option (a list, new in this fork) maps to
this: give it one filename per entry in `eci_hosts`, in the same order.
The original single-string `datapoints_file` option still works exactly as
before if you only have one ECI/stick, but logs a warning at startup if you
have more than one `eci_hosts` entry, since that combination is exactly
what caused the bug.

## Migrating from the original xcomfortd-go App

1. Export **one datapoint file per ECI** from MRF (not one merged file for
   the whole house). If you already have per-floor exports lying around
   (many people do, from before combining them into one file), those work
   directly - see `example_datapoint_files/` in this repository for a
   worked example matching a real 4-floor house.
2. Copy those files into `/config/` on your Home Assistant instance (same
   place the old single merged file lived).
3. Install this App, configure `eci_hosts` exactly as before, and set
   `datapoints_files` to the list of filenames in the *same order* as
   `eci_hosts` - e.g. if `eci_hosts` is
   `["192.168.1.15", "192.168.1.14", "192.168.1.16", "192.168.1.17"]`,
   set `datapoints_files` to
   `["eci_r_minus1.txt", "eci_r_0.txt", "eci_r_1.txt", "eci_r_2.txt"]`
   (filenames only, no `/config/` prefix - that's added automatically).
4. Leave `datapoints_file` empty.
5. Start the App and confirm in the log that it reports using N per-ECI
   files ("Using N per-ECI datapoint file(s)").
6. Once you've confirmed devices on every floor are updating correctly,
   you can stop and remove the original xcomfortd-go App.

## Building

Requires Go 1.24+. If your build environment can reach `proxy.golang.org`
and `golang.org` directly, a plain `go build .` in `src/` is enough. If it
can't (e.g. a restricted network egress policy), this worked instead:

```sh
export GOPROXY=direct
export GOSUMDB=off
go mod tidy   # regenerates go.sum against the replace directives below
go build .
```

`go.mod` carries `replace` directives sending `golang.org/x/net`,
`golang.org/x/sync` and `golang.org/x/text` to their `github.com/golang/*`
mirrors - only needed when `golang.org` itself (used for the vanity-import
redirect) is unreachable; harmless otherwise.

## Status / disclaimer

This is a personal, unofficial patch, not affiliated with or endorsed by
the upstream author. It changes exactly one thing (per-interface datapoint
file loading) on top of an unmodified v0.74 checkout. Use at your own
risk, same as the original.
