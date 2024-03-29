# Hardware

## Overview

Hardware | Part # | Description | Documentation
---|---|---|---
SoC | Ingenic JZ4755 | 336MHz XBurst-1 (mips32r1) 32-bit | [DS](datasheet/JZ4755_DS.pdf) / [PM](datasheet/JZ4755_pm.pdf) / [Core](datasheet/XBurst1_Core_PM.pdf) / [MXU](datasheet/XBurst1_ISA_MXU2.pdf)
SDRAM | A3V56S40ETP-G6 | 64MiB: 2 chips x 4 banks x 8192 rows x 512 columns x 16 bits | [DS](datasheet/A3V56S40ETP-G6.pdf)
NAND | K9LBG08U0D | 4GiB: 2 planes x 4096 blocks x 128 pages x (4k + 218) bytes | [DS](datasheet/K9LBG-08U0D.pdf)
LCD | LMS430HF15 | 480x272 24-bit parallel RGB | [DS](datasheet/LMS430HF15_Samsung.pdf)

## Interfaces

SoC pin locations and important test pads are annotated here: \
[SoC pins and test pads](images/SoC.svg)

## I2C devices

These I2C addresses respond with ACKs:

```text
I2C scan:
-- 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
00 00 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
10 10 -- -- -- -- -- -- -- -- -- -- 1b -- -- -- --
20 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40 -- -- 42 -- -- -- -- -- -- -- -- -- -- -- -- --
50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70 -- -- -- -- -- -- -- -- 78 -- -- -- -- -- -- --
```

Address | Part # | Description | Documentation
---|---|---|---
0x00 | - | General call address | -
0x10 | AR1010 | FM radio | [DS](datasheet/AR1000-AIROHA.pdf) / [PG](datasheet/ar1000F_progguide-0.81.pdf)
0x1b | WM8731 | Audio codec | [DS](datasheet/WolfsonWM8731.pdf)
0x42 | STMPE2403 | GPIO/keypad controller | [DS](datasheet/STMPE2403TBR.pdf)
0x78 | - | Reserved for 10-bit addressing | -

## Keypad IO mapping

The main keypad is connected to the STMPE2403 controller.

Row and column GPIO mapping are annotated here: \
[Keyboard IO mapping](images/Keyboard.svg)
