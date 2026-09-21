# .ndz Format Spec

2026-09-19

## File layout

A .ndz is a .nds compressed in 8 KiB blocks so the cart can decode any block on demand. Four regions, in this order. All integers are little-endian.

| Region | Size | Holds |
| --- | --- | --- |
| Header | 16384 B | identity fields and the banner copy |
| Raw dictionary | value at 0x2414, often 0 | verbatim bytes, present only when flag bit 5 is set |
| Frames | sum of the frame table's csize column | 128 KiB of rom each (the last one shorter), as a block index plus zstd blocks |
| Trailer | 8 B per frame + 8 B | frame table, frame count, magic |

A reader takes the trailer first, walks the frame table to place every frame, then reads a frame's block index to place every block. Nothing in the file points forward except the sizes.

## Header

Fixed 16384 bytes. Game code, original size and header CRC together name the exact dump the file was made from. Bytes not listed are zero.

| Offset | Size | Field |
| --- | --- | --- |
| 0x0000 | 4 | magic "NDZ1", 0x315A444E |
| 0x0004 | 4 | header size, 16384 |
| 0x0008 | 4 | original .nds size |
| 0x000C | 4 | flags, see the next table |
| 0x0010 | up to 0x2400 | banner copied from the .nds; 0x840 B, 0x940 B for v2, 0xA40 B for v3, 0x23C0 B for v0103 |
| 0x2410 | 4 | game code, bytes 0x0C..0x0F of the .nds |
| 0x2414 | 4 | dictionary bytes stored in the file |
| 0x2418 | 2 | header CRC16, bytes 0x15E..0x15F of the .nds |
| 0x2428 | 4 | dictionary bytes after decoding; equals 0x2414 for a raw dictionary |
| 0x2430 | 32 | delta only: BLAKE2b-256 of the base .nds |
| 0x2450 | 4 | delta only: base .nds size |
| 0x2454 | 2 | delta only: base header CRC16 |
| 0x2458 | 4 | delta only: options word for the DS side, ignored by the cart |
| 0x2460 | 64 | delta only: hack name, UTF-8, zero padded |
| 0x24A0 | 32 | delta only: hack version, UTF-8, zero padded |

## Flags

The u32 at 0x000C. The studio sets bits 0, 1 and 3 on every file, bit 5 when a dictionary paid off, bits 4 and 6 on a delta.

| Bit | Meaning |
| --- | --- |
| 0 | v2 layout: each frame begins with a block index. Required. |
| 1 | blocks are zstd frames. Required, the only codec. |
| 2 | trained dictionary. Retired, readers reject the file. |
| 3 | one mode byte per block follows the block index |
| 4 | delta: the file is a layer over a base .ndz |
| 5 | a raw dictionary sits right after the header |
| 6 | delta: stored blocks were compressed against the base's dictionary |
| 8..15 | log2 of the block size. 0 means 4096. The studio writes 13 (8 KiB); the cart accepts 512 B to 32 KiB. |

## Frames and blocks

The rom is cut into 128 KiB frames and each frame into blocks of the flagged size. The last frame, and the last block of a frame, are shorter. Frames sit back to back after the dictionary, in rom order.

| Frame body | Size | Content |
| --- | --- | --- |
| block index | 4 B per block | u32 compressed size of each block, in order |
| mode bytes | 1 B per block, only with flag bit 3 | the block's mode, see Block modes |
| blocks | sum of the index | block payloads back to back |

Blocks per frame = ceil(frame dsize / block size).

| Trailer | Size |
| --- | --- |
| frame table | 8 B per frame: u32 csize, u32 dsize |
| frame count | u32 |
| magic | u32 "LZ4B", 0x4C5A3442 |

Zstd blocks are single frames written at level 19 with no content size, no checksum and no dictionary id. A plain block's window is the block size. A dictionary block may reference the whole dictionary, so the decoder's window limit has to cover dictionary plus block; the cart uses log2 of that sum, 15 at minimum.

## Block modes

The mode byte says how a block payload turns back into rom bytes. Modes 2 to 6 run a byte filter before zstd; the packer tries every candidate on a block and keeps the smallest.

| Mode | Payload | Decode |
| --- | --- | --- |
| 0 | zstd frame | decompress with the raw dictionary |
| 1 | zstd frame | decompress |
| 2 | zstd frame | decompress, undo byte delta, stride 1 |
| 3 | zstd frame | decompress, undo byte delta, stride 2 |
| 4 | zstd frame | decompress, undo byte delta, stride 4 |
| 5 | zstd frame | decompress, undo byte shuffle, stride 2 |
| 6 | zstd frame | decompress, undo byte shuffle, stride 4 |
| 7 | u32 base offset, 4 B | delta only: copy the block from the base rom at that offset |

Byte delta with stride s stores out[i] = in[i] - in[i - s] for i >= s; undo it front to back by adding in[i - s] back. Byte shuffle with stride s writes all bytes at positions 0 mod s, then 1 mod s, and so on; undo it by interleaving. Filters are only tried on full-size blocks.

## Raw dictionary

An optional slab of the rom's own content, stored verbatim right after the header and used as a zstd raw-content dictionary by mode 0 blocks. Flag bit 5 marks it; 0x2414 and 0x2428 both carry its size. The packer picks it from a duplicate census of the rom (content-defined chunks ranked by how often they repeat) and keeps it only if at least 4 KiB came out. The cart pins the dictionary in PSRAM for as long as the file is open, so its size is capped at PSRAM minus a 512 KiB block cache.

## Delta files

A rom hack ships as a delta .ndz: the patched rom packed against the base game's .ndz, so the card holds one copy of the game and a small file per hack. The packer's input is the patched rom, however it was made (an .xdelta applied to the base, or a pre-patched dump), plus the base rom. On the cart the base is opened first and the delta layers over it.

| Rule | Value |
| --- | --- |
| flags | bits 0, 1, 3 and 4; bit 6 when the base has a dictionary |
| block size | same as the base |
| dictionary | none of its own; mode 0 blocks use the base's |
| identity | 0x2450 and 0x2454 must equal the base's original size and header CRC; a base packed before the CRC field reads 0 and matches on size alone |
| banner, game code | the hack's |

Every block of the patched rom that exists byte for byte somewhere in the base becomes a mode 7 block: four bytes naming the base offset. Matches come from the two roms' file allocation tables first (a repacked NitroFS shifts whole files), then from a sampled hash index over the base, and each one is verified by comparing the bytes. Everything else is compressed like a normal block. Before writing, the packer rebuilds the rom from base plus delta and compares it with its input.

The options word at 0x2458 is read by the DS side and ignored by the cart.

| Bit | Meaning |
| --- | --- |
| 0 | run the hack on the base game's binaries. Retired: the loader still honours it, nothing sets it. |
| 1 | boot in NTR (DS) mode although the base is DSi-enhanced |

## xdelta patches

The studio makes and applies .xdelta files with xdelta3 3.2.0 built to WebAssembly from the upstream C source, so a patch it writes is byte for byte what xdelta3 on the desktop writes for the same settings. Making a patch takes a base rom and a target rom and produces VCDIFF (RFC 3284) with these choices.

| Option | Values | Default |
| --- | --- | --- |
| secondary compression | none, djw | djw is smaller; none for patchers that reject any secondary compression, such as Rom Patcher JS |
| compression level | 1, 3, 6, 9 | 3 |
| target checksum | adler32 on or off | on |
| application header | target//source/ | as xdelta3 writes it |

The studio does not produce LZMA secondary compression, but it applies patches that use it: decoding covers none, djw and LZMA. A hack normally arrives as base .nds plus .xdelta; the studio applies the patch, then packs the result as a delta .ndz. The .xdelta itself never goes on the card.

## DSi-enhanced hacks

In TWL mode the game's card driver checks every read against the cart's digest tables: a SHA1-HMAC per 0x400-byte sector, blocks of 32 sector hashes, and a master HMAC over the block table in the header. A byte patch leaves the base's tables behind, so the first verified read of changed data white-screens. What the hack ships decides how it is packed.

| arm9i and arm7i in the patched rom | Packing |
| --- | --- |
| fill bytes (the usual case: the patch tool left them empty) | untouched, option bit 1 set, boots in NTR mode |
| present, tables verify | untouched, boots in TWL mode |
| present, tables do not verify | tables recomputed over the hack's own data with the key every DSi cart carries, boots in TWL mode |

Two stretches are hashed by Nintendo in a form a dump does not hold, the first 2 KiB of the secure area and the modcrypt area of arm9i. Their entries cannot be recomputed; where the hack keeps the base's bytes there, it keeps the base's entries. The launcher can flip a hack between NTR and TWL afterwards by rewriting option bit 1 in place.

## Reading a .ndz on the cart

The cart never inflates the file. It decodes one block at a time into a PSRAM cache and serves 512-byte sectors of the decompressed image out of that cache, so the game sees a plain rom.

| Command | Purpose |
| --- | --- |
| E6 | open: payload is the file's FAT cluster map and size, so the cart reads the .ndz straight off the SD card |
| EC | poll the open: 0 busy, 1 ready, 2 and up an error code |
| E7 | close |
| E3 with bit 32 set | request 512 bytes at the given offset into the decompressed image, in 512-byte units |
| E4, E5 | poll ready, fetch the 512 bytes, as for a raw .nds |

Open parses the header, checks the flags, pins the dictionary in PSRAM, reads the trailer and frame table, and pre-warms the block cache so the game's first overlay loads do not all miss at once. Opening a delta while a base is open keeps the base's blocks in the cache and starts serving the hack; a base open replaces whatever was there.
