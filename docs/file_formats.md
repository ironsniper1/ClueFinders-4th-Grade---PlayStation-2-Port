# File Formats

## RSC Container
NE (New Executable) format used as a resource container.
- Resource types: 0x800F (directory), 0xFF01 (ASEQ), 0xFF02 (RRGB/WAVE), 0xFF03 (WAVE)
- Directory entries: 12 bytes each (size, flags, id, type_string)
- Type strings: ASEQ (sprites), WAVE (audio), NFNT (fonts), RRGB (palettes)

## RRGB Palette
256 entries × 6 bytes = 1536 bytes per palette.
Format: R16, G16, B16 where each 16-bit value has the same byte doubled.

## ASEQ Sprite (Single Frame)
- Header: 14 bytes (version, frames=1, bpp=8, flags)
- Sub-header with color/palette data
- `FF FF 00 00 04 00` marker + width(u16) + height(u16)
- RLE data: 0xFF=run (color, count), 0x00=transparent (count, 0x00=EOL), else=literal

## ASEQ Sprite (Multi Frame)
- Header: version, frame_count, bpp=8, flags
- Frame offset table: N entries × 4 bytes (uint32)
- Frame 0: full sub-header + FF FF 00 00 marker + W/H + RLE
- Frames 1..N-2: RLE at frame_offsets[0..N-3], same W/H as frame 0
- frame_offsets[N-1]: sentinel value

## NFNT Font
Modified Macintosh NFNT bitmap format stored little-endian.

## Audio
Standard WAV: 22050 Hz, 8-bit, mono PCM inside RSC containers.

## Video
RAD Game Tools Smacker (.SMK) format.

## RSC File Naming
- `xxxP.RSC` = Primary (background + palette)
- `xxxi.RSC` / `xxxi1-3.RSC` = Interactive layers
- `xxxo.RSC` / `xxxo1-3.RSC` = Overlay layers
