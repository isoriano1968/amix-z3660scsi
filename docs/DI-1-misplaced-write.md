# DI-1 — one 2 KiB piece of an inode block misplaced on the Z3660 SCSI path

**Analysis written 2026-10-03. No driver code was changed for it.** Tags: **measured** (read
off an image, a disassembly or a source line, cited), **derived** (arithmetic over measured
facts, shown), **assumed** (neither; the check that settles it is named).

## 1. Verdict

1. **The damage is one 2048-byte request's worth of data at the wrong LBA, and it was on the
   disk before the last session booted.** The piece at +2048 of the inode block for inodes
   6656–6719 holds the piece from +4096. It is byte-identical to the master's +4096 piece except
   bytes 16–23 (`ic_atime`) of three dinodes. Those atimes belong to the **inode numbers**
   6675/6676/6677 (the rc scripts S16firstboot, S99zen, S20monsw). The 10-03 session stamped them
   after it had loaded the already-copied dinodes from disk. They are not a "newer in-core copy".
   The session's own writes of this block were correct. *(measured, §2.2–2.3)*
2. **On the shipping kernel an 8 KiB metadata write never reaches this driver as one request.**
   `gen_strategy` → `buf_breakup` sends it as four 2048-byte WRITEs at consecutive LBAs
   (34304, 34308, 34312, 34316 for this block). Every request lies inside one 4 KiB page. The
   driver passes each one straight to the mailbox. *(measured, §3, disassembly of b9cee294)*
3. **Ranked first: a register-value race below the driver** (Z3660 firmware/FPGA, cross-repo).
   The third request (LBA 34312, source +4096) ran with the *previous* request's block number
   (34308) in `WRITE_ADDR1`. This one capture error yields exactly the observed table. Piece 2
   (+2048) is overwritten with piece 3 (+4096). Piece 3's own LBA is not written in that pass
   (it holds identical old bytes). Nothing else changes. The Z3660 fork already measures this
   ("the FPGA raises the write event before its AXI write of the value lands", f1e8f40,
   `regrace_check()`). *(derived; the firmware confirmation is owed, §4.1)*
4. **Refuted: the page-crossing mechanism in `z3660_rw`.** It is refuted by the path (no request
   spans a page) and by the shape (a crossing garbles 4–6 KiB, never exactly one 2 KiB piece).
   Also refuted: cache maintenance (16-byte granular), the bounce path (unreachable), the
   completion FIFO, and any ZS-04 write-side analogue. *(§4.4–4.8)*
5. **Recommended first experiment** (§7): a rig run of the shipping kernel with the amiberry
   D245 write capture on. It runs a tagged raw-copy torture and a metadata churn. It clears or
   convicts the kernel+driver request stream byte for byte. The rig's mailbox is synchronous by
   value, so it cannot show the firmware race. The same guest script on metal, with the regrace
   firmware, is the decisive second step.

## 2. Evidence re-derived from the images

Reference: the card master `k1-desktop-rtg.hdf`, sha256 `403ca277902ae0ac…`. Damaged: the same
file pulled from the D-DEV card on 2026-10-03, sha256 `6ad5ba8fac65d516…`, crc32 59EA6326
(equal to the device CRC). Both are 209 715 200 B. Read with `amix-installng/tools/lib/ufs.py`,
read-only mmap.

### 2.1 Geometry (measured)

| fact | value |
|---|---|
| RDB | block 0; UNIX_Root sectors 128–342143 (UFS base byte 0x10000), UNIX_Swap 342144–403583, UNIX_Boot 403584–407679 |
| UFS | magic 0x011954, **bsize 8192, fsize 1024, frag 8**, ncg 132, ipg 512, fpg 1296, inopb 64, sblkno 16, cblkno 24, iblkno 32, dblkno 96, cgoffset 16, cgmask 0xFFFFFFF0, fsbtodb 1, optim SPACE |
| inode 6656 | cg 13, `itod` = frag 17088 → fs byte 0x10B0000 → **image byte 0x10C0000 = image sector 34304** (root-relative sector 34176) |
| the four 2 KiB pieces | sectors 34304 / 34308 / 34312 / 34316 = inodes 6656–6671 / 6672–6687 / 6688–6703 / 6704–6719 |
| fs_time | master 1992-03-25 05:42:25 (install); damaged 1992-03-25 08:59:52 (last clean unmount) — box clock |

The .hdf is the whole piscsi unit, so an image sector number is the LBA the driver puts in
`WRITE_ADDR1`.

### 2.2 The block, piece by piece (measured)

| piece | inodes | damaged vs master |
|---|---|---|
| 0 (+0) | 6656–6671 | differs only in ordinary atime updates (rc scripts run at the 10-03 boot) |
| 1 (+2048) | 6672–6687 | = **master piece 2** except 15 bytes: offsets 16–23 (`ic_atime`) of dinodes 6675, 6676, 6677 |
| 2 (+4096) | 6688–6703 | **byte-identical to the master** |
| 3 (+6144) | 6704–6719 | byte-identical to the master |

Consequences in the namespace (measured): 6672–6674 (symlinks under /usr/net/nls) and 6675–6677
(S16firstboot, S99zen, S20monsw) now carry the dinodes of bkhtofile, bkid, bkmadd, bkmsg,
bkregdisp and bkregvals. 6683 (the directory /usr/sadm/bkup/bin) is now a regular file. 6678–6682
are no longer free. Hence fsck's DUPs on I=6688–6693 and the Bench's symptom (S20monsw = an ELF,
S99zen = bkregdisp).

### 2.3 When it happened — the atime bytes date it (measured + derived)

The 15 differing bytes are the atimes of inode **numbers** 6675/6676/6677:

| inode | name in the directory | atime written (box clock) |
|---|---|---|
| 6675 | /etc/rc2.d/S16firstboot | 08:48:02 |
| 6677 | /etc/rc2.d/S20monsw | 08:53:58 |
| 6676 | /etc/rc2.d/S99zen | 08:54:41 |

Clustering every post-install timestamp in the damaged image gives the session map. The 10-03
10:24→10:36 session is box 08:47:47 (o.log) → 08:59:52 (fs_time): mount 08:47:58, rc activity
from 08:48:00, login 08:52:30 (lastlog), the SEC-1 chmods 08:56, shutdown 08:59:3x. The last
first-boot artefacts (firstboot.conf ctime 08:45:22, /var/adm/firstboot 08:45:45,
/etc/inet/hosts 08:46:08) belong to the 10-02 20:47→20:50 session.

So the 10-03 boot executed "S16firstboot" at 08:48:02 from a dinode that already pointed at
bkmsg's data. Its atime was stamped on that copied dinode, and `iupdat` wrote back the whole
copied dinode with only the times changed. S20monsw and S99zen got their atimes about a minute
after login, which is consistent with the operator inspecting them. **Therefore:**

- the copy was on disk (or in the boot's first read) before box 08:48:02 on 10-03;
- piece 2 then held exactly the master bytes, so **the source of the copy cannot be told apart
  as in-core or on-disk**. The "newer time fields ⇒ in-core" reading does not hold;
- the 10-03 session's writes of this block were **correct** 4×2 KiB writes. They carried the
  damage plus the atime updates, and piece 2/3 were rewritten with unchanged content;
- the damaging event follows the 10-02 20:47 boot, which still ran the first-boot dialog from
  the intact S16firstboot. The most likely event is a write of this block during that session or
  its halt flush. *(assumed: the box-clock-to-session mapping. The Bench's console frames
  settle it.)*

### 2.4 Block-wise diff classification (measured)

814 differing sectors, all inside UNIX_Root. The RDB, swap and boot partitions are
byte-identical. The full run listing is in `DI-1-sector-diff.txt`.

| class | sectors | reading |
|---|---:|---|
| inode blocks | 273 | per dinode: times-only updates, alloc/free, and runtime content changes (wtmp/wtmpx, mnttab, nodename, profile, vfstab, saf pids, firstboot files, /dev/zen0 reusing /dev/syscon's slot, /etc/shadow rewritten so 2144 now holds syslog.pid, /tmp recreated in root.shadow's slot). **The only non-runtime pattern is the 16 dinodes of §2.2.** |
| cylinder-group headers | 163 | allocation maps and summaries |
| data, newly allocated | 199 | wtmpx/wtmp/utmpx/lastlog/cron log/.xinitrc and similar |
| data, reallocated | 62 | freed and reused fragments |
| data, freed | 45 | fragments of since-removed files |
| data, free in both | 42 | stale bytes of temp files created and removed |
| data, same owner | 30 | directories and small config files edited at runtime |

**Neighbour-copy scan.** Every changed 2 KiB piece was compared with the pieces ±2/4/6 KiB away,
in both images. One true copy was found: the DI-1 piece. There were five near-matches, all
explained: zero-dominated small-file fragments (nodename, syslog.pid, a /var/saf directory) and
/dev inode blocks whose dinodes share one template. This damage is a single event.

## 3. How an 8 KiB inode-block write reaches the mailbox on KD060-Z (measured)

Disassembly is of the shipping kernel `KD060-Z-c0-u6`, sha256 `b9cee2946bd2784d…` (the RT07c
twin is byte-identical). Addresses are ELF `.text` offsets. Its z3660 object is 53f43bd = HEAD
minus the ZS-04 guard; `z3660_rw` is unchanged since.

1. **`bwrite` → `bdevsw[].d_strategy` = `gen_strategy`.** `fix_swtbls` @0x3d684 copies every
   block major's strategy into `shadowbsw` and installs `gen_strategy` (all configured majors use
   `nullflag`; amix-kernel-analysis `vm-map/BIO-PFN-PHYS-KVA-CENSUS.md`). Buffer-cache data comes
   from `kmem_zalloc(bsize)` (`bio.c` getfreeblk); no PFN arithmetic is involved.
2. **`gen_strategy` @0x3d988:** for `b_bcount` ≤ 4096 it passes the buf to the real strategy.
   For more than 4096 with `b_pages == NULL` (any buffer-cache buf) it calls
   `buf_breakup(gen_strategy, bp)` (@0x3da0e).
3. **`buf_breakup` @0x3d17a** (Model-B patched first chunk, stock loop):
   - first chunk `d2 = min(bcount, 4096 − (b_addr & 4095))` (@0x3d1b4–0x3d1d0). If that is a
     whole page (> 4095) it skips straight into the loop (@0x3d1da);
   - loop chunk `d2 = min(resid, 0x800)`. `pea 0x800` @0x3d21e is the stock 2 KiB constant,
     deliberately left unpatched (port `patch_modelb.py` note);
   - each chunk is a copy of the buf with `b_bcount = d2`, through `gen_strategy` (≤ 4096 →
     `ddstrategy`). The loop waits for B_DONE, then `b_blkno += (d2+511)>>9` (@0x3d278) and
     `b_addr += d2` (@0x3d288).
   - **So every request lies inside one 4 KiB page, whatever the buffer's alignment.** An 8 KiB
     buffer at a page boundary becomes (34304, A), (34308, A+2048), (34312, A+4096),
     (34316, A+6144).
4. **`ddstrategy` @0xbe84 → `startio` @0xc0c8:** one `sdcom` per buf. The block comes from
   `sdblkno(bp)`, `addr = vtop(b_addr, b_proc)`, the CDB is WRITE(10), count = `b_bcount>>9`,
   nbyte = `b_bcount`. Both the LBA and the address are read from the same buf in one call. The
   dd completion-order shims are active: @0xc0ae → `dd_issue_first`, @0xc0b8 →
   `dd_startio_tail`.
5. **`z3660queue` → `z3660_rw` @0xf6ec0:** one chunk (≤ 64 KiB, so one for 2 KiB). The WRITE
   arm is `z3660_push`, then `+0x240 ← block`, `+0x244 ← len`, `+0x248 ← pa`, `+0x000 ← unit`
   (@0xf6f64–0xf6f70), then return. **Nothing is read after a WRITE doorbell.** The READ arm
   writes `+0x20/+0x24/+0x28`, rings `+0x04`, then reads `USED_DMA` (+0x9C).
6. **Raw path** (fsck, mkfs, tunefs, clri, dd to /dev/rdsk): `physio` → dd `breakup` @0xbe6a →
   `amiga_dma_pageio` @0xdba8 uses one `ngeteblk(0x800)` bounce for the whole transfer.
   Chunk = `min(resid, 0x800)`, `b_blkno += (d2+511)>>9`, `b_addr += d2`. So it is also 4×2 KiB
   at consecutive LBAs, with the **same** `pa` in every request.

The port station's reading ("ddstrategy queues the buf whole → one 16-block request") misses
steps 1–3. On this kernel the block path and the raw path both issue 2 KiB requests.

## 4. Ranked mechanisms

### 4.1 #1 — register-value race on the mailbox's block register (Z3660 firmware/FPGA)

**Mechanism.** On the real-CPU path the 060's register write lands in a memory cell at
`RTG_BASE + zaddr` and raises an event. The ARM takes the event and reads the value **from the
cell at service time**: `rtg.c` ~602–646, `zdata = swap32(*address)` →
`handle_piscsi_reg_write()`. `scsi.c` then latches `piscsi_u32_write[i] = val`
(@1702–1707). The WRITE doorbell seeks with `piscsi_u32_write[0]` (@1553–1557) and sources
`f_write` from `piscsi_u32_write[2]` (@1593–1620). If the event is serviced before the AXI write
of the new value lands, the latch keeps **the previous value written to that register**, i.e.
the previous request's.

**Which stale register gives which picture** (derived). For a run of consecutive 2 KiB writes k
= 0..3, LBA L+4k, source A+2048k:

| register captured stale in request k | effect | visible? |
|---|---|---|
| `WRITE_ADDR1` (block) | piece k's data written at LBA of piece k−1; piece k's LBA keeps old bytes | **yes — DI-1 = k=2** |
| `WRITE_ADDR3` (address) | piece k's LBA gets piece k−1's data (opposite direction) | yes (block path); invisible on the raw path (same bounce `pa`) |
| `WRITE_ADDR2` (length) | none while every request is 2048 | no |
| `DRVNUMX`, the doorbell value (unit) | none (always unit 6) | no |
| `READ_ADDR1` / `READ_ADDR3` | in-core piece k = disk piece k−1 (or lands one piece back); only reaches disk if written back, and then in the **opposite** direction to DI-1 | — |

Exactly one capture fits DI-1: request k=2's block register. A later-read variant fits too: the
address of request k read after request k+1 wrote it. It needs the FPGA to post at least four of
the 060's writes ahead of the ARM, and only the firmware log can tell the two variants apart. A
read-side event cannot produce DI-1's direction under either variant. So DI-1 is a **write**
event: one 8 KiB metadata write in which the third 2 KiB request landed one piece low.

**Driver's share.** The driver writes block → len → address → doorbell with nothing in between
and nothing after a WRITE doorbell. Nothing in the driver detects or prevents a stale latch. The
READ arm's `USED_DMA` read serializes completion, but it cannot undo a value latched stale at
event time.

**Confirm / refute.** On metal with f1e8f40: `[REGRACE] reg 0x240 …` lines (and 0x248, 0x020,
0x028) during a write-heavy run. The §7 tag script on metal then finds displaced pieces whose
direction matches the stale register. Refuted if displaced pieces keep appearing over a long
torture while REGRACE stays at zero stale first reads.

**Predictions.**
- The installng prevalence sweep should look for **both** directions ("piece k-1 = copy of
  piece k" and "piece k = copy of piece k-1") at 2 KiB granularity in metadata blocks.
- The B_PAGEIO path (`gen_strategy` multi-page, 4 KiB per request) gives the same pictures at
  4 KiB granularity.
- f1e8f40's `regrace_check` returns the re-read value, so a fork build carrying it already
  masks the stale-first-read variant. Which firmware the D-DEV card ran matters: the pull log's
  boot menu reads as 1.03 BETA 21.

### 4.2 #2 — the kernel pairs one chunk's LBA with another chunk's address

Refuted statically. `startio` reads `b_blkno` (via `sdblkno`) and `b_addr` (via `vtop`) from
the same buf in one call. The z3660 transfer happens inside that call, so the pair is
snapshotted. `buf_breakup` / `amiga_dma_pageio` advance both fields together, in process
context, after B_DONE. Interleaving would need `startio` to run from an interrupt between
@0x3d284 and @0x3d288, and even that gives (new LBA, old address): disk piece k ← memory piece
k−1, the opposite direction. **Residual:** kernel memory aliasing, i.e. another 2 KiB buffer
overlapping this buffer's +2048 slice (the Model-B 2 KiB/4 KiB history). That is port-owned, and
§7's capture convicts it if it is real.

### 4.3 #3 — firmware stale file pointer after a failed `f_lseek`

Stock firmware continues a request after a failed seek; the fork refuses it (`scsi.c` comment
@1308–1312). The write then lands at the previous request's end. For that to be +2048 →
displaced, the previous request must have ended at LBA 34312, which needs a duplicated or
reordered request. Ranked low. The serial log and the fast-seek (link map) state on the card
settle it.

### 4.4 #4 — page crossing in `z3660_rw` (the coordinator's candidate) — refuted

- **Path:** §3. No buffer-cache request above 4096 B reaches the driver unsplit, and every
  split piece lies inside one 4 KiB page. Identity-mapped buffers (below 0x40000000, DTT0) are
  physically linear anyway.
- **Shape** (derived). Suppose an 8 KiB request were sent whole from scattered 4 KiB frames:

  | buffer start | pieces sourced correctly | pieces sourced from the next physical frame |
  |---|---|---|
  | page-aligned | 0, 1 | 2, 3 |
  | page offset 2048 | 0 | 1, 2, 3 |

  Neither gives "piece 1 wrong, pieces 2–3 right". It would also take the next physical frame
  to hold this same block's other half, which is not credible.
- `z3660_pagecross_n` **is in the shipping object**: the test `(pa & 4095) + len > 4096`
  @0xf6f12–0xf6f24 uses 4 KiB on this line. By §3 it should read 0. A non-zero value means a
  ≤ 4096-byte non-pageio buffer that is not page-aligned (see §6.2), not DI-1.

### 4.5 #5 — data-cache maintenance — refuted

The shipping kernel has `z3660_cache` initialised to 0x3C (60) in `.data`, so CPUSHL+CINVL run
over [pa, pa+len) before each WRITE. Any cache error is 16-byte-line granular. A whole, exactly
aligned 2 KiB copy of another piece is not a cache signature.

### 4.6 #6 — bounce / shared static buffer — refuted

Driver staging fires only for `pa < 0x08000000`. AMIX RAM starts at 0x08000000, so it is
unreachable (`z3660_bounce_wr_n` counts it). The firmware sources from its own bounce only for
addresses outside its DDR window (`scsi.c` @1597–1600); that would garble every write, not one.
The raw path's single bounce is per-transfer, waits for B_DONE per chunk, and the transfer is
synchronous.

### 4.7 #7 — completion FIFO, duplicate issue, interrupt races — refuted

The data moves at issue time with a snapshotted (LBA, address) pair (§4.2). The FIFO only orders
the delivery of `ihandle`. A duplicate issue re-sends an identical pair. `z3660_cq_overflow` and
`z3660_nest_hits` are the in-kernel checks (§5).

### 4.8 #8 — ZS-04 / a 2048-byte unit block size — refuted

There is no shared block-size variable. `bs` comes from `z3660_geom[unit]` per command, and is
512 for unit 6. ZS-04 refuses `blocks > nbyte/bs`, which dd.c never produces on a 512-byte disk,
so the guard would not have fired. A wrong `bs` would misplace data ×4 in LBA and length, not
move one piece. The write-side analogue does not exist (a CD write is rejected with DATA
PROTECT before any doorbell). The shipping object lacking ZS-04 is irrelevant to DI-1.

## 5. Observables in the shipping kernel (measured, `KD060-Z-c0-u6.map`)

Runtime address = load base + offset. The map header measures the base as 0x08000000 on the
Z3660 060. **Verify it first:** `z3660_cache` must read 0x0000003C. Read with an lseek+read tool
(`zz9k_memprobe -q`), never `dd skip=`. All are BSS, so they cover the current boot only, not
the DI-1 session.

| symbol | offset | expected |
|---|---|---|
| `z3660_cache` (D) | 0x11dd50 | 0x3C (base check) |
| `z3660_pagecross_n` | 0x13e820 | 0 (§4.4) |
| `z3660_bounce_wr_n` | 0x1503e0 | 0 |
| `z3660_bounce_rd_n` | 0x13e578 | 0 |
| `z3660_cq_overflow` | 0x1417cc | 0 |
| `z3660_nest_hits` | 0x13d6d0 | 0 |
| `z3660_lastblock` / `z3660_lastlen` | 0x165c84 / 0x149a5c | last request |

The shipping object has no record of *which* chunk crossed. Driver objects built from source
after 2026-10-03 add `z3660_pagecross_ring`, one entry per counted crossing (unit, LBA, len, pa,
request base, R/W, seq). The layout and the decode recipe are in `docs/PAGECROSS-RING.md`.

## 6. Fix specs (no code in this round)

### 6.1 Driver defence against a stale register latch (relevant to DI-1)

The real fix belongs to the firmware/FPGA (latch after the AXI write lands, or carry the value
in the event). As defence in depth the driver can make every request self-checking:

- After writing `ADDR1/2/3` (READ or WRITE triple), **read each back** before the doorbell. The
  read handler returns the latched `piscsi_u32_*[i]` (`scsi.c` @2092–2096). On a mismatch,
  rewrite that register, re-read, and count `z3660_regrace_n`. After N retries, fail the request
  (EIO, no doorbell).
- After a **WRITE** doorbell, read one register (e.g. `P_DRVNUM`). This serializes completion
  like the READ arm's `USED_DMA` read.
- Keep the ZS-04 guard and the order DRVNUMX → ADDR1 → ADDR2 → ADDR3 → doorbell.
- **Cost:** 3 extra reads before the doorbell and 1 after a WRITE, against a 2 KiB SD write plus
  the existing 5 accesses. Measure it with the driver's existing per-request timing on the rig
  and on metal rather than assume it.
- **Tests:**
  - host harness: give `mock_regs.h` a "stale first latch" mode. Gates: the mismatch is detected
    and corrected, the counter moves, the retry bound fails the request, and the normal path's
    byte stream is unchanged;
  - rig: an amiberry fault-injection knob that latches the previous value with probability p
    (amiberry repo), plus the §7 tag torture with and without the read-back;
  - metal: REGRACE plus the tag torture.
- **This must not be read as papering over the firmware.** The counter makes the firmware defect
  visible, and the firmware fix remains owed.

### 6.2 Page splitting (asked for; not the DI-1 fix)

The driver **cannot** split at page boundaries. `sdcom.addr` is a physical address by contract
(NOTES 2026-07-11 (a)), and with no virtual address there is nothing to `vtop()` the next page
from. A driver bounce cannot gather the source pages either. The guarantee has to come from the
issuer, and on this kernel it does for every buf above 4096 B (§3). The remaining gap is a
≤ 4096-byte non-pageio buffer that is not page-aligned and not identity-mapped. Spec:

- keep `z3660_pagecross_n` (§5) as the gate;
- if it is ever non-zero, add a small kmem ring {block, len, pa} for crossing chunks and find
  the issuer;
- the fix then goes at the issuer (port repo): extend `gen_strategy`'s single-page test
  @0x3d994 to `(b_addr & 0xfff) + bcount ≤ 4096`, else route through `buf_breakup`;
- ZS-04 is kept. The hot path costs nothing.

## 7. Recommended first experiment — the rig, shipping kernel, D245 capture

**Goal.** Prove or convict the kernel + driver request stream (§4.2/4.4/4.6/4.7) byte for byte,
with the instrument that will later read metal. The amiberry mailbox applies register writes by
value and runs the doorbell synchronously (`src/z3660_scsi.cpp` ~397). **A clean rig therefore
moves DI-1 below the driver; it does not reproduce #1.**

**Host preparation** (durable scratch, e.g. `Amix/tmp/2026-10-03-di1-rig/`):
1. `root6.hdf` = a copy of the master (sha256 `403ca277…`, verify); keep `root6.pre.hdf` too.
   Its UNIX_Boot carries KD060-Z b9cee294, the kernel under test.
2. `di1scratch.hdf`, 48 MiB in three areas: A (sectors 0–32767), B (32768–65535, tags),
   C (65536–98303). Area B sector s = `b"DI1TAG\0\0" + pack(">II", s, 0xD11D1100)` + 496 bytes
   `(s*7+i) & 0xFF`. A and C start zeroed. Keep `di1scratch.pre.hdf`.
3. Config: the K1-desktop piscsi rig shape used for the cut-4 rows (kerntools
   `rig/matrix.tsv`), root on unit 6, plus
   `uaehfN=hdf,rw,DH5:/…/di1scratch.hdf,32,1,2,512,0,,scsi5_z3660` (unit 5 → `/dev/rdsk/c5d0s0`,
   well under the stock-sddevsize 603 MB edge).
4. Environment: `AMIBERRY_D245_LOG=<dir>/cap`, `AMIBERRY_D245_LOG_UNITS=di1` (matches both
   paths when the root copy is named `di1root6.hdf`), `AMIBERRY_D245_LOG_READS=hash`. Bring it
   up through `piscsi-rig.sh up` in a held slot.

**Guest script** (root shell on the console; pre-POSIX sh):
```
# raw path: 4 x 2 KiB requests per 8 KiB, consecutive LBAs, one bounce pa
R=/dev/rdsk/c5d0s0
dd if=$R of=$R bs=8k skip=2048 seek=0    count=2048    # B -> A
for p in 1 2 3; do
  dd if=$R of=$R bs=8k skip=0    seek=4096 count=2048  # A -> C
  dd if=$R of=$R bs=8k skip=4096 seek=0    count=2048  # C -> A
done
# block path: buf_breakup 4 x 2 KiB per inode block, the DI-1 path on the DI-1 layout
mkdir /var/tmp/di1; i=1
while [ $i -le 512 ]; do dd if=/dev/zero of=/var/tmp/di1/f$i bs=$i count=1 2>/dev/null; i=`expr $i + 1`; done
r=0; while [ $r -lt 30 ]; do touch /var/tmp/di1/*; sync; sleep 1; r=`expr $r + 1`; done
# counters (section 5), then a clean halt from the console root shell
sync; sync; init 0
```

**Host analysis:**
1. `amiberry/tools/d245-writelog.py verify cap --before *.pre.hdf --after *.hdf` for each unit:
   the emulator stored what it was told.
2. Tag scan of `di1scratch.hdf`: every sector of A and C must carry tag index (its offset within
   the area). Report mismatches per 2 KiB piece, with direction.
3. Capture scan: for every WRITE record to unit 5, each 512-byte sector's tag must equal LBA −
   area base. For root writes, `region --offset` over the inode blocks of /var/tmp/di1 (from
   `ufs.py`), and every 2 KiB piece must decode as dinodes whose number matches the directory.
4. Root: `ufscheck.py` reports 0 DUP fragments, and `/var/tmp/di1/fN` has size N.
5. §5 counters as expected.

**Reading the result.** All clean → the kernel and driver issue only consistent (LBA, address)
pairs on this path. Run the identical guest script on the A4000D with the Z3660 station's
regrace firmware: REGRACE lines plus tag mismatches there decide #1. Any rig mismatch → the
capture names the exact request (sequence number, LBA, length, payload), and the hunt moves to
the kernel (port) or this driver accordingly.
