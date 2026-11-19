# ESPHome patches for IT8951 e-paper displays

Author note: Generated using Claude Opus 5.5.

Patches for ESPHome that make IT8951 e-paper panels render faster. Tested on
the Seeed Studio reTerminal E1003 only; other IT8951 panels should work but
are untested.

- **Much faster image drawing**: a full-screen 1872x1404 image goes into the
  framebuffer in about 62 ms instead of about 3.4 s, so the device stays
  responsive.
- **Faster sending to the panel**: 29 ms less per full frame.
- **Faster loading of downloaded images**: about 8 % less decode time.
- **Half the memory per image** compared with 8-bit `GRAYSCALE`, thanks to a new
  4-bit greyscale type (`type: GRAYSCALE_4BIT`). The fast drawing path needs it.

## Which patches for which ESPHome version

| ESPHome | Folder |
|---|---|
| 2026.11.0 | [`esphome-patches/2026.11/`](esphome-patches/2026.11/) |
| 2026.9.x, 2026.10.x | [`esphome-patches/2026.9-2026.10/`](esphome-patches/2026.9-2026.10/), see [below](#esphome-20269-and-202610) |

The patches only apply to the exact versions listed. Each folder is a complete
set: apply every patch in it, and nothing from the other folder.

## The 2026.11 patches

Checked against the 2026.11.0 release. The numbers match the older set, so
05 and 06 are missing on purpose: ESPHome 2026.11 does that work itself.

| Patch | What it does |
|---|---|
| 01 | Display hook for the fast drawing path |
| 02 | 4-bit greyscale image type |
| 03 | Draws 4-bit images, and hands whole images to the fast path |
| 04 | Lets you use `GRAYSCALE_4BIT` in YAML. Also makes greyscale PNGs load as opaque: stock ESPHome treats them as transparent, which rules out the fast path |
| 07 | Writes downloaded images (`online_image`) in 4-bit |
| 08 | IT8951: the fast drawing path, and sending four rows per SPI transfer instead of one |
| 09 | IT8951: header for patch 08 |
| 10 | Faster decoding of downloaded 4-bit images |

Measured on the reTerminal E1003, full-screen GC16 repaint:

| | without | with |
|---|---|---|
| Drawing a 1872x1404 page image (01-04, 08) | ~3,400 ms, pixel by pixel | **62 ms** |
| Sending the frame to the panel (08), with `psram_dma: true` | 572 ms, one row per transfer | **543 ms** |
| Downloading and decoding a 1872x1404 4-bit PNG (10) | 4.73 s | **4.35 s** |

Unlike the older set, this set contains only speed-ups. It leaves out:

- **The light-grey fix.** Stock ESPHome maps 8-bit grey to the panel's 16
  levels by dividing by 16 instead of 17, so greys from the middle up render
  one step too light and the lightest grey comes out white. 4-bit images
  drawn through the fast path are not affected, since they are copied as they
  are; everything else (text, shapes, 8-bit images) is. The fix is one line,
  the body of `quantize_8bit_to_nibble()` in `it8951/it8951.cpp`:
  `return static_cast<uint8_t>((static_cast<uint32_t>(value) * 241u + 2048u) >> 12);`
- **The timing log.** The driver's `Update took N ms` line stays at VERBOSE.

## How to apply

The patches change the installed ESPHome itself, not your project. That means
they need ESPHome installed with pip or uv (the command-line version). The
Home Assistant ESPHome add-on and the ESPHome Docker image bring their own copy
of ESPHome, which every update replaces.

1. Pin ESPHome to an exact version, for example
   `pip install esphome==2026.11.0`. **Every upgrade or reinstall removes the
   patches**, and you have to apply them again.
2. Activate the virtual environment ESPHome is installed in, then run this from
   the root of this repository (Git Bash, macOS or Linux), with the folder for
   your version:

   ```bash
   PATCHES=esphome-patches/2026.11
   SITE=$(python -c "import esphome, os; print(os.path.dirname(os.path.dirname(esphome.__file__)))")
   for p in "$PATCHES"/*.patch; do patch -p1 -N -b -r - -d "$SITE" < "$p"; done
   ```

   `-b` keeps a `.orig` backup of every changed file. `-N` makes an accidental
   second run skip files that are already patched instead of stopping to ask,
   and `-r -` stops `patch` from leaving `.rej` files behind.
3. Check it worked: `esphome config your-device.yaml` must accept
   `type: GRAYSCALE_4BIT`. If it says the type is unknown, the patches are not
   applied.

**To undo**, reinstall the same version:
`pip install --force-reinstall --no-deps esphome==2026.11.0`.

## How to use

Set `type: GRAYSCALE_4BIT` on your images and draw them as usual:

```yaml
image:
  - platform: file
    file: images/icon.png
    id: my_icon
    type: GRAYSCALE_4BIT

  - platform: online_image
    url: http://example.local/page.png
    id: my_page
    format: PNG
    type: GRAYSCALE_4BIT

display:
  - platform: it8951
    psram_dma: true
    # ...
    lambda: |-
      it.image(0, 0, id(my_page));
      it.image(100, 200, id(my_icon));
```

The fast drawing path only applies when all of these hold. Otherwise the image
still draws correctly, just at the old speed:

- the image is not transparent;
- the whole image is on the screen, and nothing clips it;
- its x position and its width are even numbers;
- the display is not rotated by 90 or 270 degrees;
- the display is in greyscale mode and `invert_colors` is off.

It is fastest when x and the width are multiples of 8.

A lambda can also copy part of a 4-bit image with
`it.draw_gray4_packed(x, y, w, h, src, src_stride)`, where `src` points at the
first byte to copy (`image->get_data_start()` plus the offset) and
`src_stride` is `image->get_width_stride()`. It returns `false`, and draws
nothing, when the conditions above do not hold.

## Tips

- `psram_dma: true` on the display (shown above) makes sending each frame to
  the panel about a third faster and needs no patch.
- Downloaded images decode fastest when they arrive at the size the image is
  configured for (`resize:`, or no `resize:` at all), so serve them at the size
  you draw them.
- The 4-bit type does not work with LVGL.

## ESPHome 2026.9 and 2026.10

The older set in [`esphome-patches/2026.9-2026.10/`](esphome-patches/2026.9-2026.10/)
was checked against 2026.9.0, 2026.9.1 and 2026.10.2. Besides the speed-ups it
also has the light-grey fix and raises the driver's `Update took N ms` log line
to INFO. Its patches 05 and 06 make image decoding about 4x faster; 2026.11 has
that built in.

**On 2026.10**, one part of patch 08 (a watchdog tweak) is already in ESPHome.
`patch` reports `Hunk #8 FAILED` and `1 out of 9 hunks FAILED` for it. That
is expected; the rest of patch 08 is applied. Any other failure means the
patches do not fit your version; undo them.

With this set, use RGB PNGs, not greyscale ones, for images built into the
firmware (`platform: file`): it does not have the 2026.11 set's fix for
greyscale PNGs loading as transparent. Images downloaded with `online_image`
are not affected. To convert one in place (Pillow comes with ESPHome):

```bash
python -c "import sys; from PIL import Image; Image.open(sys.argv[1]).convert('RGB').save(sys.argv[1])" icon.png
```

Skip images that really are transparent; converting them drops the
transparency.

## License

GPLv3, the same license as ESPHome's C++ code, which these patches modify.
