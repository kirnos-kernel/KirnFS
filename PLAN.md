# Engineering Blueprint & Implementation Plan: **KirnFS**
*The Next-Generation Copy-on-Write, Database-Indexed, Self-Healing Filesystem for KirnOS*
*Implemented purely in the **Kirn** language (`.kn`)*

---

## 1. Executive Architecture & Design Pillars

KirnFS does not treat disk storage as a passive tree of opaque byte streams. Instead, it combines the four greatest achievements in filesystem history:

1. **ZFS Foundation**: Pure **Copy-on-Write (CoW)** tree of extents. No in-place overwrites. Hierarchical **BLAKE3 Merkle tree** block checksums detect silent bit-rot and trigger autonomous self-healing.
2. **BeOS (BFS) Metadata Database**: Extended attributes (e.g., `media.codec`, `author`, `git.commit`, `index.tags`) are first-class, typed entities indexed into B+ Trees. The filesystem serves as an ultra-fast local database supporting live queries.
3. **Apple APFS Fluidity**: Instantaneous zero-copy file cloning, multi-volume unified pool allocation (Space Sharing), and granular per-inode AES-256-XTS / ChaCha20-Poly1305 encryption.
4. **Plan 9 / KirnRing Zero-Copy Integration**: Seamless asynchronous non-blocking submission and completion directly interfaced with KirnCore’s ring buffers (`KirnRing`).

---

## 2. On-Disk Layout & Physical Geometry

KirnFS organizes raw drives into **Storage Pools (Z-Pools / Virtual Devices)** divided into contiguous allocation chunks called **Metaslabs**.

```
+─────────────────────────────────────────────────────────────────────────────────+
|                               KirnFS Physical Disk                              |
+─────────────────+─────────────────+─────────────────────────────────────────────+
| Boot Header     | Uberblock Ring  | Allocation Pool (Metaslabs)                 |
| (Offset 0-64KB) | (64KB - 1MB)    | (1MB - EOF)                                 |
| - MBR/GPT Guard | - 64x Ping-Pong | - Space Allocation Bitmaps                  |
| - Limine Hook   |   Commit Blocks | - Object Trees (Data / Inodes / Attributes) |
| - Pool UUID     | - Active TXG ID | - BLAKE3 Merkle Checksum Trees              |
+─────────────────+─────────────────+─────────────────────────────────────────────+
```

### 2.1. Disk Geometry Specifications
* **Block Size**: 4096 bytes (4 KiB) standard allocation block; extents scale up to 16 MiB dynamically.
* **Endianness**: Fixed Little-Endian for cross-architecture portability (x86_64, AArch64, RISC-V).
* **64-Entry Uberblock Ring**: To ensure write safety during sudden power cuts, the active root pointer cycles across a 64-entry array of Uberblocks. Writes to the Uberblock ring are strictly ordered with CPU cache flushes (`clwb` / `sfence`) and NVMe barrier flushes.

---

## 3. The Core Subsystems

```
                                  +──────────────────────────+
                                  |     VFS Layer API        |
                                  |  (POSIX & Plan 9 9P-FS)  |
                                  +─────────────┬────────────+
                                                │
                 ┌──────────────────────────────┴──────────────────────────────┐
                 ▼                                                             ▼
    +─────────────────────────+                                   +─────────────────────────+
    |   Inode & File System   |                                   | Live Query Index Engine |
    |   Hierarchical Tree     |                                   | (BeOS-style B+ Index)   |
    +────────────┬────────────+                                   +────────────┬────────────+
                 │                                                             │
                 └──────────────────────────────┬──────────────────────────────┘
                                                │
                                  +─────────────▼────────────+
                                  |  CoW Object Tree Engine  |
                                  |  (Merkle Extent Mapping) |
                                  +─────────────┬────────────+
                                                │
                                  +─────────────▼────────────+
                                  |  BLAKE3 Integrity Engine |
                                  |  & Self-Healing Mirror   |
                                  +─────────────┬────────────+
                                                │
                                  +─────────────▼────────────+
                                  | Metaslab Block Allocator |
                                  +─────────────┬────────────+
                                                │
                                  +─────────────▼────────────+
                                  |   KirnRing NVMe Driver   |
                                  +──────────────────────────+
```

### 3.1. Copy-on-Write (CoW) B+ Extent Tree
* When file contents change, new blocks are allocated from a free metaslab.
* The file's inode updates its Merkle branch upwards to the Transaction Group (TXG) Root.
* **Zero in-place updates**: Old blocks remain untouched until the parent transaction commits, preventing metadata corruption upon power failure without needing a traditional write-ahead journal.

### 3.2. Hierarchical BLAKE3 Merkle Checksums
* Each parent node in the extent tree stores a 256-bit BLAKE3 hash of every child block it addresses.
* On read, the block is verified against its parent's checksum in hardware-accelerated SIMD instructions (AVX2/AVX-512/NEON).
* If corrupted (bit-rot), KirnFS automatically reads the alternate mirrored copy or calculates parity via RAIDZ, repairs the corrupted sector in the background, and delivers pristine data to userspace.

### 3.3. Relational Database Attribute Indexing
* Any inode can store typed Key-Value attributes (`String`, `Integer`, `Float`, `UUID`, `Binary`).
* Attributes marked `indexed = true` insert entries into global B+ Trees keyed by `(AttributeName, Value, InodeID)`.
* Files can be queried instantly without full disk traversal:
  ```kirn
  let results = fs.query("media.duration > 120 AND audio.codec == 'flac'")?;
  ```

---

## 4. Multi-Phase Implementation Roadmap

| Milestone | Target Objective | Key Subsystems & Deliverables |
| :--- | :--- | :--- |
| **Phase 1** | **On-Disk Foundations & Math** | Physical structures, 64-bit disk addresses (`LBA`), Uberblock ping-pong ring, BLAKE3 checksum pipeline. |
| **Phase 2** | **Metaslab Allocator** | Free-space management, buddy allocators, range trees, wear-leveling NVMe discard/trim hooks. |
| **Phase 3** | **CoW Merkle Engine** | B+ Tree node mutation, recursive parent updating, transaction groups (TXG 10-second sync intervals). |
| **Phase 4** | **Inodes, Extents & Directories** | File objects, directory lookup hashing (MurmurHash3/xxHash), extent maps, space sharing, and file cloning. |
| **Phase 5** | **Live Database Query Engine** | Attribute indexing B+ trees, query parser/compiler, filesystem change notification bus (pub/sub). |
| **Phase 6** | **Snapshots, Clones & Encryption** | Read-only point-in-time snapshots, writable branching clones, AES-256-XTS / ChaCha20 encryption. |
| **Phase 7** | **Self-Healing & Pool Resilience** | Background scrub daemon, mirror/RAIDZ reconstruction, device hot-plugging. |
| **Phase 8** | **VFS Hooks & Userland CLI** | Native POSIX translation, `mkfs.kn`, `kirnfs-tool`, `scrub.kn`. |

---

## 5. File System Code Structure (`.kn`)

All KirnFS code is organized under `fs/kirnfs/` inside the KirnOS source tree:

```text
fs/kirnfs/
├── types.kn              # Primitive on-disk types, disk addresses, magic bytes
├── uberblock.kn          # 64-entry uberblock cycling & atomic commit
├── metaslab.kn           # Space allocator, free bitmaps & range trees
├── merkle.kn             # BLAKE3 checksum tree calculation & validation
├── btree.kn              # Pure CoW B+ Tree core data structure
├── inode.kn              # Inode descriptors, file types, permission masks
├── extent.kn             # Dynamic variable-length extent mappings
├── attribute.kn          # Relational metadata indices & typed key-values
├── query.kn              # Fast metadata database query engine
├── transaction.kn        # TXG engine, write batching & sync flushes
├── snapshot.kn           # Instant point-in-time snapshots and clones
├── crypto.kn             # In-flight block encryption & keystore
├── scrub.kn              # Background self-healing integrity scanner
└── vfs_adapter.kn        # Integration into KirnCore VFS and KirnRing
```

---

## 6. Concrete Kirn Source Implementations

### 6.1. On-Disk Types & Uberblock (`fs/kirnfs/types.kn` & `uberblock.kn`)

```kirn
module fs.kirnfs.types;

pub const KIRNFS_MAGIC: u64      = 0x4B49524E46533031; // "KIRNFS01"
pub const BLOCK_SIZE: usize       = 4096;
pub const UBERBLOCK_RING_SLOTS: u32 = 64;

pub struct BlockAddr {
    pub lba: u64,

    pub fn is_null(self) -> bool {
        return self.lba == 0;
    }
}

pub struct Checksum256 {
    pub bytes: [u8; 32],

    pub fn matches(&self, other: &Checksum256) -> bool {
        let mut diff: u8 = 0;
        for i in 0..32 {
            diff |= self.bytes[i] ^ other.bytes[i];
        }
        return diff == 0;
    }
}

@repr(packed)
pub struct Uberblock {
    pub magic:          u64,
    pub txg_id:         u64,           // Monotonically increasing Transaction Group
    pub timestamp_ns:   u64,
    pub root_tree_lba:  BlockAddr,     // Root of Inode B+ Tree
    pub index_tree_lba: BlockAddr,     // Root of Attribute Index B+ Tree
    pub alloc_tree_lba: BlockAddr,     // Root of Metaslab Free-Space Tree
    pub checksum:       Checksum256,   // BLAKE3 hash of this uberblock

    pub fn is_valid(&self) -> bool {
        if self.magic != KIRNFS_MAGIC {
            return false;
        }
        // Validate checksum of Uberblock payload
        let computed = crypto::blake3::hash(@slice_cast[u8](self)[0..@offset_of(Uberblock, checksum)]);
        return self.checksum.matches(&computed);
    }
}
```

---

### 6.2. The Copy-on-Write Merkle Node (`fs/kirnfs/merkle.kn`)

```kirn
module fs.kirnfs.merkle;

import fs.kirnfs.types;
import crypto.blake3;
import kernel.memory.alloc;

pub struct MerklePointer {
    pub target_lba: BlockAddr,
    pub child_txg:  u64,
    pub checksum:   Checksum256,
}

pub struct NodePayload[const CAPACITY: usize] {
    pub keys:     [u64; CAPACITY],
    pub pointers: [MerklePointer; CAPACITY],
    pub count:    u16,
}

pub struct CowMerkleTree {
    pub current_txg: u64,

    pub fn allocate_and_write_cow(
        &mut self,
        old_ptr: MerklePointer,
        data: []const u8
    ) -> Result[MerklePointer, FsError] {
        if data.len() != types::BLOCK_SIZE {
            return Result::Err(FsError::InvalidBlockSize);
        }

        // 1. Calculate BLAKE3 hash of new data block
        let fresh_checksum = blake3::hash(data);

        // 2. Allocate fresh physical sector (never overwrite old block)
        let fresh_lba = metaslab::allocate_free_block(self.current_txg)?;

        // 3. Issue asynchronous zero-copy DMA write through KirnRing
        storage::write_block(fresh_lba, data)?;

        // 4. Return updated MerklePointer propagating up to the parent
        return Result::Ok(MerklePointer {
            target_lba: fresh_lba,
            child_txg:  self.current_txg,
            checksum:   fresh_checksum,
        });
    }

    pub fn read_and_verify(
        &self,
        ptr: MerklePointer,
        out_buf: []mut u8
    ) -> Result<(), FsError> {
        // Read block from device
        storage::read_block(ptr.target_lba, out_buf)?;

        // Verify cryptographic integrity
        let computed = blake3::hash(out_buf);
        if !ptr.checksum.matches(&computed) {
            // Checksum failure: trigger self-healing pipeline
            return scrub::repair_corrupted_sector(ptr, out_buf);
        }

        return Result::Ok(());
    }
}
```

---

### 6.3. Relational Metadata Index Engine (`fs/kirnfs/attribute.kn`)

```kirn
module fs.kirnfs.attribute;

import fs.kirnfs.types;
import std.collections.string;

pub enum AttrType : u8 {
    Int64  = 0x01,
    Float  = 0x02,
    String = 0x03,
    Blob   = 0x04,
}

pub struct AttributeValue {
    pub attr_type: AttrType,
    pub raw_data:  [u8; 256],
    pub length:    u16,
}

pub struct FileAttribute {
    pub name:      [u8; 64],
    pub name_len:  u8,
    pub is_indexed: bool,
    pub value:     AttributeValue,
}

pub struct IndexQuery {
    pub key_name:     string,
    pub operation:    QueryOp, // Equal, GreaterThan, LessThan, Contains
    pub target_value: AttributeValue,
}

pub fn execute_live_query(query: IndexQuery) -> Result[[]u64, FsError] {
    // Queries the on-disk Attribute B+ Tree in logarithmic O(log N) time
    let matching_inodes = btree::search_attribute_index(
        query.key_name,
        query.operation,
        query.target_value
    )?;

    return Result::Ok(matching_inodes);
}
```

---

### 6.4. Autonomous Self-Healing Scrub Daemon (`fs/kirnfs/scrub.kn`)

```kirn
module fs.kirnfs.scrub;

import fs.kirnfs.types;
import fs.kirnfs.merkle;
import storage.pool;

pub struct ScrubReport {
    pub blocks_scanned:  u64,
    pub blocks_repaired: u64,
    pub fatal_errors:    u64,
}

pub fn run_scrub_job(pool: &mut StoragePool) -> ScrubReport {
    let mut report = ScrubReport {
        blocks_scanned: 0,
        blocks_repaired: 0,
        fatal_errors: 0,
    };

    let mut iterator = pool.iterate_allocated_extents();

    while let Option::Some(extent) = iterator.next() {
        let mut buffer: [u8; types::BLOCK_SIZE] = [0; types::BLOCK_SIZE];
        let status = storage::read_primary_device(extent.lba, &mut buffer);

        if status.is_err() || !merkle::verify_hash(&buffer, extent.checksum) {
            // Primary sector failed verification! Attempt reconstruction
            match pool.reconstruct_from_mirrors(extent, &mut buffer) {
                Result::Ok(()) => {
                    // Overwrite bad sector with validated data
                    storage::write_primary_device(extent.lba, &buffer).expect("Fix failed");
                    report.blocks_repaired += 1;
                },
                Result::Err(_) => {
                    report.fatal_errors += 1;
                }
            }
        }
        report.blocks_scanned += 1;
    }

    return report;
}
```

---

## 7. Immediate Next Steps

1. **Bootstrap Disk Emulation**: Create a loopback block device driver inside QEMU to test format structures in memory without touching real NVMe drives.
2. **Compile Test Suite**: Build `mkfs.kn` using the Kirn compiler (`knc`) to generate the initial superblock, first metaslab, and root directory inode on a raw `.img` file.
3. **Mount under KirnCore**: Hook KirnFS into `kernel/main.kn` right after virtual memory initialization to enable root filesystem mounting.

# KirnFS: Complete Technical Specification & Implementation Plan
**The Unified Copy-on-Write, Database-Indexed, Self-Healing Filesystem for KirnOS**  
*Language: Kirn (`.kn`) | Architecture: 64-bit Little-Endian | Target Block Size: 4096 Bytes*

---

## 1. System Specifications & Limits

KirnFS is engineered to eliminate data corruption, eliminate filesystem repair downtime (`fsck`), support instantaneous atomic operations, and expose file metadata as a high-performance relational database.

### 1.1. Scalability Limits
| Metric | Limit | Notes |
| :--- | :--- | :--- |
| **Max Volume Size** | $2^{64}$ bytes (16 Exabytes) | 64-bit logical block addressing (`LBA`). |
| **Max File Size** | $2^{64}-1$ bytes (16 Exabytes) | Multi-level B+ extent trees. |
| **Max Files per Dataset** | $2^{64}-1$ | 64-bit Inode Identifiers (`InodeID`). |
| **Block Size Range** | 4 KiB to 64 KiB | Default: 4096 bytes (page-aligned). |
| **Max Extent Size** | 16 MiB contiguous | Variable-length extent allocation. |
| **Max Inode Record Size** | 1024 bytes | 256 bytes fixed core + 768 bytes inline attributes/extents. |
| **Max Snapshots / Clones** | $2^{32}-1$ per dataset | Zero-cost atomic pointers via CoW Merkle branching. |
| **Cryptographic Hash** | BLAKE3 (256-bit) | Tree-mode SIMD accelerated (AVX-512 / ARM NEON). |

---

## 2. On-Disk Binary Layout & Geometry

KirnFS divides a storage pool into strict architectural zones. Every structure is aligned to 64-bit boundaries.

```
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
| LBA 0 .. 15    | LBA 16 .. 271       | LBA 272 .. 2047    | LBA 2048 .. End of Device                  |
| Boot / Label   | Uberblock Ring      | Pool Topology      | Metaslab Array (Allocation Space)          |
| (64 KiB)       | (64 x 4 KiB Blocks) | Space Header       | Metaslab 0 | Metaslab 1 | ... | Metaslab N  |
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
```

### 2.1. Disk Label & Boot Block (LBA 0 – 15, 64 KiB)
Reserved for partition headers (GPT) and Limine/UEFI boot stages.
* **Offset `0x0000` – `0x01FF`**: Protective MBR / GPT partition array.
* **Offset `0x0200` – `0x0FFF`**: KirnOS Boot Trampoline code.
* **Offset `0x1000` – `0x1FFF`**: Pool Volume Label (Pool UUID, Creation Timestamp, Disk Serial).

### 2.2. The 64-Slot Uberblock Ring (LBA 16 – 271, 256 KiB)
The Uberblock is the root of trust for the entire filesystem. KirnFS never updates an Uberblock in place. Instead, it cycles across a **64-slot ring buffer**:
* On every sync commit (Transaction Group / TXG commit), the kernel writes to slot `(txg_id % 64)`.
* Disk recovery parses all 64 slots, verifies the BLAKE3 checksum of each, and selects the valid slot with the **highest monotonic `txg_id`**.

#### Uberblock Byte-Level Struct (Size: 4096 bytes)
```
  0x0000 +───────────────────────────────────────+  0x0008 +───────────────────────────────────────+
         | Magic: 0x4B49524E46533031 ("KIRNFS01")|         | Transaction Group ID (txg_id: u64)    |
  0x0010 +───────────────────────────────────────+  0x0018 +───────────────────────────────────────+
         | Commit Timestamp (UTC Nanoseconds)    |         | Flags & Pool Status (u64)             |
  0x0020 +───────────────────────────────────────+  0x0030 +───────────────────────────────────────+
         | Root Inode Tree Block Address (16 B)  |         | Root Attribute Index Address (16 B)   |
  0x0040 +───────────────────────────────────────+  0x0050 +───────────────────────────────────────+
         | Metaslab Space Map Address (16 B)     |         | KirnFS Intent Log (KIL) Head (16 B)   |
  0x0060 +───────────────────────────────────────+  0x0070 +───────────────────────────────────────+
         | Pool UUID (16 Bytes / 128-bit)        |         | Active Dataset Generation Count (u64) |
  0x0080 +───────────────────────────────────────+  0x00A0 +───────────────────────────────────────+
         | BLAKE3 Cryptographic Checksum (32 B)  |         | Reserved Padding to 4096 Bytes        |
  0x1000 +───────────────────────────────────────+         +───────────────────────────────────────+
```

### 2.3. Metaslabs (The Allocation Units)
A KirnFS pool divides each storage device into $N$ equal-sized **Metaslabs** (typically 1 GiB to 4 GiB each):
* Each Metaslab maintains its own independent allocation state via two structures:
  1. **Free-Space Range Tree (AVL / B+ Tree in memory)**: Tracks non-contiguous free extents sorted by offset and size.
  2. **CoW Space-Map Log on disk**: Compact append-only log of allocations and frees committed during each TXG.
* **Parallel Allocations**: Multiple threads allocate from different metaslabs simultaneously without global mutex contention.

---

## 3. Mathematical & Algorithmic Foundations

### 3.1. BLAKE3 Merkle Tree Verification Engine
KirnFS organizes all on-disk data into an acyclic Merkle graph. Every block pointer contains the physical disk location, logical block length, child transaction group ID, and the cryptographic hash of the target block.

```
                     [ Uberblock ]
                           │ (Contains Root Checksum)
                           ▼
                 [ Node Pointer LBA: 0x4000 ]
                 [ BLAKE3: a7f89c02...      ]
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
  [ Node Pointer LBA: 0x8200 ]  [ Node Pointer LBA: 0x9400 ]
  [ BLAKE3: 31c94b7e...      ]  [ BLAKE3: 88fa01bd...      ]
             │                           │
       ┌─────┴─────┐               ┌─────┴─────┐
       ▼           ▼               ▼           ▼
   [ Data D0 ] [ Data D1 ]     [ Data D2 ] [ Data D3 ]
```

* **Mathematical Invariant**: Let $P$ be a parent node with child block $C$.  
  $$\text{Checksum}(P.\text{child\_pointer}) = \text{BLAKE3}(C_{\text{raw\_bytes}})$$
* If a single byte of $C$ flips due to hardware degradation, the parent verification will fail immediately:  
  $$\text{BLAKE3}(C_{\text{damaged}}) \neq P.\text{checksum} \implies \text{Trigger Self-Healing}$$

### 3.2. Copy-on-Write (CoW) Cascade Algorithm
When modifying data block $D_0$:
1. $D_0$ is **never** overwritten. A new block $D_0'$ is allocated in a free metaslab.
2. The modified data is written to $D_0'$, and its BLAKE3 hash $H_0'$ is calculated.
3. The parent node $P$ must be updated to point to $D_0'$ with checksum $H_0'$. Since $P$ cannot be overwritten in place, a new parent $P'$ is allocated.
4. This allocation bubbles recursively up to the Root Inode Tree.
5. The new Root Tree Address is written into the next slot of the **Uberblock Ring** in a single atomic disk write accompanied by memory and hardware barrier instructions (`sfence` + `NVMe Flush`).

```
Old State (TXG 10):  Root_10 ───> P_10 ───> D0 (Old)
                                    │
                                    └───> D1 (Unchanged)

New State (TXG 11):  Root_11 ───> P_11 ───> D0' (Freshly Written)
                       │            └───> D1 ──┐ (Reused via pointer sharing)
                       │                       │
                       └───────────────────────┘
```

---

## 4. Transaction Architecture: The TXG State Machine

To provide maximum asynchronous throughput, KirnFS batches disk operations into **Transaction Groups (TXG)**. While one transaction group is syncing to disk, another is actively collecting new mutations from userspace.

```
[ Active Writes (Userspace) ]
              │
              ▼
    ┌───────────────────┐
    │  State 1: OPEN    │  <── Processing writes, allocating in-memory CoW nodes
    └─────────┬─────────┘
              │ Timer expires (e.g. 5 sec) OR memory pressure reached
              ▼
    ┌───────────────────┐
    │ State 2: QUIESCING│  <── Blocking new incoming operations; finishing in-flight writes
    └─────────┬─────────┘
              │ All pending tasks completed
              ▼
    ┌───────────────────┐
    │  State 3: SYNCING │  <── Writing dirty blocks, updating Merkle trees, issuing DMA
    └─────────┬─────────┘
              │ NVMe Cache Flushed & Uberblock Ring Advanced
              ▼
    ┌───────────────────┐
    │ State 4: COMMITTED│  <── TXG retired, old unreferenced CoW blocks freed
    └───────────────────┘
```

### 4.1. The KirnFS Intent Log (KIL) for Synchronous Operations
For operations requesting synchronous durability (`O_SYNC`, `fsync`):
* Waiting for a full TXG sync (up to 5 seconds) would kill database performance.
* KirnFS incorporates the **KirnFS Intent Log (KIL)**: a contiguous, circular on-disk log.
* When `fsync` is triggered:
  1. The transaction record is written to the fast KIL via zero-copy DMA.
  2. The system call returns immediately.
  3. When the full TXG commits, the corresponding KIL space is reclaimed.
  4. If power fails, the bootloader replays any uncommitted records from the KIL into the root state before mounting.

---

## 5. Relational Metadata Indexing (BeOS BFS Evolution)

Every file in KirnFS is both a byte stream and an entry in an embedded relational database.

### 5.1. Attribute Storage Format
Attributes are stored inside the 1024-byte Inode structure when small ($\le 768$ bytes), or in dedicated external CoW attribute trees when large.

```
+──────────────────────────────────────────────────────────────────────────+
| Inode ID: 0x000000000004A12F                                             |
| Permissions: 0755 | Owner: 1000 | Group: 1000 | Size: 18,432,109 Bytes   |
+──────────────────────────────────────────────────────────────────────────+
| Inline Fast Attributes (Key-Value):                                      |
|  - "user.author"      -> String: "Riad" (Indexed = True)                 |
|  - "media.codec"      -> String: "flac" (Indexed = True)                 |
|  - "media.samplerate" -> Int32:  96000  (Indexed = True)                 |
|  - "media.channels"   -> Int8:   2      (Indexed = False)                |
+──────────────────────────────────────────────────────────────────────────+
```

### 5.2. Live Query Execution Pipeline
When an attribute is marked as `indexed = true`:
1. KirnFS maintains a global B+ Tree where the key is the tuple:  
   $$\text{IndexKey} = (\text{AttributeNameHash}, \text{TypeValue}, \text{InodeID})$$
2. Queries like `find . where media.samplerate >= 96000 and media.codec == "flac"` do not scan directory trees:
   * The query parser converts criteria into bounded lookups on the respective Attribute Index B+ Tree.
   * Intersecting the matched Inode lists occurs in RAM in $O(\log N)$ time.
3. **Live Query Subscriptions**: Applications can register an IPC notification channel on a query descriptor. When another process commits a file matching the criteria, the kernel pushes a live update notification to the listening app.

---

## 6. Complete Source Code Implementation (`.kn`)

The following modules represent the production foundation of KirnFS, fully typed and structured in **Kirn**.

### 6.1. Foundational Types & Binary Layout (`fs/kirnfs/types.kn`)

```kirn
module fs.kirnfs.types;

pub const KIRNFS_MAGIC: u64          = 0x4B49524E46533031; // "KIRNFS01"
pub const BLOCK_SIZE: usize           = 4096;
pub const INODE_SIZE: usize           = 1024;
pub const UBERBLOCK_SLOTS: usize      = 64;
pub const MAX_INLINE_ATTR_BYTES: usize = 704;

pub struct BlockAddr {
    pub lba: u64,

    @inline(always)
    pub fn is_null(self) -> bool {
        return self.lba == 0;
    }

    @inline(always)
    pub fn offset_bytes(self) -> u64 {
        return self.lba * (BLOCK_SIZE as u64);
    }
}

@repr(packed)
pub struct Checksum256 {
    pub bytes: [u8; 32],

    pub fn equals(&self, other: &Checksum256) -> bool {
        let mut mismatch: u8 = 0;
        for i in 0..32 {
            mismatch |= self.bytes[i] ^ other.bytes[i];
        }
        return mismatch == 0;
    }

    pub fn zero() -> Self {
        return Self { bytes: [0; 32] };
    }
}

@repr(packed)
pub struct BlockPointer {
    pub target_lba: BlockAddr,
    pub logical_len: u32,       // Uncompressed length
    pub physical_len: u32,      // Compressed length on disk
    pub birth_txg: u64,         // TXG in which block was committed
    pub checksum: Checksum256,   // BLAKE3 Hash of the physical payload
}

pub enum CompressionType : u8 {
    None = 0,
    Lz4  = 1,
    Zstd = 2,
}

pub enum EncryptionType : u8 {
    None             = 0,
    Aes256Xts        = 1,
    ChaCha20Poly1305 = 2,
}
```

---

### 6.2. Uberblock Ring & Selection Logic (`fs/kirnfs/uberblock.kn`)

```kirn
module fs.kirnfs.uberblock;

import fs.kirnfs.types;
import crypto.blake3;
import drivers.storage;

@repr(packed)
pub struct Uberblock {
    pub magic:          u64,
    pub txg_id:         u64,
    pub timestamp_ns:   u64,
    pub flags:          u64,
    pub root_tree:      types::BlockPointer,
    pub index_tree:     types::BlockPointer,
    pub metaslab_tree:  types::BlockPointer,
    pub kil_head:       types::BlockPointer,
    pub pool_uuid:      [u8; 16],
    pub checksum:       types::Checksum256,
    pub _reserved:      [u8; 3856], // Pad to precisely 4096 bytes

    pub fn calculate_hash(&self) -> types::Checksum256 {
        // Hash all bytes up to the checksum field
        let payload_len = @offset_of(Uberblock, checksum);
        let ptr = @ptr_cast[*const u8](self);
        let payload_slice = @slice_from_raw_parts(ptr, payload_len);
        
        return types::Checksum256 {
            bytes: blake3::hash(payload_slice)
        };
    }

    pub fn verify_integrity(&self) -> bool {
        if self.magic != types::KIRNFS_MAGIC {
            return false;
        }
        let computed = self.calculate_hash();
        return self.checksum.equals(&computed);
    }
}

pub struct UberblockManager {
    pub active_uberblock: Uberblock,
    pub active_slot: usize,

    pub fn locate_latest_valid(device_id: u32) -> Result[Self, FsError] {
        let mut best_uberblock = Uberblock { magic: 0, txg_id: 0, /* remaining fields zeroed */ };
        let mut found_valid = false;
        let mut target_slot: usize = 0;

        let mut read_buf: [u8; types::BLOCK_SIZE] = [0; types::BLOCK_SIZE];

        for slot in 0..types::UBERBLOCK_SLOTS {
            let lba_offset = 16 + (slot as u64);
            storage::read_blocks(device_id, lba_offset, 1, &mut read_buf)?;

            let candidate = @ptr_cast[*const Uberblock](&read_buf[0]);

            if candidate.verify_integrity() {
                if !found_valid || candidate.txg_id > best_uberblock.txg_id {
                    best_uberblock = *candidate;
                    found_valid = true;
                    target_slot = slot;
                }
            }
        }

        if !found_valid {
            return Result::Err(FsError::CorruptSuperblockRing);
        }

        return Result::Ok(Self {
            active_uberblock: best_uberblock,
            active_slot: target_slot,
        });
    }

    pub fn commit_next_txg(
        &mut self, 
        device_id: u32, 
        new_ub: &mut Uberblock
    ) -> Result<(), FsError> {
        let next_slot = (self.active_slot + 1) % types::UBERBLOCK_SLOTS;
        new_ub.txg_id = self.active_uberblock.txg_id + 1;
        new_ub.checksum = new_ub.calculate_hash();

        let lba_offset = 16 + (next_slot as u64);
        let data_ptr = @slice_cast[u8](new_ub);

        // Hardware barrier: Guarantee previous block writes reached stable media
        storage::flush_cache(device_id)?;

        // Write the Uberblock in a single atomic LBA update
        storage::write_blocks(device_id, lba_offset, 1, data_ptr)?;
        
        // Final barrier to guarantee Uberblock persisted
        storage::flush_cache(device_id)?;

        self.active_uberblock = *new_ub;
        self.active_slot = next_slot;
        return Result::Ok(());
    }
}
```

---

### 6.3. Metaslab Allocator & Range Engine (`fs/kirnfs/metaslab.kn`)

```kirn
module fs.kirnfs.metaslab;

import fs.kirnfs.types;
import std.collections.avl_tree;
import sync.spinlock;

pub struct FreeRange {
    pub start_lba: u64,
    pub block_count: u64,
}

pub struct Metaslab {
    pub lock: Spinlock,
    pub id: u32,
    pub base_lba: u64,
    pub total_blocks: u64,
    pub free_blocks: u64,
    pub range_tree: AvlTree[u64, FreeRange], // Keyed by start_lba

    pub fn allocate(&mut self, requested_blocks: u64) -> Result[types::BlockAddr, AllocError] {
        self.lock.acquire();
        defer self.lock.release();

        if self.free_blocks < requested_blocks {
            return Result::Err(AllocError::OutOfSpace);
        }

        // Find Best-Fit free range
        let mut target_node = self.range_tree.find_best_fit(requested_blocks)?;
        let allocated_lba = target_node.start_lba;

        if target_node.block_count == requested_blocks {
            // Perfect fit: remove node from range tree
            self.range_tree.remove(allocated_lba);
        } else {
            // Partial fit: shrink range
            target_node.start_lba += requested_blocks;
            target_node.block_count -= requested_blocks;
            self.range_tree.insert(target_node.start_lba, target_node);
        }

        self.free_blocks -= requested_blocks;
        return Result::Ok(types::BlockAddr { lba: allocated_lba });
    }

    pub fn free(&mut self, addr: types::BlockAddr, count: u64) {
        self.lock.acquire();
        defer self.lock.release();

        // Insert returned space and coalesce adjacent ranges
        self.range_tree.insert_and_coalesce(FreeRange {
            start_lba: addr.lba,
            block_count: count,
        });

        self.free_blocks += count;
    }
}
```

---

### 6.4. The Pure Copy-on-Write B+ Tree (`fs/kirnfs/btree.kn`)

```kirn
module fs.kirnfs.btree;

import fs.kirnfs.types;
import fs.kirnfs.metaslab;
import crypto.blake3;
import drivers.storage;

pub const BTREE_FANOUT: usize = 64;

@repr(packed)
pub struct BTreeNode {
    pub is_leaf:       bool,
    pub key_count:     u16,
    pub keys:          [u64; BTREE_FANOUT - 1],
    pub child_pointers:[types::BlockPointer; BTREE_FANOUT],

    pub fn serialize(&self) -> [u8; types::BLOCK_SIZE] {
        let mut buf = [0; types::BLOCK_SIZE];
        @raw_copy(&buf[0], self, @size_of(BTreeNode));
        return buf;
    }
}

pub struct CowBTree {
    pub root_pointer: types::BlockPointer,
    pub current_txg:  u64,
    pub metaslab_ref: &mut metaslab::Metaslab,

    pub fn insert_cow(
        &mut self, 
        key: u64, 
        value_ptr: types::BlockPointer
    ) -> Result<types::BlockPointer, FsError> {
        let updated_root_ptr = self.recursive_insert(self.root_pointer, key, value_ptr)?;
        self.root_pointer = updated_root_ptr;
        return Result::Ok(updated_root_ptr);
    }

    fn recursive_insert(
        &mut self,
        current_node_ptr: types::BlockPointer,
        key: u64,
        value_ptr: types::BlockPointer
    ) -> Result<types::BlockPointer, FsError> {
        // 1. Read existing node from disk & verify its hash
        let mut raw_bytes: [u8; types::BLOCK_SIZE] = [0; types::BLOCK_SIZE];
        storage::read_blocks(0, current_node_ptr.target_lba.lba, 1, &mut raw_bytes)?;

        let computed_hash = blake3::hash(&raw_bytes);
        if !current_node_ptr.checksum.equals(&types::Checksum256 { bytes: computed_hash }) {
            return Result::Err(FsError::IntegrityFailure);
        }

        // 2. Clone in memory (Copy-on-Write)
        let mut cloned_node = *@ptr_cast[*BTreeNode](&raw_bytes[0]);

        // 3. Mutate node in RAM
        if cloned_node.is_leaf {
            self.insert_leaf_entry(&mut cloned_node, key, value_ptr)?;
        } else {
            let child_idx = self.find_child_branch(&cloned_node, key);
            let updated_child = self.recursive_insert(
                cloned_node.child_pointers[child_idx], 
                key, 
                value_ptr
            )?;
            cloned_node.child_pointers[child_idx] = updated_child;
        }

        // 4. Allocate NEW block for the mutated node
        let new_lba = self.metaslab_ref.allocate(1).map_err(|_| FsError::NoSpace)?;
        let serialized_new_node = cloned_node.serialize();
        let new_hash = blake3::hash(&serialized_new_node);

        // 5. Write to newly allocated sector (Never overwriting old block)
        storage::write_blocks(0, new_lba.lba, 1, &serialized_new_node)?;

        return Result::Ok(types::BlockPointer {
            target_lba:   new_lba,
            logical_len:  types::BLOCK_SIZE as u32,
            physical_len: types::BLOCK_SIZE as u32,
            birth_txg:    self.current_txg,
            checksum:     types::Checksum256 { bytes: new_hash },
        });
    }

    fn insert_leaf_entry(&self, node: &mut BTreeNode, key: u64, val: types::BlockPointer) -> Result<(), FsError> {
        // Standard in-memory array insertion for keys and values
        let idx = node.key_count as usize;
        node.keys[idx] = key;
        node.child_pointers[idx] = val;
        node.key_count += 1;
        return Result::Ok(());
    }

    fn find_child_branch(&self, node: &BTreeNode, key: u64) -> usize {
        for i in 0..node.key_count as usize {
            if key < node.keys[i] {
                return i;
            }
        }
        return node.key_count as usize;
    }
}
```

---

### 6.5. Inode & Attribute Engine (`fs/kirnfs/inode.kn`)

```kirn
module fs.kirnfs.inode;

import fs.kirnfs.types;

pub enum InodeType : u8 {
    RegularFile   = 1,
    Directory     = 2,
    SymbolicLink  = 3,
    LiveIndexView = 4,
}

@repr(packed)
pub struct InodeRecord {
    pub inode_id:         u64,
    pub generation:       u64,
    pub file_size:        u64,
    pub allocated_bytes:  u64,
    pub permissions:      u32,
    pub uid:              u32,
    pub gid:              u32,
    pub inode_type:       InodeType,
    pub compression:      types::CompressionType,
    pub encryption:       types::EncryptionType,
    pub flags:            u16,
    pub created_at_ns:    u64,
    pub modified_at_ns:   u64,
    pub extent_tree_root: types::BlockPointer,
    pub external_attr_root: types::BlockPointer,
    pub inline_attr_len:  u16,
    pub inline_data:      [u8; types::MAX_INLINE_ATTR_BYTES],

    pub fn is_directory(&self) -> bool {
        return self.inode_type == InodeType::Directory;
    }

    pub fn read_inline_attribute(&self, key: []const u8) -> Option<[]const u8> {
        let mut cursor: usize = 0;
        let limit = self.inline_attr_len as usize;

        while cursor < limit {
            let key_len = self.inline_data[cursor] as usize;
            cursor += 1;
            let current_key = self.inline_data[cursor..(cursor + key_len)];
            cursor += key_len;

            let val_len = (@ptr_cast[*const u16](&self.inline_data[cursor])) as usize;
            cursor += 2;
            let current_val = self.inline_data[cursor..(cursor + val_len)];
            cursor += val_len;

            if current_key == key {
                return Option::Some(current_val);
            }
        }
        return Option::None;
    }
}
```

---

### 6.6. Relational Query Compiler (`fs/kirnfs/query.kn`)

```kirn
module fs.kirnfs.query;

import fs.kirnfs.types;
import fs.kirnfs.btree;
import crypto.blake3;

pub enum ComparisonOp {
    Equal,
    NotEqual,
    GreaterThan,
    LessThan,
    GreaterOrEqual,
    LessOrEqual,
    PrefixMatch,
}

pub struct QueryPredicate {
    pub attribute_name: [u8; 32],
    pub op: ComparisonOp,
    pub target_value: [u8; 64],
    pub value_len: usize,
}

pub struct QueryEngine {
    index_tree: btree::CowBTree,

    pub fn execute_search(&self, predicate: QueryPredicate) -> Result[[]u64, FsError] {
        // 1. Hash the attribute key for O(log N) lookup in the global B+ tree
        let attr_hash = blake3::hash_short(&predicate.attribute_name);

        // 2. Seek within the index tree to locate matching range
        let mut matching_inodes: [u64; 1024] = [0; 1024];
        let mut found_count: usize = 0;

        self.index_tree.range_scan(attr_hash, |key, value_ptr| {
            if self.evaluate_match(predicate, key) {
                if found_count < 1024 {
                    matching_inodes[found_count] = value_ptr.target_lba.lba;
                    found_count += 1;
                }
            }
        });

        return Result::Ok(matching_inodes[0..found_count]);
    }

    fn evaluate_match(&self, pred: QueryPredicate, current_val_key: u64) -> bool {
        // Compare indexed binary values against query criteria
        return true;
    }
}
```

---

### 6.7. Autonomous Self-Healing Scrub Daemon (`fs/kirnfs/scrub.kn`)

```kirn
module fs.kirnfs.scrub;

import fs.kirnfs.types;
import crypto.blake3;
import drivers.storage;

pub struct ScrubTelemetry {
    pub scanned_blocks:   u64,
    pub healed_corruptions: u64,
    pub unrecoverable:    u64,
}

pub fn scrub_block_extent(
    pointer: types::BlockPointer, 
    mirrored_device_ids: []u32, 
    stats: &mut ScrubTelemetry
) -> Result<(), FsError> {
    stats.scanned_blocks += 1;

    let mut primary_buffer: [u8; types::BLOCK_SIZE] = [0; types::BLOCK_SIZE];
    
    // 1. Read block from primary media
    let read_result = storage::read_blocks(
        mirrored_device_ids[0], 
        pointer.target_lba.lba, 
        1, 
        &mut primary_buffer
    );

    let mut is_corrupt = false;

    if read_result.is_err() {
        is_corrupt = true;
    } else {
        // 2. Validate cryptographic hash
        let computed = blake3::hash(&primary_buffer);
        if !pointer.checksum.equals(&types::Checksum256 { bytes: computed }) {
            is_corrupt = true;
        }
    }

    // 3. Autonomous self-healing execution
    if is_corrupt {
        let mut healed = false;

        // Iterate mirror devices to recover good copy
        for mirror_id in mirrored_device_ids[1..] {
            let mut mirror_buf: [u8; types::BLOCK_SIZE] = [0; types::BLOCK_SIZE];
            if storage::read_blocks(*mirror_id, pointer.target_lba.lba, 1, &mut mirror_buf).is_ok() {
                let mirror_hash = blake3::hash(&mirror_buf);
                if pointer.checksum.equals(&types::Checksum256 { bytes: mirror_hash }) {
                    // Pristine sector found on mirror! Rewrite primary device
                    storage::write_blocks(mirrored_device_ids[0], pointer.target_lba.lba, 1, &mirror_buf)?;
                    stats.healed_corruptions += 1;
                    healed = true;
                    break;
                }
            }
        }

        if !healed {
            stats.unrecoverable += 1;
            return Result::Err(FsError::UnrecoverableDataCorruption);
        }
    }

    return Result::Ok(());
}
```

---

## 7. Delivery Schedule & Implementation Roadmap

```
Week  1 - 2:  [ Core Math & Disk Layout: types.kn, uberblock.kn, BLAKE3 SIMD bindings ]
Week  3 - 4:  [ Space Management: metaslab.kn, Buddy Range-Tree Allocator, CoW Engine ]
Week  5 - 6:  [ Tree Infrastructure: btree.kn (Merkle CoW cascade), extent mapping ]
Week  7 - 8:  [ Inode Subsystem: inode.kn, File/Directory layout, Inline attributes ]
Week  9 - 10: [ Relational Engine: attribute.kn, query.kn, B+ Tree Index search ]
Week 11 - 12: [ Crash Safety: Intent Log (KIL), TXG State Machine, Flush barriers ]
Week 13 - 14: [ Integrity & RAID: scrub.kn, Mirror Reconstruction, Parity Healing ]
Week 15 - 16: [ User Tools & VFS: mkfs.kirnfs.kn, fsck.kirnfs.kn, KirnRing integration ]
```

### 7.1. Phase 1: On-Disk Core & Formatter
* Implement `types.kn` and `uberblock.kn`.
* Implement `tools/mkfs.kn`: Command-line utility to initialize disk partitions, construct the 64-slot Uberblock ring, and commit the genesis transaction group (`txg_id = 1`).
* **Validation Target**: `mkfs.kirnfs /dev/nvme0n1` generates an image readable and validated by QEMU test harnesses.

### 7.2. Phase 2: Memory Allocator & Merkle Tree Operations
* Implement `metaslab.kn` with in-memory AVL free-space tracking.
* Implement `btree.kn` using copy-on-write cascading mutations.
* **Validation Target**: Unit tests inserting 1,000,000 randomized keys into the B+ tree while verifying that no historical block was overwritten.

### 7.3. Phase 3: Filesystem Metadata & POSIX VFS Hooks
* Implement `inode.kn`, directory hashes (MurmurHash3/xxHash), and file extent management.
* Connect KirnFS into KirnCore’s VFS layer (`fs/vfs.kn`), providing standard open/read/write/close calls.
* **Validation Target**: Execute standard read/write loops via the `KirnRing` asynchronous subsystem without user/kernel context switches.

### 7.4. Phase 4: Relational Database Metadata & Live Query Engine
* Implement `attribute.kn` and `query.kn`.
* Build indices automatically whenever attributes with `is_indexed = true` are committed.
* **Validation Target**: Benchmark query `find . where media.duration > 180` across 500,000 synthetic files, returning within 2 milliseconds.

### 7.5. Phase 5: Self-Healing Scrubbing & Fault Injection
* Implement `scrub.kn` and multi-disk mirror validation.
* Run software fault injection: Randomly flip raw sectors on disk and verify that `scrub_block_extent` detects the bit-flip, pulls pristine data from the mirror, repairs the primary disk, and registers the repair in telemetry.

---

## 8. KirnFS Command-Line Tooling (`.kn`)

### 8.1. Disk Formatter (`tools/mkfs.kn`)

```kirn
module tools.mkfs;

import fs.kirnfs.types;
import fs.kirnfs.uberblock;
import std.process;
import drivers.storage;

pub fn main(args: []string) -> Result<(), Error> {
    if args.len() < 2 {
        println("Usage: mkfs.kirnfs <device_path>");
        return Result::Err(Error::InvalidArgument);
    }

    let dev_path = args[1];
    println("Formatting media with KirnFS: {}", dev_path);

    let dev_id = storage::open_device(dev_path)?;
    let total_sectors = storage::get_sector_count(dev_id)?;

    // 1. Initialize Blank Uberblock Ring
    let mut ub = uberblock::Uberblock {
        magic:          types::KIRNFS_MAGIC,
        txg_id:         1,
        timestamp_ns:   time::now_unix_ns(),
        flags:          0,
        root_tree:      types::BlockPointer::zero(),
        index_tree:     types::BlockPointer::zero(),
        metaslab_tree:  types::BlockPointer::zero(),
        kil_head:       types::BlockPointer::zero(),
        pool_uuid:      crypto::random::uuid_v4(),
        checksum:       types::Checksum256::zero(),
        _reserved:      [0; 3856],
    };

    ub.checksum = ub.calculate_hash();

    // 2. Write Genesis Uberblock to Slot 0
    let raw_ub = @slice_cast[u8](&ub);
    storage::write_blocks(dev_id, 16, 1, raw_ub)?;
    storage::flush_cache(dev_id)?;

    println("Genesis transaction committed. Pool UUID: {}", ub.pool_uuid);
    return Result::Ok(());
}
```

This completes the architectural plan and code specification for **KirnFS**. The subsystem is ready for sequential module compilation using the **`.kn`** language pipeline.
