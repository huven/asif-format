# ASIF binary specification

With macOS 26, Apple introduced the Apple Sparse Image Format (ASIF) for virtual disks.

As usual, there are no technical details provided about the layout of the ASIF file.

This repository collects reverse-engineered technical details and community-contributed findings about the Apple Sparse Image Format ASIF, for the benefit of forensics engineers, virtualization tool authors, archivists, and researchers.

ASIF is a two‑level indirect block scheme.

## Header

Integers are stored in big‑endian order.

| Offset (hex) | Length (bytes) | Description                         |
|--------------|----------------|-------------------------------------|
| 0x00         | 4              | Magic 'shdw' (73 68 64 77)          |
| 0x04         | 4              | Version (ex.: 00 00 00 01)          |
| 0x08         | 4              | Header size (ex.: 00 00 02 00)      |
| 0x0C         | 4              | Flags                               |
| 0x10         | 8              | Offset to Directory (bytes)         |
| 0x18         | 8              | Offset to Directory (bytes)         |
| 0x20         | 16             | UUID                                |
| 0x30         | 8              | Sector count                        |
| 0x38         | 8              | Maximum sector count                |
| 0x40         | 4              | Chunk size (bytes)                  |
| 0x44         | 2              | Sector/block size (bytes)           |
| 0x46         | 2              | ??                                  |
| 0x48         | 8              | Metadata offset (unit: chunk size)  |

### Definitions

Let N be the number of data chunks per chunk‑group.  
Let S be the size of a chunk‑group in bytes.  
Let C be the number of chunk‑groups that fit into a single table.  
Let D be the total amount of data addressed by one table.  
Let T be the maximum number of tables required for the image.

```
N = 4 * sector size  
S = 8 * (N + 1)  
C = floor(chunk size / S)  
D = C * N * chunk size  
T = ceil(max sector count * sector size / D)
```

## Directory

The header references two directories. The directory with the highest sequence number is considered the active one.

To locate a table for a given logical offset, divide the offset by D (the data range covered by a single table). The integer quotient yields the index into the directory; the corresponding entry provides the chunk number that holds the desired table.

| Offset (hex) | Length (bytes) | Description                          |
|--------------|----------------|--------------------------------------|
| 0x00         | 8              | Sequence number                      |
| 0x08         | 8              | Chunk number that contains Table 0   |
| 0x10         | 8              | Chunk number that contains Table 1   |
...
| 8 * T        | 8              | Chunk number that contains Table T-1 |

## Table

Each table is composed of C chunk‑groups placed sequentially. To determine the chunk-group, divide the relative offset by N * chunk size (the data range covered by a single chunk-group).

| Offset (hex)   | Length (bytes) | Description                         |
|----------------|----------------|-------------------------------------|
| 0x00           | S              | Chunk group 0                       |
| S              | S              | Chunk group 1                       |
...
| S * (C - 1)    | S              | Chunk group C-1                     |

## Chunk Group

A chunk-group stores N data-chunk references followed by a single bitmap-chunk reference. The most-significant nine bits of each chunk number are reserved for flag bits. The remaining 55 bits represent the physical chunk index.

To determine the data chunk number, divide the relative offset by chunk size.

| Offset (hex) | Length (bytes) | Description                         |
|--------------|----------------|-------------------------------------|
| 0x00         | 8              | Data chunk 0                        |
| 0x08         | 8              | Data chunk 1                        |
...
| 8 * (N - 1)  | 8              | Data chunk N-1                      |
| 8 * N        | 8              | Bitmap chunk                        |

### Chunk Entry Flags

The high 9 bits of each 64-bit chunk entry are flags; the low 55 bits are the physical chunk index.

| Bit(s) | Mask                 | Label    | Notes                                                                 |
|--------|----------------------|----------|-----------------------------------------------------------------------|
| 63     | 0x8000000000000000   | Flag A   | Observed set on allocated data entries after clean unmount.           |
| 62     | 0x4000000000000000   | Flag B   | Observed set on allocated data entries after clean unmount.           |
| 61-55  | 0x3F80000000000000   | Reserved | Keep zero until their meaning is known.                               |

Observed behavior on macOS: when a chunk is allocated (data written), both high bits are set on the corresponding data entry and remain set after a clean eject. Bitmap entries appear unflagged. Treat these as allocation/state bits until more is known.

## Bitmap Chunk

The bitmap chunk is likely a 2-bits-per-sector map for the entire chunk-group.

From the definitions above:

- Chunk-group data size = `N * chunk size`
- Sectors per chunk-group = `(N * chunk size) / sector size` = `4 * chunk size`
- At 2 bits per sector, bitmap size = `(4 * chunk size * 2) / 8` = `chunk size`

This matches the bitmap chunk size exactly.

Observed behavior in sample images:

- Writes cause bytes in the bitmap to flip from `0x00` to `0x55` (bit pattern `01` repeated).
- A write at sector 2048 flipped the bitmap byte at offset `0x200` (since `2048 / 4 = 0x200`). Sector 2047 mapped to `0x1ff`.

Byte layout (one byte = 4 sectors):

```
bit:  7 6  5 4  3 2  1 0
      [s3] [s2] [s1] [s0]
```

Each byte packs four 2-bit tuples for four consecutive sectors. The ordering is LSB-first: sector 0 uses bits 1:0, sector 1 uses bits 3:2, sector 2 uses bits 5:4, and sector 3 uses bits 7:6. The metadata bitmap byte `0x05` at offset `0xFFE00` implies the first two sectors in that 4-sector group are set, which matches the 0x200-byte metadata header plus plist data.

Addressing formula:

```
byte offset = floor(sector index / 4)
bit pair = (sector index % 4) * 2
state = (bitmap byte >> bit pair) & 0x3
```

| Bits | Label   | Notes                                             |
|------|---------|---------------------------------------------------|
| 00   | State 0 | Observed in untouched sectors.                    |
| 01   | State 1 | Observed where sectors have data written.         |
| 10   | State 2 | Not observed yet.                                 |
| 11   | State 3 | Not observed yet.                                 |

Sectors marked 00 should be treated as sparse (returning all zeros on read).

## Metadata

An ASIF image contains metadata consisting of a header and property list (XML). To read the metadata, use offset

> metadata offset * chunk size

Note that this will read past the current sector count, but inside the maximum sector count (typically the theoretically last chunk of the image).

### Metadata header

| Offset (hex) | Length (bytes) | Description                         |
|--------------|----------------|-------------------------------------|
| 0x00         | 4              | Magic 'meta' (6d 65 74 61)          |
| 0x04         | 4              | Version (ex.: 00 00 00 01)          |
| 0x08         | 4              | Header size (ex.: 00 00 02 00)      |
| 0x0C         | 4              | Flags                               |
| 0x10         | 4              | Offset to metadata property list (bytes)    |

### Metadata property list

The [property list](https://en.wikipedia.org/wiki/Property_list) is an XML dictionary. In observed images it contains:

| Key               | Type  | Notes                                              |
|-------------------|-------|----------------------------------------------------|
| internal metadata | dict  | Contains a `stable uuid` string.                   |
| user metadata     | dict  | Present but empty in observed images.              |

`stable uuid` is a UUID string (unique per image).

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>internal metadata</key>
	<dict>
		<key>stable uuid</key>
		<string>dc5c7a3b-1915-43c2-944d-46c6c304b3b7</string>
	</dict>
	<key>user metadata</key>
	<dict/>
</dict>
</plist>
```

## Creating Test Images

```
$ diskutil image create blank --format ASIF --size 123g -fs None test.asif
$ diskutil image info test.asif 
```

## Testing Write

```
diskutil image attach --noMount  test.asif
echo -n 1 | dd of=/dev/disk27 bs=1 count=1
hdiutil eject /dev/disk27
```

## References

- Python implementation by Fox-IT:
    - [asif.py](https://github.com/fox-it/dissect.hypervisor/blob/main/dissect/hypervisor/disk/asif.py)
    - [c_asif.py](https://github.com/fox-it/dissect.hypervisor/blob/main/dissect/hypervisor/disk/c_asif.py)
