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
golden tests the **wire encoding**, isolated from the hash functions, except
that decoding Coco checks each key's placement against its hash.

## Fixtures

| File | Sketch | kind_id | State |
| ---- | ------ | ------- | ----- |
| `hll_classic_p12` | HLL Classic, P12 | `01 01` | 4096 registers, set: `[0]=1, [1]=7, [100]=42, [4095]=3` |
| `hll_ertl_mle_p12` | HLL Ertl-MLE, P12 | `01 02` | same register pattern |
| `hll_hip_p12` | HLL HIP, P12 | `01 03` | same registers + `hip_kxq0=1.5, hip_kxq1=2.5, hip_est=3.0` |
| `hll_classic_p14` | HLL Classic, P14 | `01 01` | 16384 registers, set: `[0]=1, [1]=7, [8192]=42, [16383]=51` |
| `hll_ertl_mle_p14` | HLL Ertl-MLE, P14 | `01 02` | same register pattern |
| `hll_hip_p14` | HLL HIP, P14 | `01 03` | same registers + `hip_kxq0=16380.5, hip_kxq1=0.25, hip_est=4.125` |
| `cms_i64_regular_2x3` | Count-Min i64, RegularPath | `02 00` | 2×3 row-major `[[0,1,127],[128,300,65536]]` |
| `cms_f64_fast_2x3` | Count-Min f64, FastPath | `02 00` | 2×3 row-major `[[0.0,1.5,2.25],[3.75,4.125,5.0625]]` |
| `cs_i64_regular_2x4` | Count Sketch i64, RegularPath | `04 00` | 2×4 row-major `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]` |
| `cs_i64_fast_2x4` | Count Sketch i64, FastPath | `04 00` | same matrix — differs from the above only by `mode` |
| `cs_i32_regular_2x4` | Count Sketch i32, RegularPath | `04 00` | same matrix — differs from the first only by `counter_type` |
| `kll_f64_k200` | KLL f64, k=200 | `06 00` | integers `1..=50`, compaction seed 42 (recorded in metadata as `seed`) |
| `kll_i64_k200` | KLL i64, k=200 | `06 00` | integers `1..=50`, compaction seed 42 (recorded in metadata as `seed`) |
| `kll_dynamic_f64_k200` | KLLDynamic f64, k=200 | `06 01` | `[2.5, -1.0, 0.0, 1e300, -0.125, 42.0, 3.0e-5]` in that order, compaction seed 42 (not in metadata) |
| `kll_dynamic_i64_k200` | KLLDynamic i64, k=200 | `06 01` | `[0, 1, -1, 127, -32, 128, -33, 255, -128, 256, -129, 65535, -32768, 65536, -32769, 4294967295, -2147483648, 4294967296, -2147483649, i64::MAX, i64::MIN]` in that order, compaction seed 42 (not in metadata) |
| `ddsketch_positive_a001` | DDSketch, α=0.01, positive only | `05 00` | `metadata_version` 1; positive store `[1,0,127,128,300,65536,4294967296]` at offset `-40`; `sum=2181071000.0, min=0.453125, max=0.5078125` |
| `ddsketch_signed_a001` | DDSketch, α=0.01, signed | `05 00` | `metadata_version` 2; positive store `[3,0,2]` at offset `310`; negative store `[5,1]` at offset `-208`; `zero_count=7`; `sum=2523.90625, min=-0.016, max=515.0` |
| `ddsketch_empty_a001` | DDSketch, α=0.01, empty | `05 00` | `metadata_version` 1; empty positive store at offset `0`; `sum=0.0, min=+inf, max=-inf` |
| `cmsheap_i64_regular_2x3_strkeys` | CMSHeap i64, RegularPath, `string` keys | `03 00` | 2×3 row-major `[[0,1,127],[128,300,65536]]`; `k=5`; heap `{"hot":65536, "warm":300, "mild":300, "cold":1}` |
| `cmsheap_i32_fast_2x3_i64keys` | CMSHeap i32, FastPath, `i64` keys | `03 00` | same matrix; `k=3`; heap `{-1:7, -129:7, 4294967296:3}` |
| `cmsheap_i64_regular_2x3_i64tie` | CMSHeap i64, RegularPath, `i64` keys | `03 00` | same matrix; `k=5`; heap `{2:9, -1:5, 1:5, 0:5, i64::MIN:5}` |
| `cmsheap_i64_regular_2x3_strtie` | CMSHeap i64, RegularPath, `string` keys | `03 00` | same matrix; `k=5`; heap `{"hot":9, "b":5, "aa":5, "Z":5, "a":5}` |
| `cmsheap_i64_regular_2x3_empty` | CMSHeap i64, RegularPath, empty heap | `03 00` | same matrix; `k=4`; no entries |
| `csheap_i64_regular_2x4_strkeys` | CSHeap i64, RegularPath, `string` keys | `0a 00` | the Count Sketch 2×4 matrix `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]`; `k=5`; heap `{"alpha":4294967296, "beta":127, "delta":127, "gamma":-33}` |
| `hydra_kll_2x2_k200` | Hydra, KLL counter (k=200, m=8), schema `["region","service"]` | `07 00` | 2×2 grid, row-major cells: `[1.0..=5.0]` seed 1, empty seed 2, `[2.5, -1.0, 0.0, 1e300, -0.125]` seed 3, `[3.0e-5]` seed 4; each coin is `[seed, 0, 0]` |
| `hydra_cm_2x2_counter_2x2` | Hydra, Count-Min counter (2×2 i32, FastPath), same schema | `07 01` | 2×2 grid, row-major cells: `[[0,1],[127,128]]`, `[[255,256],[300,65535]]`, `[[65536,1000000],[2147483647,0]]`, all zero |
| `hydra_cs_2x2_counter_2x2` | Hydra, Count Sketch counter (2×2 i32, FastPath), same schema | `07 02` | 2×2 grid, row-major cells: `[[0,-1],[127,-32]]`, `[[-33,128],[-128,-129]]`, `[[-32768,65536],[-32769,2147483647]]`, `[[-2147483648,1],[0,0]]` |
| `hydra_hll_1x2_p14` | Hydra, HLL Ertl-MLE counter (P14), same schema | `07 03` | 1×2 grid; cell 0 registers `[0]=1, [1]=7, [100]=42, [16383]=3`; cell 1 `[0]=2, [8192]=51`; all others 0 |
| `hydra_univmon_1x2` | Hydra, UnivMon counter (2 layers of 1×2, heap 2, `u64` keys), same schema | `07 04` | 1×2 grid; cell 0: layer 0 counts `[5,-3]`, l2 `34`, heap `{7:5, 300:2}`, incomplete; layer 1 counts `[0,2]`, l2 `4`, heap `{4294967296:2}`, complete; total weight 7, standard mode; cell 1 empty |
| `coco_3x7` | Coco, 3×7 table | `0c 00` | 15 occupied buckets `(row, col): key=value`: `(0,0) "uint32-min"=65536`, `(0,1) "uint16-max"=65535`, `(0,2) "fixint-max"=127`, `(0,3) ""=1`, `(0,4) "uint8-max"=255`, `(0,5) "emoji-😀"=5`, `(1,0) "uint16-min"=256`, `(1,1) "str8-min-32-bytes-0123456789abcd"=3`, `(1,2) "uint8-min"=128`, `(1,3) "clé-ünïcode-流量"=4`, `(1,4) "fixstr-max-31-bytes-0123456789a"=2`, `(1,5) "uint64-min"=4294967296`, `(1,6) "zero"=0`, `(2,1) "uint64-max"=u64::MAX`, `(2,5) "uint32-max"=4294967295`; the other 6 unoccupied |
| `elastic_4b_2x4` | Elastic, 4 heavy buckets, light 2×4 i32 RegularPath | `0b 00` | heavy `(flow_id, vote+, vote-, eviction)`: free with the flag set, `("10.0.0.1:443>192.168.10.20:5123",127,128,false)`, `("",1,65535,true)`, `("10.0.0.1:443>192.168.10.20:51234",2147483647,256,true)`; light row-major `[[0,255,65536,2147483647],[-1,-33,-32768,-2147483648]]`; `stale_copies=false` |
| `elastic_4b_2x4_stale` | Elastic, same geometry | `0b 00` | same state — differs from the above only by `stale_copies=true` |
| `univmon_str_l3_2x4_h5` | UnivMon, 3 layers of 2×4, heap 5, `string` keys | `10 00` | layer 0 counts `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]`, l2 `[4294999809, 4611686019501130818]`, heap `{"alpha":65536, "beta":300, "delta":128}`, complete; layer 1 counts `[[3,-2,0,1],[0,0,5,-4]]`, l2 `[14, 41]`, heap `{"gamma":5}`, incomplete; layer 2 counts `[[0,7,0,0],[-6,0,0,0]]`, l2 `[49, 36]`, heap `{"epsilon":9, "zeta":2}`, complete; total weight 70000, standard mode |
| `univmon_i64_l3_2x4_h5` | UnivMon, same shape, `i64` keys | `10 00` | same layers — differs from the above only by `key_type` and keys: layer 0 heap `{i64::MIN:65536, -1:300, -129:128}`, layer 1 heap `{128:5}`, layer 2 heap `{4294967296:9, 7:2}` |
| `univmon_empty_l3_2x4_h5` | UnivMon, same shape | `10 00` | freshly constructed: all counts and l2 zero, heaps empty, every layer complete, total weight 0, unset mode; `key_type` `u64` |
| `count_l2hh_2x4_seed7` | CountL2HH, 2×4, seed index 7 | `19 00` | counts row-major `[[127,128,65535,-32768],[-32,-33,-2147483648,i64::MIN]]`; l2 `[65536, i64::MAX]` |
| `set_aggregator_strings` | SetAggregator | `08 00` | `{"", "ab", "abc", "abcdefghijklmnopqrstuvwxyz012345", "api", "fixstr-max-31-bytes-0123456789a", "web", "é", "中", "～", "😀"}` |
| `set_aggregator_empty` | SetAggregator, empty | `08 00` | `{}` |
| `delta_result_strings` | DeltaResult | `09 00` | added `{"queue", "été", "😀"}`; removed `{"", "cache", "db", "中"}` |
| `delta_result_empty` | DeltaResult, empty | `09 00` | added `{}`; removed `{}` |

The HLL fixtures set the register bytes directly. Each estimator has a P12 and
a P14 fixture; the P14 set holds register value 51, the largest a P14 register
takes, and touches the first, a middle and the last index.

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
hashes — it orders raw numeric values — so inserting known values places exactly
those retained samples. `k=200` keeps every input below the level-0 capacity, so
no compaction fires (`num_levels = 1`, `levels = [0, n]`, items in input order)
and the state is fully deterministic. The fixed compaction seed (42) pins the
carried coin state, `[42, 0, 0]`. Compact KLL records the seed in metadata;
KLLDynamic never emits the `seed` key, so its metadata differs only by omitting
it. The two variants share the payload shape and differ by `kind_id`. The
dynamic f64 items are negative, zero and fractional; the dynamic i64 items span
positive fixint / uint8 / uint16 / uint32 / uint64 and negative fixint / int8 /
int16 / int32 / int64.

DDSketch never hashes, so its fixtures set the bucket stores, offsets, zero
count and the `sum` / `min` / `max` scalars directly. The positive fixture's
counts span positive fixint / uint8 / uint16 / uint32 / uint64 and its offset is
an int8. The signed fixture is `metadata_version` 2, which adds the negative
store and zero count; its offsets are a uint16 and an int16. The empty fixture
is a fresh sketch: no buckets and the `0.0` / `+inf` / `-inf` scalars. α is a
single metadata `f64`, so all three use 0.01.

The CMSHeap fixtures set the matrix and the heap entries directly. All reuse
the Count-Min i64 matrix; between them they cover two counter types, both modes
and two key types. Entries are emitted by descending count, ties by key: a
signed key compares as its two's-complement bit pattern read unsigned, so
`i64tie` emits `0, 1, i64::MIN, -1`; a string compares byte-wise, a proper
prefix first, so `strtie` emits `"Z", "a", "aa", "b"`. The `i64keys` keys span
negative fixint / int16 / uint64. The empty heap emits `key_type` `"u64"` with
two empty arrays.

The CSHeap fixture sets the Count Sketch matrix and the heap entries directly,
so its `counts` array is byte-identical to the Count Sketch fixtures'. Its heap
is one entry short of `k` and holds a count tie (`beta` before `delta`). The
heap counts span uint64 / positive fixint / negative int8: a CSHeap heap count
is a signed median.

The Hydra fixtures set every cell's state directly, so neither the subkeys nor
the values are hashed: the matrix cells from storage, the HLL registers by
pre-hashed values crafted to land on each index and rank, the KLL cells by
inserting raw values (k=200, so no compaction fires) under a distinct
compaction seed per cell, and the UnivMon layers by counter deltas at named
cells plus explicit heap entries. Each grid keeps the empty cell's shape in the
bytes. The 2×2 grids pin grid row-major order; the matrix counters' 2×2 runs pin
row-major order inside a cell. The Count-Min counters span positive fixint /
uint8 / uint16 / uint32 up to `i32::MAX`; the Count Sketch counters add negative
fixint / int8 / int16 / int32 down to `i32::MIN`. The HLL fixture holds register
value 51, the largest a P14 register takes.

The Coco fixture sets every bucket's key and value directly. Each key sits in
the column its row hashes it to, folded `% cols`, because a decoder rejects any
other placement; the bytes themselves carry no hash. The width is 7, not a power
of two, so a port that folds with a mask (`& (cols-1)`) instead of `% cols`
misplaces the keys and fails the fixture. The values span positive fixint /
uint8 / uint16 / uint32 / uint64 at both ends of each width. The keys cover the
empty string (an occupied bucket, distinct from an unoccupied `nil` one), a
31-byte fixstr, a 32-byte str8, and UTF-8 code points of two, three and four
bytes; `"zero"` is an occupied bucket holding 0.

The Elastic fixtures set the heavy buckets and the light Count-Min cells
directly; no flow id is hashed. The heavy table holds a free bucket (`nil`)
whose eviction flag is set, as a vacated bucket keeps it, an empty flow id
(`""`) and ids of 31 and 32 bytes, the fixstr / str8 boundary. The
votes span positive fixint / uint8 / uint16 / uint32 up to `i32::MAX`; the light
row 0 spans uint8 / uint32 up to `i32::MAX` and row 1 negative fixint / int8 /
int16 / int32 down to `i32::MIN`. The two files differ in one byte, the
`stale_copies` bool.

The UnivMon fixtures set every layer directly: counters by deltas at named
cells, which carry each row's `l2` accumulator, and heap entries by explicit
`(key, count)` pairs; no hash reaches the bytes. The shape's four parameters
(3 layers, 2 rows, 4 columns, heap 5) are pairwise distinct, so no dimension can
stand in for another; the heaps hold 3, 1 and 2 entries. Layer 0 holds the
Count Sketch matrix, so the counters span positive fixint / uint8 / uint32 and
negative fixint / int8 / int16 / int32, and both of its `l2` values are uint64.
The heap counts span uint32 / uint16 / uint8 / positive fixint and the `i64`
keys int64 / negative fixint / int16 / uint8 / uint64 / positive fixint. The two
populated files differ only in `key_type` and `keys`. The empty file pins the
encoding of a pyramid with no keys, whose `key_type` is `u64`. The fixtures
cover `update_mode` 0 (unset) and 1 (standard), not 2 (terminal-only): only a
hashed insert selects that mode.

The CountL2HH fixture sets the counts, the per-row `l2` accumulators and the
seed index directly, through the sketch's serde form; no key is hashed. Each
`l2` is set apart from the counts, since it is carried state rather than the
row's sum of squares. The cells span positive fixint / uint8 / uint16 and
negative fixint / int8 / int16 / int32 / int64 down to `i64::MIN`; the `l2`
values are a uint32 and `i64::MAX`, the accumulator's saturation value, a
uint64. Between them every msgpack integer width appears.

The SetAggregator and DeltaResult fixtures set the string sets directly; neither
type hashes. Each array is in ascending UTF-8 byte order. The strings cover the
empty string, a proper-prefix chain (`"ab"` before `"abc"` before the 32-byte
key), a 31-byte fixstr and a 32-byte str8, and UTF-8 code points of two, three
and four bytes;
`"～"` (U+FF5E) precedes `"😀"` (U+1F600), the reverse of their UTF-16 order.

## Coverage

The fixtures cover twenty-one `kind_id`s: HLL's three estimators (at P12 and P14), Count-Min, CMSHeap,
Count Sketch, CSHeap, DDSketch, both KLL variants (compact and dynamic),
Hydra's five counter variants, Coco, Elastic, UnivMon, CountL2HH, SetAggregator
and DeltaResult. Every other
`kind_id` the spec's registry marks *implemented* — Bloom, Space-Saving,
UniformSampling, KMV, UnivMon Optimized, UnivMon-Q, ExponentialHistogram and
EHSketchList — has **no fixture**. The spec fixes their bytes; nothing here checks them.

## Changing a fixture

The bytes are authored by `asap_sketchlib` (rmp_serde is the reference
encoder); other implementations conform to them, never the reverse.

1. Commit the new or changed `.hex` here, with its row in the table above.
2. In each consumer, bump the `asapv1_golden` submodule to that commit and
   update its golden test in the same PR.
