# ASIF binary specification

With macOS 26, Apple introduced the Apple Sparse Image Format (ASIF) for virtual disks.

As usual, there are no technical details provided about the layout of the ASIF file.

This repository collects reverse-engineered technical details and community-contributed findings about the Apple Sparse Image Format ASIF, for the benefit of forensics engineers, virtualization tool authors, archivists, and researchers.

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

