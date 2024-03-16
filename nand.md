# NAND

## Partitions

NAND partition layout extracted from kernel binary.

Extracted files are from a working device, ECC corrected.

Name | MTD description | Format | Offset | Size | Extracted
---|---|---|---|---|---
u-boot | NAND BOOT partition | binary | 0 MiB | 4 MiB | [u-boot_trimmed.bin](https://github.com/OpenNoah/d88/releases/download/d88/u-boot_trimmed.bin)
kernel | NAND KERNEL partition | binary | 4 MiB | 8 MiB | [kernel_trimmed.bin](https://github.com/OpenNoah/d88/releases/download/d88/kernel_trimmed.bin)
rootfs | NAND ROOTFS partition | yaffs2 | 8 MiB | 248 MiB | [rootfs.tar.xz](https://github.com/OpenNoah/d88/releases/download/d88/rootfs.tar.xz)
vfat3 | NAND vfat partition | mtdblock-jz vfat cp936 | 256 MiB | 1000 MiB | [vfat3.tar.xz](https://github.com/OpenNoah/d88/releases/download/d88/vfat3.tar.xz)
vfat4 | NAND VFAT partition | mtdblock-jz vfat cp936 | 1256 MiB | 2712 MiB | [vfat4.tar.gz](https://github.com/OpenNoah/d88/releases/download/d88/vfat4.tar.gz)
unused | - | - | 3968 MiB | 128 MiB | (only `0xff`)

Full NAND dump: [nand_dump_oob.bin.gz](https://github.com/OpenNoah/d88/releases/download/d88/nand_dump_oob.bin.gz)

Note: Although actual NAND OOB size is 218 bytes, the NAND dump code aligns it to 4-byte boundaries, so OOB size stored in the NAND dump is 220 bytes, with 2 extra bytes filled with `0xff`.

## ECC details

Depending on partition, ECC is located at different places and calculated based on different data.

ECC is calculated by JZ4755's hardware BCH ECC module, 8-bit mode. \
ECC block output is 13 bytes.

Type | First page (byte offset) | ECC OOB offset (bytes) | ECC data (block size)
---|---|---|---
u-boot-spl | `0x0000 (0x00000000)` | 3 | page data (512)
u-boot, kernel | `0x0004 (0x00004000)` | 24 | page data (512)
file systems | `0x0800 (0x00800000)` | 24 | page data (512) + OOB data (3)

For file systems, ECC protects OOB data as well, since OOB data may be used by the specific file system to store metadata. \
e.g. for yaffs2 tags, Ingenic special mtdblocks.

Each NAND page requires 4096 / 512 = 8 ECC blocks. \
When OOB data is also protected, the 24 OOB bytes before ECC offset gets divided to 3-byte chunks, appended to ECC encoder data input after page data.

Total number of ECC bytes for a NAND page is 13 * 8 = 104 bytes. \
OOB data usage is 24 + 104 = 128 bytes. \
The NAND has 218 bytes OOB, the remaining bytes appear unused.

The Ingenic special mtdblock regions (mtdblock-jz) may store OOB metadata when page data is all `0xff`, but do not calculate ECC parity bytes. i.e., OOB data does get protected in this case. \
When reading NAND data for these regions, need to skip ECC decoding if page data is all `0xff`.

## Extracting yaffs2 files

I found [devttys0/yaffshiv](https://github.com/devttys0/yaffshiv) works well with yaffs2 from NAND dump.

```bash
sudo yaffshiv -f rootfs_oob.bin -p 4096 -s 220 -n -o -d rootfs
```

## Ingenic special mtdblocks (mtdblock-jz)

The vfat3 and vfat4 regions are created based on Ingenic's special flavoured mtdblock implementation. \
Source code can be find in D8's kernel source, at \
`linux-2.6.24.3/drivers/mtd/mtdblock-jz.c`

### Allocation

mtdblock-jz blocks are allocated based on NAND erase blocks. \
An erase block is only allocated when accessed by upper layers. It may be mapped to any virtual address. \
A lifetime counter is implemented in each erase blocks for wear leveling. \
Erase blocks can be marked as bad blocks and skipped. \
A certain number of erase blocks can be reserved for wear leveling and hidden from total mtdblock size visible by upper layers. \
mtdblock-jz exposed virtual block size is only 512 bytes. It maintains some internal RAM cache to deal with the block size mismatch.

### OOB format

Example OOB data:

```text
PAGE 0x00098500 OOB:
 00000000  ff ff ff ff 80 08 00 00  80 08 00 00 00 00 00 00
 00000010  ff ff ff ff ff ff ff ff  51 8d eb d4 c8 39 e0 89
 00000020  52 e0 6d c9 09 71 8b 25  ac 7c 3d c1 a3 0c d4 b7
 00000030  9b ab cf 3f e0 78 de 5f  5e 78 1b 0a f6 d4 a4 28
 00000040  71 5c 7e 51 74 42 91 60  a9 95 e3 f2 12 89 80 99
 00000050  ba d9 c3 ac 8a 67 49 4a  34 16 1f f3 90 43 bd b9
 00000060  e9 3e 4a 48 80 b1 94 42  56 74 62 0b fa e4 c1 a4
 00000070  e1 59 4e 25 e1 c2 ad b8  ae 15 f7 de b1 7b db 07
```

First 4 bytes (`0x00`-`0x03` `0xffffffff`) is likely used by mtd tags. \
Next 4 bytes (`0x04`-`0x07` `0x00000880`) is virtual erase block address. \
Next 4 bytes (`0x08`-`0x11` `0x00000880`) is virtual erase block address repeated, for sanity checking. \
Last 4 bytes (`0x12`-`0x15` `0x00000000`) is erase block lifetime, i.e. how many times has this block been erased. \
Then ECC parity data starts from byte offset 24: `51 8d eb d4...`

### Badblock marker

From NAND datasheet:

> Samsung makes sure that the last page of every initial invalid block has non-FFh data at the column address of 4,096.

Which means, the first OOB byte of the last page in each erase block should be checked to identify and skip bad blocks.

This is implemented by `drivers/mtd/nand/nand_base.c`, see `NAND_LARGE_BADBLOCK_POS` in `include/linux/mtd/nand.h`.

NB. Apparently for small page size (<= 512 bytes) NAND, the badblock marker is located at offset 5 instead.

### Multi-plane operation

For D88, NAND multi-plane support is only enabled for these mtdblock-jz vfat partitions.

From NAND datasheet:

> Figure 2. K9LBG08U0D Array Organization \
> Column Address: A0 ~ A12 \
> Row Address: \
> Page Address: A13 ~ A19 \
> Plane Address: A20 \
> Block Address: A21 ~ the last Address

The NAND plane pages are paired by the plane address bit. \
Our NAND has 2 planes, e.g. page `0x1234` and page `0x12b4` are paired.

When reading/writing pages, both plane pages will be read/written in one operation. Similarly, when erasing blocks, both plane blocks must be erased at the same time. \
The page data is constructed using lower plane page first, followed by higher plane page data. \
Effectively, the NAND now has 2x page size (and oob size).

For mtdblock-jz, OOB layout and OOB data before ECC parity bytes should be identical on both plane pages. Only ECC parity bytes are different between plane page pairs.

For dumping NAND, we just need to concatenate page data from both planes, treat them as a single page. \
For example, NAND page reading order would be like: \
`0x1234 -> 0x12b4 -> 0x1235 -> 0x12b5 -> ...`

Note that also means there are 2 badblock markers for each plane block pair. \
Both markers need to be checked, and both plane blocks are unusable if either of them is a bad block.

## NAND dump extraction script

Full script code I used for extracting NAND dumps. \
Python scripts used can be found at (TODO) repo.

```bash
#!/bin/bash -ex
if=nand_dump_oob.bin

planes=2
erase_pages=128
page_size=4096
oob_size=220

block_size=$((page_size+oob_size))

mkdir -p extract

# offset 0, size 8 KiB: u-boot SPL
fname=extract/u-boot-spl
dd if=$if of=${fname}_oob.bin bs=$block_size count=2
./split_oob.py -p $page_size -s $oob_size ${fname}_oob.bin ${fname}.bin extract/oob.bin

# offset 8 KiB, size 8 KiB: u-boot SPL backup
fname=extract/u-boot-spl_backup
dd if=$if of=${fname}_oob.bin bs=$block_size skip=2 count=2
./split_oob.py -p $page_size -s $oob_size ${fname}_oob.bin ${fname}.bin extract/oob.bin

# offset 0, size 4 MiB: u-boot
fname=extract/u-boot
dd if=$if of=${fname}_oob.bin bs=$block_size count=$((4*1024*1024/$page_size))
./split_oob.py -p $page_size -s $oob_size ${fname}_oob.bin ${fname}.bin extract/oob.bin
./trim_end.py -a 4 -p 255 ${fname}.bin ${fname}_trimmed.bin

# offset 4 MiB, size 4 MiB: kernel
fname=extract/kernel
dd if=$if of=${fname}_oob.bin bs=$block_size skip=$((4*1024*1024/$page_size)) count=$((4*1024*1024/$page_size))
./split_oob.py -p $page_size -s $oob_size ${fname}_oob.bin ${fname}.bin extract/oob.bin
./trim_end.py -a 4 -p 255 ${fname}.bin ${fname}_trimmed.bin
tail -c+65 ${fname}_trimmed.bin | zcat > extract/vmlinux.bin

# offset 8 MiB, size 248 MiB: rootfs
fname=extract/rootfs
dd if=$if of=${fname}_oob.bin bs=$block_size skip=$((8*1024*1024/$page_size)) count=$((248*1024*1024/$page_size))

# https://github.com/zhiyb/yaffshiv
sudo $(which yaffshiv) -f ${fname}_oob.bin -p $page_size -s $oob_size -n -o -d $fname
#sudo yaffshiv -f ${fname}_oob.bin -p $page_size -s $oob_size -n -o -d $fname
sudo tar Jcvf ${fname}.tar.xz -C ${fname} .
sudo rm -rf ${fname}

# offset 256 MiB, size 1000 MiB: vfat3
fname=extract/vfat3
dd if=$if of=${fname}_oob.bin bs=$block_size skip=$((256*1024*1024/$page_size)) count=$((1000*1024*1024/$page_size))
./mtdblock_jz.py -l $planes -b $erase_pages -p $page_size -s $oob_size ${fname}_oob.bin ${fname}.bin
mkdir -p ${fname}
sudo mount -t vfat -o ro,codepage=936,iocharset=cp936 ${fname}.bin ${fname}
sudo tar Jcvf ${fname}.tar.xz -C ${fname} .
sudo umount ${fname}
rmdir ${fname}

# offset 1256 MiB, size 2712 MiB: vfat4
fname=extract/vfat4
dd if=$if of=${fname}_oob.bin bs=$block_size skip=$((1256*1024*1024/$page_size)) count=$((2712*1024*1024/$page_size))
./mtdblock_jz.py -l $planes -b $erase_pages -p $page_size -s $oob_size ${fname}_oob.bin ${fname}.bin
mkdir -p ${fname}
sudo mount -t vfat -o ro,codepage=936,iocharset=cp936 ${fname}.bin ${fname}
sudo tar zcvf ${fname}.tar.gz -C ${fname} .
sudo umount ${fname}
rmdir ${fname}

# offset 3968 MiB, size 128 MiB: unused
fname=extract/unused
dd if=$if of=${fname}_oob.bin bs=$block_size skip=$((3968*1024*1024/$page_size)) count=$((128*1024*1024/$page_size))
```
