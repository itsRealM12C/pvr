# PVR

Extract the image in a PVR file.

### Identifying the format

The first step was running the example `file` on the binary:

```
HDMI-1.pvr: PowerVR 3.0 texture: 1080 x 1920, ETC1
```

A PVR3 file starts with a 52-byte header followed by optional metadata, then raw texture data. The header layout:

| Offset | Size | Name |
|--------|------|-------|
| 0 | 4 | Magic (`0x03525650`) |
| 4 | 4 | Flags |
| 8 | 8 | Pixel format (lo + hi u32) |
| 16 | 4 | Colour space |
| 20 | 4 | Channel type |
| 24 | 4 | Height |
| 28 | 4 | Width |
| 32 | 4 | Depth |
| 36 | 4 | Num surfaces |
| 40 | 4 | Num faces |
| 44 | 4 | Mip-map count |
| 48 | 4 | Metadata size |
| 52 | N | Metadata block |
| 52+N | — | Texture data |

Parsing the header for `HDMI-1.pvr`:

- **Dimensions:** 1920 × 1080
- **Pixel format:** `6` → ETC1
- **Mip levels:** 1
- **Metadata size:** 15 bytes
- **Texture data:** 1,036,800 bytes

### Expected vs actual colour format

The initial assumption was that the texture would be greyscale (B&W) — a common guess for embedded/captured textures used as UI elements or masks. It turned out to be **ETC1**, which is a full RGB compressed format.

## ETC1 — How It Works

ETC1 (Ericsson Texture Compression 1) is a lossy block compression format designed for mobile GPUs. It stores RGB colour but no alpha channel.

### Block structure

The image is divided into **4×4 pixel blocks**, each encoded in exactly **8 bytes** (64 bits). For a 1920×1080 image:

```
blocks wide  = ceil(1920 / 4) = 480
blocks tall  = ceil(1080 / 4) = 270
total blocks = 480 × 270     = 129,600
total bytes  = 129,600 × 8   = 1,036,800  ✓
```

### Encoding scheme (differential mode)

Each block is split into two **2×4 or 4×2 sub-blocks** (controlled by the flip bit). Each sub-block gets:

- A **base colour** (5-bit R/G/B in differential mode, shared between sub-blocks with a 3-bit signed delta)
- A **modifier table index** (3 bits, selects one of 8 modifier tables)

Each pixel then gets a **2-bit index** into the modifier table, which adds a luminance offset to the base colour:

```
modifier tables (8 total, 4 values each):
  [±2, ±8]   [±5, ±17]   [±9, ±29]   [±13, ±42]
  [±18, ±60] [±24, ±80]  [±33, ±106] [±47, ±183]
```

Final pixel value: `clamp(base_channel + modifier, 0, 255)` applied to R, G, and B equally — which is why it was mistaken for greyscale. The *modifier* is a luminance shift, but the *base colour* is full RGB.

### Decoding a block

```
word0 (big-endian u32): colour + flags
word1 (big-endian u32): 2-bit pixel indices (16 pixels × 2 bits)

diff_bit  = (word0 >> 1) & 1
flip_bit  = (word0 >> 0) & 1

if diff_bit:
    R1 = (word0 >> 27) & 0x1F  →  expand to 8-bit: (R1 << 3) | (R1 >> 2)
    dR = (word0 >> 24) & 0x7   →  sign-extend from 3 bits
    R2 = clamp(R1 + dR)        →  expand same way
    (same for G, B)
else:
    R1 = (word0 >> 28) & 0xF   →  expand: (R1 << 4) | R1
    R2 = (word0 >> 24) & 0xF   →  expand same way
    (same for G, B)

table1 = (word0 >> 5) & 0x7
table2 = (word0 >> 2) & 0x7

for each pixel i in 0..15:
    col = i >> 2,  row = i & 3
    sub_block = flip_bit ? (row < 2 ? 0 : 1) : (col < 2 ? 0 : 1)
    base  = sub_block == 0 ? (R1,G1,B1) : (R2,G2,B2)
    table = sub_block == 0 ? table1 : table2
    msb   = (word1 >> (i + 16)) & 1
    lsb   = (word1 >> (i +  0)) & 1
    mod   = modifier_table[table][(msb << 1) | lsb]
    pixel = clamp(base + mod)  ← applied to each channel
```

> **Why it expected to be greyscale:** the modifier is applied identically to R, G, and B. If the base colour happened to be near-neutral grey, the output looks monochrome even though the format is RGB.

## Decoding

A pure Python ETC1 decoder was written to validate the approach before building the browser tool.

```python
import struct, math
from PIL import Image

modifiers = [
    [2, 8, -2, -8],   [5, 17, -5, -17],
    [9, 29, -9, -29],  [13, 42, -13, -42],
    [18, 60, -18, -60],[24, 80, -24, -80],
    [33, 106,-33,-106],[47, 183,-47,-183],
]

def clamp(v): return max(0, min(255, v))

def decode_etc1_block(block_bytes, px_out, bx, by, width, height):
    w0, w1 = struct.unpack('>2I', block_bytes)
    diff = (w0 >> 1) & 1
    flip = w0 & 1

    if diff:
        r1 = (w0>>27)&31; dr=(w0>>24)&7
        if dr>=4: dr-=8
        r2 = r1+dr
        g1 = (w0>>19)&31; dg=(w0>>16)&7
        if dg>=4: dg-=8
        g2 = g1+dg
        b1 = (w0>>11)&31; db=(w0>>8)&7
        if db>=4: db-=8
        b2 = b1+db
        r1=(r1<<3)|(r1>>2); r2=clamp((r2<<3)|(r2>>2))
        g1=(g1<<3)|(g1>>2); g2=clamp((g2<<3)|(g2>>2))
        b1=(b1<<3)|(b1>>2); b2=clamp((b2<<3)|(b2>>2))
    else:
        r1=(w0>>28)&15; r1=(r1<<4)|r1; r2=(w0>>24)&15; r2=(r2<<4)|r2
        g1=(w0>>20)&15; g1=(g1<<4)|g1; g2=(w0>>16)&15; g2=(g2<<4)|g2
        b1=(w0>>12)&15; b1=(b1<<4)|b1; b2=(w0>>8) &15; b2=(b2<<4)|b2

    t1, t2 = (w0>>5)&7, (w0>>2)&7

    for i in range(16):
        col, row = i>>2, i&3
        sub = (row<2) if flip else (col<2)
        base = (r1,g1,b1) if sub else (r2,g2,b2)
        tbl  = t1 if sub else t2
        bit  = col*4+row
        mod  = modifiers[tbl][((w1>>(bit+16))&1)<<1|((w1>>bit)&1)]
        ox, oy = bx*4+col, by*4+row
        if ox < width and oy < height:
            idx = (oy*width+ox)*3
            px_out[idx]   = clamp(base[0]+mod)
            px_out[idx+1] = clamp(base[1]+mod)
            px_out[idx+2] = clamp(base[2]+mod)

with open('HDMI-1.pvr','rb') as f:
    data = f.read()

width, height = 1920, 1080
header_size = 52 + 15  # 67 bytes
etc_data = data[header_size:]

bw, bh = math.ceil(width/4), math.ceil(height/4)
px = bytearray(width * height * 3)

for i in range(bw * bh):
    decode_etc1_block(etc_data[i*8:(i+1)*8], px, i%bw, i//bw, width, height)

Image.frombytes('RGB',(width,height),bytes(px)).save('HDMI-1.png')
```

Output: a correct full-colour 1920×1080 PNG.

## Tool

A standalone browser-based extractor was built as a single HTML file. No server, no upload — decoding runs entirely client-side using the Web Canvas API.

**Features:**

- Drag-and-drop or file picker
- PVR3 header parser with validation log (magic check, format ID, block count, data size)
- ETC1 (fmt 6), ETC2 RGB (fmt 22), and ETC2 RGBA (fmt 23) decoding
- Live progress bar (yields to the browser every 2,000 blocks to stay responsive)
- Preview with checkerboard background (shows alpha for ETC2 RGBA)
- Export as PNG or JPG (with quality slider)

**Key implementation note:** the decode loop must `await` a `setTimeout(0)` on every chunk, not just occasionally — otherwise the browser never gets a chance to repaint and the result never appears, even though decoding completed successfully.

## Result

| Property | Value |
|----------|-------|
| Input | `HDMI-1.pvr` (1,012 KB) |
| Format | ETC1 / PVR3 |
| Dimensions | 1920 × 1080 |
| Blocks decoded | 129,600 |
| Output | Full-colour RGB PNG |
| Surprise | Expected greyscale, got full colour |
