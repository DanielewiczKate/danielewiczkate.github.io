---
layout: default
title: "FPGA-Accelerated Triangle Optimizer"
permalink: /fpga_triangle_optimizer.html
image: /images/fpga/mona_lisa_triangles.png
excerpt: "A hill-climbing image approximator with its scoring loop offloaded to a SystemVerilog datapath"
---

# FPGA-Accelerated Triangle Optimizer

This project rebuilds a target image out of semi-transparent
triangles. It's a hill-climber: propose a random triangle, rasterize and
alpha-blend it onto the current best canvas, measure whether the
sum-of-squared-error against the target went down, and keep the proposal
only if it did. 

<div class="row">
	<div class="col-6 col-12-medium">
		<span class="image fit"><img src="/images/fpga/mona_lisa_target.png" alt="Target image" /></span>
		<p style="text-align:center">target</p>
	</div>
	<div class="col-6 col-12-medium">
		<span class="image fit"><img src="/images/fpga/mona_lisa_triangles.png" alt="78 committed triangles" /></span>
		<p style="text-align:center">78 committed triangles, 100,000 proposals</p>
	</div>
</div>

This project is actually about is the inner
loop: rasterize a candidate triangle over its bounding box, alpha-blend it
onto the current best, and accumulate the change in squared error against the
target. That loop runs once per proposal and touches every pixel in the
triangle's bbox, This was moved into hardware with a SystemVerilog
datapath that takes a triangle and a stream of pixels and hands back, `delta_SSE`. 
The host keeps the RNG, proposal generation, accept/reject, and the canvas; 
the FPGA never sees any of that.

<span class="image fit"><img src="/images/fpga/architecture.svg" alt="AXI4-Lite host interface into AXILiteWorker, feeding RasterizerDispatch then RasterizerWorker then a lane-sum Rasterizer, delta_SSE returned to the host via SSE_LO/SSE_HI" /></span>

## `AXILiteWorker` — the bus boundary

Two independent AXI interfaces plus a plain handshake into the compute
datapath. AXI4-Lite control plane: started from an open-source
register-slave skeleton, with the register map, reset behavior, and
write-1-pulse semantics reworked for this design. AXI4-Stream sink: written
from scratch against the ARM AMBA AXI-Stream spec — 64-bit beats, each
carrying a `{t_col, b_col}` pixel pair (`t_col` in `tdata[63:32]`, `b_col` in
`[31:0]`), `tlast` marking the final pixel of the bounding box. `TSTRB`/
`TKEEP` are not implemented; every beat is a full pixel pair.

| offset | register | dir | contents |
| --- | --- | --- | --- |
| `0x00` | `CTRL` | W1P | bit 0 → `start`, bit 1 → `rst_render` (both self-clearing pulses) |
| `0x04` | `STATUS` | RO | bit 0 `busy`, bit 1 `done` (sticky, cleared by `start`) |
| `0x08` | `MAX_COORD` | RW | bounding-box max, `vertex_t` layout |
| `0x0C`–`0x14` | `TRI_V0`/`V1`/`V2` | RW | triangle vertices |
| `0x18` | `TRI_COL` | RW | candidate colour |
| `0x1C` / `0x20` | `SSE_LO` / `SSE_HI` | RO | `delta_sse[63:0]`, valid once `STATUS.done` is set |

Packed types (`FPGA/common.sv`) — the register layout matches these field
orders byte-for-byte, so the driver and the RTL agree on packing without a
translation layer:

```
color_t    { logic [7:0] r, g, b, a }
vertex_t   { logic [15:0] x, y }
triangle_t { color_t color; vertex_t [2:0] verts }
```

Current top-level wiring: the datapath is kicked by `rst_render`, not
`start` — `start` is decoded but the datapath exposes no port for it (see
Known gaps).

## `RasterizerDispatch` — the edge-function front end

Converts a decoded triangle + bbox into per-pixel quantities, walking the
bounding box in raster order and emitting one set of edge-function values
per pixel. RTL port of `RasterizeTriangleV2` (the incremental edge-function
rasterizer in the C++ baseline) — same math, in a register file instead of
recomputed per pixel.

Two phases:

- **Precompute, once per triangle.** The per-edge step constants — `A0..A2`
  (each vertex pair's `Δy`) and `B0..B2` (`−Δx`) — are plain combinational
  `assign`s off the vertices. (They were registered in an earlier revision;
  that created a one-cycle stale-read hazard on the row-origin init and got
  reverted.) The edge-function value at the bbox origin, `d0_row..d2_row`, is
  registered one cycle after reset. Those origin values are the *only*
  multiplies anywhere in this block — six of them, once per triangle — and
  since they only happen once per image, they're a candidate for folding
  onto a single time-shared multiplier (still a `TODO` in the source).
- **Per-pixel, incremental.** Every consumed pixel adds `A*` to the running
  edge value; end of row is detected a cycle ahead by `x + 1 >= max_coord.x`,
  and on that boundary `x` wraps to 0 and the accumulators step by `B*`
  instead. One add per edge per pixel, no multiplies, ever, in the steady
  state. `advance = pixel_valid && render_ready` gates every update, so the
  walk only moves on cycles where a pixel actually gets consumed. `y` isn't
  tracked here at all — the stream master owns `max_coord.y` and knows when
  the packet ends.

`NUM_LANES` exists as a parameter for a future mode that processes several
bbox rows in parallel, each with its own edge-function/index offset. It's
pinned to `1` right now — the per-lane plumbing is there, the lane routing
isn't.

## `RasterizerWorker` — coverage, blend, and the error delta

Per pixel, combinational except for the accumulator register:

- **`in_tri`** — the coverage test: the three edge functions `s_d0`/`s_d1`/
  `s_d2` are all `>= 0` or all `<= 0`.
- **`c_col`** — the candidate colour after alpha-blending `tri_col` over
  `b_col` (the previous-best pixel): `(b*(255-a) + tri*a)/255`. The `/255`
  is done with the `(x*32897) >> 23` reciprocal-multiply trick from the C++
  rasterizer, unsigned, since the numerator is always in `[0, 65025]` — no
  sign handling needed.
- **The error delta** — `diff = b_col - c_col`, `sum = 2*t_col - b_col -
  c_col` per channel, and `sse_acc += Σ diff*sum` on every
  `pixel_valid && in_tri`. That product is the multiply-reduced form of
  `(t_col - c_col)² - (t_col - b_col)²` — three multiplies per pixel instead
  of six, because nobody needs the absolute SSE, only how much a proposal
  changes it.

`Rasterizer` instantiates the dispatch block plus one `RasterizerWorker` per
lane and sums the per-lane `sse_acc` into a single `t_sse_acc[63:0]`.
`FPGAAccelerator` is the synthesis top: it wires `AXILiteWorker` to
`Rasterizer` and `t_sse_acc` back into `SSE_LO`/`SSE_HI`. The datapath reset
is `s_axi_aresetn & ~rst_render`, so one `CTRL` bit-1 write both clears the
accumulator and re-primes the dispatch for the next triangle.

## Pipeline

| property | value |
| --- | --- |
| depth | 2 registered stages: stream-ingress capture in `AXILiteWorker`, then the SSE accumulator in `RasterizerWorker` |
| between stages | fully combinational — edge-function step, coverage test, alpha blend, the squared-error reduction, and the lane sum |
| throughput | 1 pixel / cycle / lane, initiation interval 1, stalls only on `render_ready` backpressure |
| setup | 1 cycle after a `CTRL.rst_render` pulse to latch the per-triangle edge origins and raise `render_ready` |
| datapath widths | 8-bit RGBA in, signed 33-bit edge functions, 64-bit signed SSE accumulator |
| not yet there | a result-valid strobe — see below |

## Verification

Every block has a cocotb testbench: 20 test functions, 19 implemented,
driven through `cocotbext-axi`. Interface-facing tests run four ways — no
stalls, idle inserter, backpressure inserter, both.

Three golden models, weakest to strongest:

1. Hand-written Python mirror of the RTL — unit-level checks.
2. C++ edge-function trace dumped to file — checks `RasterizerDispatch`'s
   pixel walk against the exact rasterizer it ports.
3. Real C++ scoring functions via a `ctypes` bridge, for the full
   interface → dispatch → worker path — an independent implementation, not
   a mirror of the RTL's own logic.

That independence caught a real bug: a `RasterizerWorker` unit test, checked
against the Python model, passed with the blend applied to the wrong
operand, because the Python model had the same bug. The C++ path didn't
share it, and the mismatch surfaced as soon as that test ran.

## Status and known gaps

**Never synthesized.** No fmax, no utilization, no power, no
hardware-vs-CPU comparison — everything above is Icarus/Verilator simulation
plus the differential test against C++. RTL gaps, roughly in fix priority:

- **No completion signalling.** `delta_sse` is combinational and always
  live; `busy`/`done` are undriven in `FPGAAccelerator`, so `STATUS.done`
  never actually asserts. Every testbench works around this by counting a
  fixed number of cycles after the last pixel and reading the accumulator
  directly — a magic number that only holds for a 5×5 test triangle on a
  stall-free stream. This needs a real `sse_valid`/`done` strobe threaded
  through dispatch → worker → register file, matched to the pipeline depth.
  Highest priority — everything else here is downstream of it.
- **`start` vs. `rst_render` isn't resolved.** The datapath is actually
  kicked by `rst_render`; `start` is decoded but never consumed. Need to
  pick one contract and wire `STATUS` to match it.
- **The worker's accumulate gate doesn't match dispatch's advance gate.**
  `RasterizerWorker` accumulates on `pixel_valid && in_tri`; dispatch's
  `advance` also requires `render_ready`. They can disagree on the prime
  cycle and double-count the first pixel — current tests don't sequence
  around it in a way that would expose it.
- **Dispatch doesn't stop itself at the bbox.** `y` isn't tracked and
  `pixel_last`/`tlast` isn't consumed by the walk logic, so stream-length
  correctness is entirely the stream master's job — feed it too many beats
  and it walks past the bottom row.
- **Multi-lane is a parameter, not a feature.** `NUM_LANES` exists; the
  per-lane routing behind it doesn't.
- **`PixelIndexer`**, a standalone raster-order coordinate generator with
  its own testbench, isn't wired into the datapath at all yet — the dispatch
  block generates its own coordinates inline instead.

Full register-level docs and the complete known-gaps list:
[repo](https://github.com/DanielewiczKate/FPGA-accelerated-triangle-optimizer).

The C++ golden model has two measured optimizations of its own: delta-SSE
scoring over a full rescan (1.66x at 100k proposals), and this same
incremental rasterizer over a fresh cross-product one (1.35x at 8k
proposals).
