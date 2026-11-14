# mosh — 1.4.0 + remote-triggerable abort fixes

This fork carries the upstream `master` (39 commits past the 1.4.0 release)
plus eight fix/test commits cherry-picked from the unmerged upstream PRs
#1419, #1420, #1423, #1424, #1430 (October 2026). Release 1.4.0 (2022) is
still what every distribution ships, and its server can be killed by a
single packet from any client.

## Why

Field case: two of our sessions died silently when mosh-server aborted on
its own (signal 6/ABRT) after a client network change. The stock 1.4.0
server asserts on non-positive terminal sizes and has several other
remote-triggerable abort paths. Full write-up: see the issues/PRs above and
our incident report (published separately).

## What is fixed

- Framebuffer::resize: non-positive sizes are ignored, oversized sizes are
  capped (4,000,000 cells) — no assert, no unbounded allocation (fixes #1386;
  covers the negative-size → huge-allocation vector too).
- Receive path: remote-triggerable `abort()`s are now dropped packets.
- Terminal emulator never aborts on host output (degrades instead).
- Socket `close()` failure no longer aborts the session.
- Frozen timestamp is monotonic — a clock step cannot abort the session.

Regression tests included: resize-safety, fragment-robustness,
statesync-robustness.

## Verified

- `make check`: 35/35 (PASS 33, XFAIL 2 stock e2e, FAIL 0).
- A/B reproduction against stock 1.4.0: a valid resize is answered; a
  zero-size resize kills the stock server with signal 6, while this build
  stays alive and keeps answering. Harness: `state/mosh-fixes-2026-11-14/`
  in our workspace (scripts + sender binary).

## Build

```
git clone --recurse-submodules https://github.com/Biotoza/mosh
cd mosh && git checkout mosh-1.4.0-fixes
./autogen.sh && ./configure && make -j$(nproc) && make check
sudo install -m 0755 src/frontend/mosh-server /usr/local/bin/mosh-server
```

Wire format is unchanged — compatible with any mosh client. The system
package stays untouched; rollback is `rm /usr/local/bin/mosh-server`.

## Status

Upstream is responsive but slow (no release since 2022; the five fix PRs are
closed unmerged). This fork exists so operators can run the fixes today and
as a ready source for an upstream merge if the maintainers want it.
