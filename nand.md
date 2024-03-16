# NAND

## Partitions

NAND partition layout extracted from kernel disassembly.

Extracted files are from a working device, ECC corrected.

Name | MTD description | Format | Offset | Size | Extracted
---|---|---|---|---|---
u-boot | NAND BOOT partition | binary | 0 MiB | 4 MiB | [u-boot_trimmed.bin](https://github.com/OpenNoah/d88/raw/gh-pages/nand_dump/u-boot_trimmed.bin)
kernel | NAND KERNEL partition | binary | 4 MiB | 8 MiB | [kernel_trimmed.bin](https://github.com/OpenNoah/d88/raw/gh-pages/nand_dump/kernel_trimmed.bin)
rootfs | NAND ROOTFS partition | yaffs2 | 8 MiB | 248 MiB | [rootfs.tar.xz](https://github.com/OpenNoah/d88/raw/gh-pages/nand_dump/rootfs.tar.xz)
vfat3 | NAND vfat partition | vfat cp936 | 256 MiB | 1000 MiB | [vfat3.tar.xz](https://github.com/OpenNoah/d88/raw/gh-pages/nand_dump/vfat3.tar.xz)
vfat4 | NAND VFAT partition | vfat cp936 | 1256 MiB | 2712 MiB | vfat4.tar.gz
unused | - | - | 3968 MiB | 128 MiB | (only 0xff)

Full NAND dump: nand_dump_oob.bin.gz
