# ASIF binary specification

With macOS 26, Apple introduced the Apple Sparse Image Format (ASIF) for virtual disks.

As usual, there are no technical details provided about the layout of the ASIF file.

This repository collects reverse-engineered technical details and community-contributed findings about the Apple Sparse Image Format ASIF, for the benefit of forensics engineers, virtualization tool authors, archivists, and researchers.

## Header

Integers stored in big-endian.

| Offset (hex) | Length (bytes) | Description                         |
|--------------|----------------|-------------------------------------|
| 0x00         | 4              | Magic 'shdw' (73 68 64 77)          |
| 0x04         | 4              | Version (ex.: 00 00 00 01)          |
| 0x08         | 4              | Header size (ex.: 00 00 02 00)      |
| 0x0C         | 4              | Flags                               |
| 0x10         | 8              | Offset to Directory                 |
| 0x18         | 8              | Offset to Directory                 |
| 0x20         | 16             | UUID                                |
| 0x30         | 8              | Sector count                        |
| 0x38         | 8              | Max Sector Count                    |
| 0x40         | 4              | Chunk size in bytes                 |
| 0x44         | 2              | Sector size  (block size)           |
| 0x46         | 2              | ??                                  |
| 0x48         | 8              | Metadata offset? (unit chunksize?)  |

## Directory

Directory with highest sequence number is active.

Let N = Count of data chunks per chunk group = 4 * sector size  
Let S = Chunk group size (in bytes) = 8 * (N + 1)  
Let C = Count of chunk groups per table = floor(chunk size / S)  
Let D = Data bytes addressed by a table = C * N * chunk size  
Let T = Count of tables = ceil(max sector count * sector size / D)

To determine table, divide offset by D and lookup in directory.

| Offset (hex) | Length (bytes) | Description                         |
|--------------|----------------|-------------------------------------|
| 0x00         | 8              | Sequence number                     |
| 0x08         | 8              | Chunk number containing Table 0     |
| 0x16         | 8              | Chunk number containing Table 1     |
...
| 8 * T        | 8              | Chunk number containing Table T-1   |

## Table

A Table consists of chunk groups.

| Offset (hex)   | Length (bytes) | Description                         |
|----------------|----------------|-------------------------------------|
| 0x00           | S              | Chunk group 0                       |
| S              | S              | Chunk group 1                       |
...
| S * (C - 1)    | S              | Chunk group C-1                     |

## Chunk group

A chunk group consists of
- N data chunk numbers
- 1 bitmap chunk number

| Offset (hex) | Length (bytes) | Description                         |
|--------------|----------------|-------------------------------------|
| 0x00         | 8              | Data chunk 0                        |
| 0x08         | 8              | Data chunk 1                        |
...
| 8 * (N - 1)  | 8              | Data chunk N-1                      |
| 8 * N        | 8              | Bitmap chunk                        |

First 9 MSB of chunk number are flags. Todo: describe.

## Creating test images

```
$ diskutil image create blank --format ASIF --size 123g -fs None test.asif
$ diskutil image info test.asif 
```

## Testing write

```
diskutil image attach --noMount  test.asif
echo -n 1 | dd of=/dev/disk27 bs=1 count=1
hdiutil eject /dev/disk27
```

## References

- Python implementation by Fox-IT:
    - [asif.py](https://github.com/fox-it/dissect.hypervisor/blob/main/dissect/hypervisor/disk/asif.py)
    - [c_asif.py](https://github.com/fox-it/dissect.hypervisor/blob/main/dissect/hypervisor/disk/c_asif.py)