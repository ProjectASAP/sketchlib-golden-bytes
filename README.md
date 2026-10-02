# sketchlib-golden-bytes

Golden byte-vectors for the ASAPv1 sketch wire format.

Each `.hex` file pins the exact bytes ASAPv1 emits for one fixed, known sketch
state. Every implementation of ASAPv1 must encode that state to these bytes and
decode these bytes back to that state. The spec is
[`docs/asapv1_wire_format.md`](https://github.com/ProjectASAP/asap_sketchlib/blob/main/docs/asapv1_wire_format.md)
in `asap_sketchlib`.

Each file is one line of lowercase hex (no `0x`, no whitespace) = the complete
ASAPv1 envelope `[ magic | version | kind_id | metadata_len | payload_len |
metadata | payload ]`.

## Using this repo

Consumers mount it as a git submodule at `asapv1_golden/`:

```bash
git submodule add https://github.com/ProjectASAP/sketchlib-golden-bytes.git asapv1_golden
```

| Consumer | Test |
| -------- | ---- |
| `asap_sketchlib` | `tests/asapv1_golden.rs` — serializes each known state and asserts `== golden`; decodes each golden and asserts the state and a byte-identical re-encode |

## Design principle: state is fixed, not hashed

Every fixture is built from a **known raw sketch state** (specific register
bytes / matrix values set directly), never by hashing input values. So the
golden tests the **wire encoding**, isolated from the hash functions.

## Fixtures

| File | Sketch | kind_id | State |
| ---- | ------ | ------- | ----- |
| `hll_classic_p12` | HLL Classic, P12 | `01 01` | 4096 registers, set: `[0]=1, [1]=7, [100]=42, [4095]=3` |
| `hll_ertl_mle_p12` | HLL Ertl-MLE, P12 | `01 02` | same register pattern |
| `hll_hip_p12` | HLL HIP, P12 | `01 03` | same registers + `hip_kxq0=1.5, hip_kxq1=2.5, hip_est=3.0` |
| `cms_i64_regular_2x3` | Count-Min i64, RegularPath | `02 00` | 2×3 row-major `[[0,1,127],[128,300,65536]]` |
| `cms_f64_fast_2x3` | Count-Min f64, FastPath | `02 00` | 2×3 row-major `[[0.0,1.5,2.25],[3.75,4.125,5.0625]]` |
| `cs_i64_regular_2x4` | Count Sketch i64, RegularPath | `04 00` | 2×4 row-major `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]` |
| `cs_i64_fast_2x4` | Count Sketch i64, FastPath | `04 00` | same matrix — differs from the above only by `mode` |
| `cs_i32_regular_2x4` | Count Sketch i32, RegularPath | `04 00` | same matrix — differs from the first only by `counter_type` |
| `kll_f64_k200` | KLL f64, k=200 | `06 00` | integers `1..=50`, compaction seed 42 (recorded in metadata as `seed`) |
| `kll_i64_k200` | KLL i64, k=200 | `06 00` | integers `1..=50`, compaction seed 42 (recorded in metadata as `seed`) |

The CMS i64 fixture deliberately spans the msgpack integer width boundaries
(positive fixint / uint8 / uint16 / uint32) to lock the "non-negative integer →
uint family, minimal width" rule (spec Section 4).

The Count Sketch fixtures cover the **negative** side, which no other fixture
reaches, because Count Sketch cells are signed — it adds `±weight`: negative
fixint / int8 / int16 / int32, alongside positive fixint / uint8 / uint32.

All three Count Sketch files hold the same matrix, so each pair isolates one
metadata key: the two i64 files differ only by `mode`, and `cs_i32_regular_2x4`
differs from `cs_i64_regular_2x4` only in the `counter_type` value, `"i64"`
against `"i32"`. The payloads are byte-identical, because msgpack encodes an
integer at its minimal width whatever the source type is. So the i32 fixture
pins that the counter type reaches the bytes, and that nothing else does.

The KLL fixtures are a special case of "state is fixed, not hashed": KLL never
hashes — it orders raw numeric values — so inserting `1..=50` places exactly
those retained samples. `k=200` keeps the input below the level-0 capacity, so no
compaction fires (`num_levels = 1`, one level `[1..50]`) and the state is fully
deterministic. The fixed compaction seed (42) pins the carried coin state. Only
the compact KLL (`06 00`) has a golden; the dynamic variant (`06 01`) shares the
payload shape but lacks a seeded constructor.

## Coverage

The fixtures cover six `kind_id`s: HLL's three estimators, Count-Min, Count
Sketch and compact KLL. Every other `kind_id` the spec's registry marks
*implemented* — Bloom, Space-Saving, CMSHeap, CSHeap, DDSketch, Hydra's five
counter variants, Elastic, Coco, UniformSampling, KMV, the UnivMon family,
CountL2HH, ExponentialHistogram and EHSketchList — has **no fixture**. The spec
fixes their bytes; nothing here checks them.

## Changing a fixture

The bytes are authored by `asap_sketchlib` (rmp_serde is the reference
encoder); other implementations conform to them, never the reverse.

1. Commit the new or changed `.hex` here, with its row in the table above.
2. In each consumer, bump the `asapv1_golden` submodule to that commit and
   update its golden test in the same PR.
