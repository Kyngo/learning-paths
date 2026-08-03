---
title: "File Systems"
weight: 5
---

# File Systems

A **file system** organizes persistent data on storage devices into a logical structure of files and directories. It translates high-level operations (open, read, write) into low-level block I/O while maintaining data integrity, supporting concurrent access, and managing metadata.

---

## File System Concepts

| Concept | Description |
|---------|-------------|
| **File** | Named collection of related data stored on secondary storage |
| **Directory** | Container that maps file names to file metadata |
| **Path** | Hierarchical location identifier (`/home/user/docs/file.txt`) |
| **Mount point** | Location where a file system attaches to the directory tree |
| **Superblock** | File system metadata (size, block count, free space) |
| **Block** | Minimum allocation unit (typically 4 KB) |

### File Metadata (stored separately from data)

```
┌─────────────────────────────────────────┐
│ File type       │ Regular, directory, symlink, device, pipe │
│ Permissions     │ rwxr-xr-x (owner/group/other)            │
│ Owner/Group     │ UID, GID                                  │
│ Size            │ Bytes of data                             │
│ Timestamps      │ atime, mtime, ctime                      │
│ Link count      │ Number of hard links                     │
│ Block pointers  │ Locations of data blocks on disk         │
└─────────────────────────────────────────┘
```

---

## Inodes

An **inode** (index node) stores all metadata about a file except its name:

```
Inode #42:
┌──────────────────────────────┐
│ type: regular file           │
│ permissions: 0644            │
│ owner: uid=1000              │
│ size: 13,847 bytes           │
│ link count: 2                │
│ blocks: 4                    │
│                              │
│ Direct pointers:             │
│   [0] → block 1024          │
│   [1] → block 1025          │
│   ...                        │
│   [11] → block 1035         │
│ Single indirect → block 2000│
│ Double indirect → block 3000│
│ Triple indirect → block 4000│
└──────────────────────────────┘
```

### Indirect Block Pointers (ext4 traditional mode)

```
Inode
├── Direct[0..11]              → 12 blocks          = 48 KB
├── Single indirect            → 1024 blocks        = 4 MB
├── Double indirect            → 1024² blocks       = 4 GB
└── Triple indirect            → 1024³ blocks       = 4 TB
```

Modern ext4 uses **extents** instead (contiguous block ranges), which are more efficient for large files.

### Hard Links vs Symbolic Links

| Type | Mechanism | Cross-filesystem | Dangling possible |
|------|-----------|-----------------|-------------------|
| **Hard link** | Multiple directory entries → same inode | No | No (link count > 0) |
| **Symbolic link** | Special file containing a path string | Yes | Yes (target deleted) |

---

## Directory Structure

A directory is a file that maps names to inode numbers:

```
Directory "/home/user/":
┌─────────────┬────────┐
│ Name        │ Inode  │
├─────────────┼────────┤
│ .           │ 1001   │  (self)
│ ..          │ 500    │  (parent)
│ docs        │ 1042   │
│ notes.txt   │ 1099   │
│ .bashrc     │ 1002   │
└─────────────┴────────┘
```

### Path Resolution

To access `/home/user/docs/file.txt`:

1. Read inode 2 (root `/`) → get directory blocks
2. Find "home" → inode 100
3. Read inode 100 → get directory blocks
4. Find "user" → inode 1001
5. Read inode 1001 → get directory blocks
6. Find "docs" → inode 1042
7. Read inode 1042 → get directory blocks
8. Find "file.txt" → inode 2048
9. Read inode 2048 → get file data blocks

Each step requires disk I/O — the **directory entry cache (dcache)** in Linux caches these lookups.

---

## File Allocation Methods

How does the file system map logical file blocks to physical disk blocks?

| Method | Description | Pros | Cons |
|--------|-------------|------|------|
| **Contiguous** | File occupies consecutive blocks | Fast sequential read, simple | External fragmentation, file can't grow |
| **Linked** | Each block points to next | No fragmentation, files grow easily | Slow random access, pointer overhead |
| **Indexed** | Index block holds all pointers | Fast random access, no fragmentation | Index block overhead |

### Contiguous Allocation

```
File A: blocks 0-4   │████████████
File B: blocks 7-9   │       │███████
                     └─── gap (external fragmentation)
```

### Linked Allocation (FAT-style)

```
Directory entry: start=3
Block 3 → Block 7 → Block 2 → Block 10 → NULL
```

### Indexed Allocation (ext-style)

```
Index block: [3, 7, 2, 10, 15, ...]
Direct access to any logical block via index.
```

---

## Journaling

A **journal** (write-ahead log) ensures file system consistency after crashes:

```
Without journaling:              With journaling:
1. Write data block              1. Write intent to journal
2. Update metadata               2. Write data block
   ← CRASH HERE                  3. Update metadata
   (metadata inconsistent!)      4. Mark journal entry complete
                                    ← CRASH at any point is safe
                                    (replay journal on recovery)
```

### Journal Modes (ext4)

| Mode | What's journaled | Performance | Safety |
|------|-----------------|-------------|--------|
| `journal` | Data + metadata | Slowest | Highest (no data loss) |
| `ordered` (default) | Metadata only (data written first) | Good | Good (no stale data) |
| `writeback` | Metadata only (data order not guaranteed) | Fastest | Stale data possible after crash |

---

## File System Comparison: ext4 vs XFS vs Btrfs

| Feature | ext4 | XFS | Btrfs |
|---------|------|-----|-------|
| **Max file size** | 16 TB | 8 EB | 16 EB |
| **Max volume size** | 1 EB | 8 EB | 16 EB |
| **Allocation** | Extents + bitmap | Extents + B+tree | Extents + B-tree (COW) |
| **Journaling** | Metadata (ordered) | Metadata only | COW (no traditional journal) |
| **Snapshots** | No | No | Yes (native, COW-based) |
| **Checksums** | Metadata only (optional) | No | Data + metadata |
| **Compression** | No | No | Yes (zlib, lzo, zstd) |
| **RAID** | No (use mdraid/LVM) | No (use mdraid) | Built-in (RAID 0/1/5/6/10) |
| **Shrink** | Yes | No | Yes |
| **Best for** | General purpose, boot | Large files, databases | Advanced features, NAS |
| **Maturity** | Very stable | Very stable | Stable (RAID5/6 still risky) |
| **Default in** | Debian, Ubuntu | RHEL 7+, SUSE | Fedora (planned), Synology |

---

## VFS (Virtual File System)

The **VFS** is a kernel abstraction layer that provides a uniform interface to all file systems:

```
User space:    open()  read()  write()  stat()  mkdir()
                │        │       │        │       │
                ▼        ▼       ▼        ▼       ▼
Kernel:     ┌─────────────────────────────────────────┐
            │         VFS Layer                        │
            │  (common interface: inode_operations,    │
            │   file_operations, super_operations)     │
            └──────┬──────────┬──────────┬────────────┘
                   │          │          │
                   ▼          ▼          ▼
            ┌──────────┐ ┌────────┐ ┌────────┐
            │   ext4   │ │  XFS   │ │  NFS   │ ...
            └──────────┘ └────────┘ └────────┘
```

### VFS Objects

| Object | Represents | Key Operations |
|--------|-----------|----------------|
| **Superblock** | Mounted file system | `alloc_inode`, `destroy_inode`, `sync_fs` |
| **Inode** | A file (any type) | `lookup`, `create`, `link`, `unlink`, `mkdir` |
| **Dentry** | Directory entry (name → inode cache) | `d_compare`, `d_delete`, `d_release` |
| **File** | Open file instance | `read`, `write`, `mmap`, `fsync`, `poll` |

---

## FUSE (Filesystem in Userspace)

**FUSE** lets you implement file systems as user-space programs:

```
Application:      open("/mnt/myfs/file")
                        │
Kernel VFS:             ▼
                  ┌───────────┐
                  │FUSE kernel│
                  │  module   │
                  └─────┬─────┘
                        │ (via /dev/fuse)
                        ▼
User space:       ┌───────────┐
                  │ FUSE      │
                  │ daemon    │  ← your code here
                  └───────────┘
```

### Common FUSE File Systems

| Name | Purpose |
|------|---------|
| sshfs | Mount remote directories over SSH |
| s3fs | Mount S3 buckets as local directories |
| rclone mount | Mount cloud storage (GDrive, Dropbox, S3) |
| encfs | Encrypted overlay file system |
| ntfs-3g | Full NTFS read/write support |

### FUSE Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| No kernel code needed | Extra context switches (user ↔ kernel) |
| Crash won't panic kernel | Higher latency than in-kernel FS |
| Any language (Python, Go, Rust) | Limited by FUSE protocol |
| Rapid development / prototyping | Not suitable for high-performance |

---

## File System Operations Internals

### Creating a File

```
1. Allocate inode from inode bitmap
2. Initialize inode (permissions, timestamps, size=0)
3. Add directory entry (name → inode number)
4. Update parent directory mtime
5. Write journal entry
```

### Deleting a File (unlink)

```
1. Remove directory entry
2. Decrement inode link count
3. If link count == 0 AND no open file descriptors:
   a. Free data blocks (update block bitmap)
   b. Free inode (update inode bitmap)
4. Write journal entry
```

The file persists until both conditions are met — this is why a process can still read a deleted file if it had it open.

---

## Key Takeaways

- Inodes separate file metadata from file names, enabling hard links and efficient metadata access
- Journaling prevents file system corruption by recording intent before making changes
- ext4 is the safe default; XFS excels with large files; Btrfs offers advanced features (snapshots, checksums) at some complexity cost
- The VFS layer means applications don't need to know which file system they're using
- FUSE democratizes file system development but adds latency from kernel-user transitions
- Path resolution requires multiple disk reads — the dcache makes this fast in practice
