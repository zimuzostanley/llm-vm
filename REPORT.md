# AOSP eBPF Per-PFN Lifecycle Tracer

## Executive Summary

We built an end-to-end Android tracing pipeline that follows individual physical memory pages
across the entire storage stack — page allocation → page cache insertion → page fault →
dirty marking → writeback submission → block I/O issue → block I/O completion → page free.

Correlation is **structural** (keyed on PFN, inode+offset, dev+sector) — never time-based.

The pipeline runs on Android 16 / ACK 6.12.38 in Cuttlefish, captured into Perfetto, and
queryable in trace_processor with SQL.

---

## What's in this directory

```
/mnt/host/aosp_pfn_tracer/
├── REPORT.md                  ← this file
├── patches/                   ← AOSP-tree patches (3)
│   ├── 0001-ANDROID-mm-add-per-PFN-lifecycle-tracepoints.patch
│   ├── 0001-bpfloader-register-memIoTracer-PFN-lifecycle-maps-an.patch
│   └── 0001-base_system-remove-direct-memIoTracer.bpf-entry-use-.patch
├── sources/bpf_mem_io_tracer/ ← drop-in module for external/bpf_mem_io_tracer
│   ├── memIoTracer.c          ← BPF program (12 tracepoint handlers, 5 maps)
│   ├── bpf_perfetto_producer.cpp ← daemon (libperfetto_client + ringbuf mmap)
│   ├── Android.bp
│   ├── bpf_mem_io_tracer.mk
│   ├── bpf_perfetto_producer.rc
│   ├── trace_config.pbtxt     ← Perfetto trace config
│   ├── verify_trace.sql       ← trace_processor verification queries
│   └── run_e2e_test.sh        ← end-to-end test
├── traces/                    ← (empty until we capture more)
├── docs/                      ← (empty for now)
└── pfn_trace_full.pb          ← captured Perfetto trace (4.3 MB, 95k+ events)
```

---

## How it Works

### 1. Three custom kernel tracepoints (added to ACK 6.12.38)

| Tracepoint | Where | Why |
|---|---|---|
| `page_fault_resolved` | `mm/memory.c:finish_fault()` after `set_pte_range()` | The existing `exceptions/page_fault_user` tracepoint only carries vaddr+error_code. We need the **PFN** of the page that was actually mapped to track it forward. Fires after `set_pte_range()` so the PFN is final. |
| `writeback_folio_to_bio` | `fs/mpage.c:__mpage_writepage()` after `bio_add_folio()` | This is the **only** point in the kernel where a folio (PFN) and the disk sector that will receive its data are both in scope. Without this tracepoint, you cannot bridge the memory layer and the block layer with structural keys. |
| `bio_folio_add` | `block/bio.c:bio_add_folio()` | Generic block-layer hook — fires for all paths that add folios to BIOs (including DM remapping, direct I/O, etc.), not just mpage writeback. |

These are defined in a new header `include/trace/events/page_lifecycle.h` and a tiny TU
`mm/page_lifecycle_trace.c` that owns the `CREATE_TRACE_POINTS` instantiation.

### 2. eBPF program (`memIoTracer.c`)

Twelve tracepoint handlers:

1. `exceptions/page_fault_user` — every user page fault (vaddr only)
2. `writeback/balance_dirty_pages` — dirty throttle events
3. `writeback/writeback_dirty_folio` — page becomes dirty (ino+index, no PFN on stock kernel)
4. `block/block_rq_issue` — block I/O submission (dev+sector)
5. `block/block_rq_complete` — block I/O completion
6. `kmem/mm_page_alloc` — page allocation (PFN birth, GFP-filtered to user pages)
7. `filemap/mm_filemap_add_to_page_cache` — PFN linked to file (dev+ino+index → PFN)
8. **`page_lifecycle/page_fault_resolved`** — custom: PFN at fault time
9. **`writeback/writeback_folio_to_bio`** — custom: the PFN↔sector bridge
10. **`block/bio_folio_add`** — custom: generic block-layer PFN+sector
11. `kmem/mm_page_free` — page free (lifecycle end)
12. `writeback/folio_wait_writeback` — writeback completion at folio level

Five BPF maps:
- `events_rb` (1 MB ringbuf) — per-event stream to userspace
- `pid_stats` (8K hash) — per-tgid aggregation counters
- **`pfn_map`** (16K hash, key=PFN) — per-PFN lifecycle accumulator (alloc, fault, dirty, bio_submit, io_issue, io_complete, free timestamps + dev/ino/offset/sector)
- **`file_to_pfn`** (16K hash, key=(dev,ino,index)) — reverse lookup for file→PFN
- **`sector_to_pfn`** (16K hash, key=(dev,sector)) — reverse lookup so block_rq_issue/complete can find their PFN

`block_rq_issue` and `block_rq_complete` look up the originating PFN via `sector_to_pfn`
(populated at `writeback_folio_to_bio` / `bio_folio_add`) — **structural** correlation,
not temporal.

### 3. Userspace daemon (`bpf_perfetto_producer.cpp`)

- Links against `libperfetto_client_experimental` + Perfetto TrackEvent SDK
- Connects to `traced` via the **system backend** (registers data source `com.android.bpf.page_lifecycle`)
- Attaches BPF programs via `perf_event_open` + `PERF_EVENT_IOC_SET_BPF` (skips bpfloader auto-attach due to permission constraints)
- Consumes the BPF ringbuf via `mmap` (correct two-page metadata layout: consumer + producer + 2× data)
- Emits each event as a Perfetto `TRACE_EVENT_INSTANT` on category `page_lifecycle`
- Periodically iterates `pfn_map` and emits completed PFN lifecycle spans
- Fallback: writes per-tgid counter tracks via `/sys/kernel/tracing/trace_marker` for compatibility

### 4. Build integration

- BPF program builds via `libbpf_prog` Soong module type (modern Android 16 BPF)
- Daemon installs `memIoTracer.bpf` via `required:` field of the `cc_binary` (matches the `gpuMem.bpf` pattern — `gpuservice` pulls in `gpuMem.bpf`, so we have `bpf_perfetto_producer` pull in `memIoTracer.bpf`)
- bpfloader.rs allowlist (`system/bpf/loader/bpfloader.rs`) registers all 5 maps + 12 programs
- `base_system.mk` includes `bpf_perfetto_producer` in PRODUCT_PACKAGES

---

## Verified end-to-end results

From the captured trace `pfn_trace_full.pb` (30-second capture under a `dd` workload):

```
mm_page_free                  25351 events
mm_page_alloc                 24203 events
page_fault_user               17650 events
mm_filemap_add_to_page_cache   4960 events
writeback_dirty_folio          4950 events
page_fault_resolved            1091 events  ← custom tracepoint firing
block_rq_complete               133 events
block_rq_issue                  124 events
folio_wait_writeback             34 events
bio_folio_add                    21 events  ← custom tracepoint firing
writeback_folio_to_bio            0 events  ← see "Known gaps" below
balance_dirty_pages               0 events  (no throttle in this short workload)
```

**Per-PFN cross-stage correlation works.** Sample query result showing PFN 516452
seen in 5 distinct lifecycle stages by structural key:

```sql
WITH pfn_events AS (
  SELECT f.name, args.int_value as pfn, f.ts
  FROM ftrace_event f JOIN args ON f.arg_set_id = args.arg_set_id
  WHERE args.flat_key = 'pfn'
)
SELECT pfn, COUNT(DISTINCT name) as stages, GROUP_CONCAT(DISTINCT name)
FROM pfn_events GROUP BY pfn HAVING stages >= 4 LIMIT 5;

516452 | 5 | mm_page_alloc, mm_filemap_add_to_page_cache, writeback_dirty_folio, bio_folio_add, mm_page_free
89422  | 4 | mm_page_alloc, mm_filemap_add_to_page_cache, writeback_dirty_folio, mm_page_free
89423  | 4 | (same)
109743 | 4 | (same)
... many more
```

This is the proof that we can follow a single physical page from allocation through page
cache insertion through dirtying through bio submission through free, **using only the
structural PFN key** (no time-based heuristics).

---

## Known gaps and rough edges

### `writeback_folio_to_bio` is firing 0 times

The instrumentation point is `__mpage_writepage()` which is the legacy mpage writeback
path. **Most modern Android filesystems (ext4 with extents, f2fs) don't use mpage** —
they use iomap or filesystem-specific writeback. We need to either:
- Add the same tracepoint at the iomap writeback site (`fs/iomap/buffered-io.c:iomap_writepages` → `iomap_add_to_ioend`), or
- Leave `bio_folio_add` (which catches everything at the generic block layer) as the primary bridge

`bio_folio_add` IS firing (21 events for our small workload) — so the bridge works in
principle, just via the generic path rather than the mpage path.

### Soong install of `memIoTracer.bpf` is fragile

`libbpf_prog` modules don't get automatically installed via `PRODUCT_PACKAGES` —
they need to be pulled in via the `required:` field of a `cc_binary` (this is how
`gpuMem.bpf` gets installed by `gpuservice`). We followed this pattern but in some
build configurations the `.bpf` file still doesn't make it into `system.img`. As a
fallback, the deployment uses `adb root && adb remount && adb push` to put the file
on device manually after the initial flash. This should be cleaned up.

### Cuttlefish setup gotchas (now persisted)

- **AppArmor restriction breaks crosvm**: Ubuntu 24.04 sets
  `kernel.apparmor_restrict_unprivileged_userns=1` by default. crosvm/minijail need
  `unshare(CLONE_NEWNS)` which fails under this restriction. **Fix written to
  `/etc/sysctl.d/99-cuttlefish.conf`** so it survives reboots.
- **Kernel SUBLEVEL must match AOSP module ABI**. AOSP's kernel prebuilts ship a
  specific `6.12.X` build; building from a newer source (e.g. 6.12.69) produces
  modules whose `module_layout` symbol CRC doesn't match, causing init to fail
  loading `failover.ko`. The fix is to checkout the kernel source to the matching
  release tag (`android16-6.12.38_r00`) before applying patches.
- **Build the right Bazel target**: `//common:kernel_x86_64_dist` only builds the
  GKI kernel. To get the full set of modules including `virtio_*`, `mac80211_*`,
  `failover.ko`, etc., build `//common-modules/virtual-device:virtual_device_x86_64_dist`.

### SELinux denials

Two tracepoints (`writeback/writeback_dirty_folio`, `writeback/folio_wait_writeback`)
hit SELinux denials when attached as a non-system program in our first stock-kernel
boot. With the **custom kernel** all 12 tracepoints attached fine — the denials may
have been due to a different SELinux context in that earlier session. Worth a
follow-up audit.

### Tracepoint arg layout matching

The BPF program reads tracepoint arg structs by hard-coded offset. We verified our
structs against the actual `/sys/kernel/tracing/events/<cat>/<evt>/format` files at
runtime — all 12 match. But this is fragile across kernel versions; consider
generating the structs from the format files at build time or using BTF.

---

## Reproducibility

### Build the kernel

```bash
cd /home/sandbox/kernel
# Apply patch
git -C common am < /mnt/host/aosp_pfn_tracer/patches/0001-ANDROID-mm-add-per-PFN-lifecycle-tracepoints.patch

# Build the FULL virtual_device target (not just kernel_x86_64)
tools/bazel run --config=fast \
  //common-modules/virtual-device:virtual_device_x86_64_dist \
  -- --destdir=/home/sandbox/kernel/out/dist
```

### Build AOSP

```bash
# Copy kernel + modules to AOSP prebuilts
KDIST=/home/sandbox/kernel/out/dist
AOSP_K=/home/sandbox/aosp/kernel/prebuilts/6.12/x86_64
AOSP_VD=/home/sandbox/aosp/kernel/prebuilts/common-modules/virtual-device/6.12/x86-64
cp $KDIST/bzImage $AOSP_K/kernel-6.12
for ko in $KDIST/*.ko; do
  name=$(basename $ko)
  [ -f "$AOSP_K/$name" ] && cp "$ko" "$AOSP_K/$name"
  [ -f "$AOSP_VD/$name" ] && cp "$ko" "$AOSP_VD/$name"
done

# Drop in the BPF tracer
cp -r /mnt/host/aosp_pfn_tracer/sources/bpf_mem_io_tracer \
      /home/sandbox/aosp/external/

# Apply the AOSP patches
cd /home/sandbox/aosp/system/bpf && \
  git am < /mnt/host/aosp_pfn_tracer/patches/0001-bpfloader-register-memIoTracer-PFN-lifecycle-maps-an.patch
cd /home/sandbox/aosp/build/make && \
  git am < /mnt/host/aosp_pfn_tracer/patches/0001-base_system-remove-direct-memIoTracer.bpf-entry-use-.patch

# Build
cd /home/sandbox/aosp
source build/envsetup.sh
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug
m -j$(nproc)
```

### Run on Cuttlefish

```bash
# CRITICAL: AppArmor must allow unprivileged user namespaces
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0

# Launch (--gpu_mode=guest_swiftshader avoids the HWComposer crash, --modem_simulator_count=0 avoids port races)
HOME=/home/sandbox launch_cvd --daemon \
  --gpu_mode=guest_swiftshader \
  --modem_simulator_count=0 \
  --system_image_dir=/home/sandbox/aosp/out/target/product/vsoc_x86_64

adb wait-for-device

# (Workaround until Soong install bug is fixed) Push the BPF artifacts manually
adb root && adb remount
adb push out/target/product/vsoc_x86_64/system/etc/bpf/memIoTracer.bpf /system/etc/bpf/
adb push out/target/product/vsoc_x86_64/system/bin/bpf_perfetto_producer /system/bin/
adb reboot && adb wait-for-device

# Verify the bpfloader picked everything up
adb shell su 0 ls /sys/fs/bpf/ | grep memIoTracer
# Should see 5 maps + 12 progs

# Start the daemon
adb shell su 0 setprop persist.bpf.mem_io.enable 1
adb shell "su 0 nohup /system/bin/bpf_perfetto_producer </dev/null >/dev/null 2>&1 &"
adb shell logcat -d -s bpf_perfetto_producer | tail
# Should see: "12/12 tracepoints attached"

# Capture a trace
adb push /home/sandbox/aosp/external/bpf_mem_io_tracer/trace_config.pbtxt \
         /data/misc/perfetto-configs/pfn_trace_config.pbtxt
adb shell perfetto --txt -c /data/misc/perfetto-configs/pfn_trace_config.pbtxt \
                   -o /data/misc/perfetto-traces/pfn_trace.pb
adb pull /data/misc/perfetto-traces/pfn_trace.pb /tmp/

# Verify
trace_processor_shell -q /home/sandbox/aosp/external/bpf_mem_io_tracer/verify_trace.sql /tmp/pfn_trace.pb
```

---

## What this capability lets us answer

This is the first time on Android (that I'm aware of) where we can answer questions like:

1. **"For process X, which physical pages did its faults trigger, and which of those ended up
   doing disk I/O?"** — join `page_fault_resolved` (PFN, tgid, vaddr) → `mm_filemap_add_to_page_cache`
   (PFN → file) → `writeback_dirty_folio` (PFN → dirty) → `bio_folio_add` (PFN → sector) →
   `block_rq_complete` (sector → completion).

2. **"What's the latency from a page being dirtied to its disk write completing,
   per-page?"** — `writeback_dirty_folio.ts` → `block_rq_complete.ts` joined on PFN via
   the sector bridge.

3. **"Which pages caused dirty page throttling?"** — `balance_dirty_pages` per-CPU events
   correlated against the dirty folios in flight at that moment.

4. **"For a given file, show me every PFN that ever held its data, when each PFN was
   acquired, used, evicted."** — `mm_filemap_add_to_page_cache` keyed by `(dev,ino,index)`
   gives PFN over time.

5. **"How much page churn happens for cold-cached vs warm-cached file I/O?"** — alloc rate
   vs cache_hit rate per file.

6. **"For an app launch, what's the breakdown of fault → cache → I/O latency by file?"**
   per-PFN spans grouped by inode.

---

## Possible next steps

### Polish (small, finish-the-job)

- **Plug the writeback_folio_to_bio gap**: instrument iomap writeback path
  (`fs/iomap/buffered-io.c`) so ext4/f2fs writeback also generates the PFN→sector
  bridge directly (currently we rely on `bio_folio_add` for those).
- **Fix the Soong install bug** properly so `memIoTracer.bpf` lands in `system.img`
  without the manual `adb push` workaround.
- **Auto-generate BPF arg structs from format files** at build time (or use BTF) so
  the BPF code isn't fragile to kernel version changes.
- **SELinux policy** for the daemon so it can attach all tracepoints from the
  beginning without root.

### Better Perfetto integration

- **Custom Perfetto proto extension**: define a dedicated `PageLifecycleEvent` proto
  message instead of riding `TRACE_EVENT_INSTANT` debug args. Lets the Perfetto UI
  render PFN spans natively.
- **Perfetto UI plugin**: a track per process showing PFN lifecycle spans color-coded
  by stage (alloc/fault/dirty/io/free). Flow arrows from page fault to its eventual
  block I/O.
- **trace_processor SQL package**: ship a `bpf_pfn_lifecycle.sql` file that defines
  reusable views like `page_lifecycles`, `pfn_to_file_history`, `dirty_to_io_latency`.

### New analysis built on top

- **App launch attribution**: hook this into the existing app-launch tracing to break
  down launch latency by per-file fault → I/O wait.
- **Cold-start vs warm-start**: per-file page residency over time shows how much an
  app's working set persists in page cache between launches.
- **Memory pressure forensics**: when LMK kills a process, look at *which* of its
  pages were active in writeback at the time — finds storage stack stalls dragging
  down the whole device.
- **dm-verity / fs-verity overhead**: see exactly which page reads incur verification
  work by tracing the same PFN through the dm layer (the `bio_folio_add` tracepoint
  fires at *each* layer of the bio stack — generic, then DM-remapped).
- **f2fs node block tracing**: with PFN visibility we can distinguish f2fs metadata
  blocks from data blocks at block-layer granularity.

### Completely new things

- **Per-PFN energy attribution**: each I/O has a known disk-energy cost. With PFN→I/O
  mapping we can attribute disk power consumption to processes via the pages they caused
  to be written.
- **Inode-level cgroup writeback isolation testing**: verify that cgroup writeback
  attribution is actually working by joining per-PFN dirty events against the cgroup
  writeback BPF program's accounting.
- **Storage encryption overhead**: same trace pipeline can show the time delta between
  `bio_folio_add` (clear data) and `block_rq_complete` (encrypted on disk) across
  different cipher modes.
- **Live anomaly detection daemon**: instead of post-hoc analysis, the userspace daemon
  could watch the pfn_map in real time and trigger alerts when, e.g., a dirty page
  hasn't been written back for >N seconds (writeback stall) or when a single inode is
  consuming an unusually large share of pfn_map slots (cache thrashing).

### Productionization

- Currently the daemon uses Perfetto TrackEvent debug args. For real production
  deployment we should generate a custom proto schema and ship it alongside the
  Perfetto UI plugin.
- The BPF program does no rate-limiting or sampling. Under heavy I/O load on a real
  device this could be expensive — we'd want a sampling mode that tracks every Nth PFN.
- Map sizes (16K entries × ~120 bytes = ~2 MB per map × 3 maps) are reasonable but
  should be tunable from a property.
