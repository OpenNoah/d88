# Software

## u-boot

### u-boot load and relocation offset

Looking at qemu access log, u-boot-spl loads NAND data starting from page 8 (offset `0x8000`). And u-boot-spl jumps to `0x80100000` after it finishes. Now we know actual u-boot data offset and load address, for loading into ghidra.

u-boot source code is available from IRIVER open source page, even compiled elf executable and binaries are available. However, it doesn't exactly match the data dumped from NAND. It is still pretty helpful for checking some function and symbol names.

After u-boot loaded and started at `0x80100000`, it then relocates itself to the end of SDRAM space, which starts from `0x83e1c000`. Just add offset of `0x03e0c000` to any u-boot address in ghidra to calculate the address after relocation.

u-boot makes uses of global offset table (GOT), so all pointers can be relocated relatively easily. GOT is updated immediately after jumped to relocated code.

### u-boot GOT bug

While implementing qemu emulation, I found that u-boot tries to access out-of-range address `0x87d460a4`. (Valid SDRAM region is `0x80000000` to `0x84000000`.)

This access comes from: \
`cmd_tbl_t *find_cmd(const char *cmd);` \
It is supposed to be `strncmp()` accessing: \
`*((cmdtp = &__u_boot_cmd_start)->name)` \
Specifically, the GOT lookup of cmdtp to struct address `(cmdtp = &__u_boot_cmd_start)->name` is good, but dereferencing the `name` pointer in struct failed.

Correct address should be `0x83f3a0a4` (`0x8012e0a4` before relocation). Difference to the out-of-range address `0x87d460a4` is exactly one relocation offset `0x03e0c000`. So it seems, the relocation offset is applied to the address somehow.

The next struct data, `maxargs` also changed from `2` to `0x03e0c002`. Later data is ok, only 2x 4-byte words changed.

By monitoring write access to the pointer location `__u_boot_cmd_start->name` (address `0x83f80db8` or `0x80174db8` before relocation), I can see it has been updated with the relocation offset twice.

First time, it is updated by GOT relocation assembly code. This is actually quite strange, as this `char *name` pointer is technically not in the `.got` section. I checked the `.got` section, its size should be `0x018a` u32 words. However, the `num_got_entries` data before `in_ram` code is somehow set to `0x018c`, dumped from NAND. The command table starts immediately after `.got` section, this means the first 2x u32 words in the command table got erroneously updated with the relocation offset, i.e. `name` and `maxargs` of the first command, `autoscr`.

Second time, it is updated by a code specifically for relocating pointers in the command table. As the `char *` pointers in the command table are not handled by `.got` section relocation, they have to be done separately. This is as expected, but now the relocation offset has been applied to the first command `name` pointer twice.

It is quite strange how this bug got into the u-boot binary. Perhaps it was partially re-compiled, `.got` section changed, but startup assembly has not been rebuilt?

---

Relocation assembly code: \
`cpu/mips/start.S`

```as
        /* Jump to where we've relocated ourselves.
         */
        addi    t0, a2, in_ram - _start
        j       t0
        nop

        .word   uboot_end_data
        .word   uboot_end
        .word   num_got_entries

in_ram:
        /* Now we want to update GOT.
         */
        lw      t3, -4(t0)      /* t3 <-- num_got_entries       */
        addi    t4, gp, 8       /* Skipping first two entries.  */
        li      t2, 2
1:
        lw      t1, 0(t4)
        beqz    t1, 2f
        add     t1, t6
        sw      t1, 0(t4)
2:
        addi    t2, 1
        blt     t2, t3, 1b
        addi    t4, 4           /* delay slot                   */
```

Linker script GOT symbols: \
`board/cetus/u-boot.lds`

```ld
        _gp = ALIGN(16);

        __got_start = .;
        .got  : { *(.got) }
        __got_end = .;

        num_got_entries = (__got_end - __got_start) >> 2;
```

Command table specific relocation code: \
`lib_mips/board.c`

```c
        /*
         * We have to relocate the command table manually
         */
        for (cmdtp = &__u_boot_cmd_start; cmdtp !=  &__u_boot_cmd_end; cmdtp++) {
                ulong addr;

                addr = (ulong) (cmdtp->cmd) + gd->reloc_off;
#if 0
                printf ("Command \"%s\": 0x%08lx => 0x%08lx\n",
                                cmdtp->name, (ulong) (cmdtp->cmd), addr);
#endif
                cmdtp->cmd =
                        (int (*)(struct cmd_tbl_s *, int, int, char *[]))addr;

                addr = (ulong)(cmdtp->name) + gd->reloc_off;
                cmdtp->name = (char *)addr;
```
