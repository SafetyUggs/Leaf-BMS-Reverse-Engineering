# EEPROM layout and block checksums

## Memory map (EEPROM byte addresses)

| EEPROM range  | Size        | Contents                  |
|---------------|-------------|---------------------------|
| 0x000-0x001   | 2           | Header: ID block          |
| 0x002-0x003   | 2           | Header: DiagCal block     |
| 0x004-0x005   | 2           | Header: block A (main)    |
| 0x006-0x007   | 2           | Header: block B (DTC)     |
| 0x008-0x009   | 2           | Header: block A copy      |
| 0x00A-0x023   | 0x1A (26)   | ID block                  |
| 0x024-0x15F   | 0x13C (316) | DiagCal block             |
| 0x160-0x1F3   | 0x94 (148)  | Block A (main)            |
| 0x1F4-0x287   | 0x94 (148)  | Block A copy              |
| 0x288-0x289   | 2           | Header: block C (main)    |
| 0x28A-0x28B   | 2           | Header: block C copy      |
| 0x28C-0x347   | 0xBC (188)  | Block C (main)            |
| 0x348-0x403   | 0xBC (188)  | Block C copy              |
| 0x404-0x6EB   | 0x2E8 (744) | Block B (DTC)             |

The map is contiguous from 0x000 to 0x6EB with no gaps.

## Header word format

Each header is 16 bits, stored low byte first (EEPROM[addr] is the low byte).

| Bits  | Meaning                                    |
|-------|--------------------------------------------|
| 0-5   | Marker, must be `0x1A`                     |
| 6-15  | 10-bit block sum (`sum10`, see below)      |

So a valid header is `hdr = (sum10 << 6) | 0x1A`.

## Checksum algorithm

```
sum10 = (sum of all bytes of the block + 1) & 0x3FF
```

It is a plain unsigned byte sum over the whole block, plus 1, kept to 10 bits.

| Block   | Header at | Data at | Bytes summed |
|---------|-----------|---------|--------------|
| ID      | 0x000     | 0x00A   | 0x1A         |
| DiagCal | 0x002     | 0x024   | 0x13C        |
| A       | 0x004     | 0x160   | 0x94         |
| A copy  | 0x008     | 0x1F4   | 0x94         |
| C       | 0x288     | 0x28C   | 0xBC         |
| C copy  | 0x28A     | 0x348   | 0xBC         |
| B       | 0x006     | 0x404   | 0x2E8        |

A block is valid only when its header has the marker `0x1A` and the header's sum matches.
There is no checksum over the whole EEPROM: each block is checked on its own.

Blocks A and C have a backup copy: if the main copy fails, the firmware uses the copy. If both
fail, the block is treated as bad. Blocks B, DiagCal and ID have no copy.

## Writing a valid block

1. Write the data bytes.
2. Compute `s = (sum(bytes) + 1) & 0x3FF`.
3. Write the header `h = (s << 6) | 0x1A` low byte first: `EEPROM[hdr] = h & 0xFF`,
   `EEPROM[hdr+1] = h >> 8`.

## Open points
- The field layout inside blocks A, B, C, DiagCal and ID is not covered here.
