# ESPHome patches for IT8951 e-paper displays

Author note: Generated using Claude Opus 5.5.

Patches for ESPHome that make IT8951 e-paper panels faster, lighter on memory
and correct in their grey levels. Tested on the Seeed Studio reTerminal E1003
only; other IT8951 panels should work but are untested.

- **Much faster drawing**, so the device stays responsive.
- **Images load about 4x faster** on 2026.9 and 2026.10. 2026.11 includes most
  of this speedup without the patches.
- **Half the memory per image** compared with 8-bit `GRAYSCALE`, thanks to a new
  4-bit greyscale type (`type: GRAYSCALE_4BIT`).
- **Correct light greys.** Stock ESPHome renders light greys one step too
  light and shows the lightest grey as white.

## Which patches for which ESPHome version

Checked against 2026.9.0, 2026.9.1, 2026.10.2 and 2026.11.0b4.

| Patch | What it does | 2026.9 | 2026.10 | 2026.11.0b4 |
|---|---|---|---|---|
| 01 | Display hook for fast image drawing | needed | needed | needed |
| 02 | 4-bit greyscale image type | needed | needed | needed, redo |
| 03 | Faster image drawing, 4-bit support | needed | needed | needed, redo |
| 04 | Lets you use `GRAYSCALE_4BIT` in YAML | needed | needed | needed, redo |
| 05 | Faster image decoding (header) | needed | needed | **built in** |
| 06 | Faster image decoding | needed | needed | **built in** |
| 07 | Faster greyscale conversion, 4-bit support | needed | needed | needed, redo |
| 08 | IT8951: grey-level fix, fast drawing, timing log | needed | needed, one part built in | needed, redo |
| 09 | IT8951: header for patch 08 | needed | needed | needed, redo |

- **needed**: apply the patch file as it is.
- **built in**: ESPHome now does this itself. Skip the patch.
- **needed, redo**: still needed, but the patch file no longer fits the new
  ESPHome code and has to be rewritten.

**On 2026.10**, one part of patch 08 (a watchdog tweak) is already in ESPHome.
`patch` reports `Hunk #8 FAILED` and `1 out of 9 hunks FAILED` for it. That
is expected; the rest of patch 08 is applied. Any other failure means the
patches do not fit your version; undo them.

**2026.11 is not released yet** (checked against beta 2026.11.0b4), and there
is no working patch set for it. Stay on 2026.10 for now.

## How to apply

The patches change the installed ESPHome itself, not your project. That means
they need ESPHome installed with pip or uv (the command-line version). The
Home Assistant ESPHome add-on and the ESPHome Docker image bring their own copy
of ESPHome, which every update replaces.

1. Pin ESPHome to an exact version, for example
   `pip install esphome==2026.10.2`. **Every upgrade or reinstall removes the
   patches**, and you have to apply them again.
2. Activate the virtual environment ESPHome is installed in, then run this from
   the root of this repository (Git Bash, macOS or Linux):

   ```bash
   SITE=$(python -c "import esphome, os; print(os.path.dirname(os.path.dirname(esphome.__file__)))")
   for p in esphome-patches/*.patch; do patch -p1 -N -b -r - -d "$SITE" < "$p"; done
   ```

   `-b` keeps a `.orig` backup of every changed file. `-N` makes an accidental
   second run skip files that are already patched instead of stopping to ask,
   and `-r -` stops `patch` from leaving `.rej` files behind.
3. Check it worked: `esphome config your-device.yaml` must accept
   `type: GRAYSCALE_4BIT`. If it says the type is unknown, the patches are not
   applied.

**To undo**, reinstall the same version:
`pip install --force-reinstall --no-deps esphome==2026.10.2`.

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

- the image is not transparent (see the PNG tip below);
- the whole image is on the screen, and nothing clips it;
- its x position and its width are even numbers;
- the display is not rotated by 90 or 270 degrees;
- the display is in greyscale mode and `invert_colors` is off.

Patch 08 also raises the driver's `Update took N ms` log line to INFO, so you
can see the time of every repaint in the normal logs.

## Tips

- `psram_dma: true` on the display (shown above) makes sending each frame to
  the panel about a third faster and needs no patch.
- Use RGB PNGs, not greyscale ones, for images built into the firmware
  (`platform: file`). ESPHome treats greyscale PNGs as transparent, and
  transparent images take the slow path. Images downloaded with
  `online_image` are not affected. To convert one in place (Pillow comes with
  ESPHome):

  ```bash
  python -c "import sys; from PIL import Image; Image.open(sys.argv[1]).convert('RGB').save(sys.argv[1])" icon.png
  ```

  Skip images that really are transparent; converting them drops the
  transparency.
- The 4-bit type does not work with LVGL.

## Porting to 2026.11

For anyone rewriting the patches. In 2026.11.0b4 ESPHome already has the
faster decoder (05, 06), row-by-row image drawing (the loop order in 03),
integer greyscale conversion (07) and the watchdog change (08). What is still
missing and has to be carried over:

- the `GRAYSCALE_4BIT` type: 02, 04, and the 4-bit parts of 03 and 07;
- the fast drawing path: 01, the hook call in 03, the blit in 08 and its
  declaration in 09;
- the grey-level fix in 08 (`quantize_8bit_to_nibble()` divides by 17, not 16);
- optionally the timing log in 08 and 09.

## License

GPLv3, the same license as ESPHome's C++ code, which these patches modify.
