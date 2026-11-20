# ESPHome patches for IT8951 e-paper displays

Author note: Generated using Claude Opus 5.5.

Patches for ESPHome that make IT8951 e-paper panels render faster. Tested on
the Seeed Studio reTerminal E1003 only; other IT8951 panels should work but
are untested.

- **Faster image drawing**: a full-screen 1872x1404 image goes into the
  framebuffer in 62 ms instead of 2.46 s, about 40x faster, because it is
  copied row by row instead of drawn pixel by pixel. ESPHome's main loop, and
  with it touch input, is blocked for that time.
- **Faster sending to the panel**: full-width updates go out four rows per SPI
  transfer instead of one, 543 ms instead of 572.5 ms per full frame (with
  `psram_dma: true`).
- **Faster loading of downloaded images**: about 8 % less decode time
  (4.73 s to 4.35 s for a 1872x1404 PNG) for an opaque 4-bit image decoded at
  its own size.
- **Half the memory per image** compared with 8-bit `GRAYSCALE`, thanks to a new
  4-bit greyscale type (`type: GRAYSCALE_4BIT`). The fast drawing path needs it.
- **Faster switching between full-screen pages** (2026.11 set, opt-in): the
  next page is loaded into the panel controller in the background, so a
  switch only sends what changed. 1.33 s to 0.83-0.87 s until the new page
  is on the glass. See [Preloading the next page](#preloading-the-next-page).

## Which patches for which ESPHome version

| ESPHome | Folder |
|---|---|
| 2026.11.0 | [`esphome-patches/2026.11/`](esphome-patches/2026.11/) |
| 2026.9.x, 2026.10.x | [`esphome-patches/2026.9-2026.10/`](esphome-patches/2026.9-2026.10/), see [below](#esphome-20269-and-202610) |

The patches only apply to the exact versions listed. Each folder is a complete
set: apply every patch in it (those named `.optional.patch` only if you want
them), and nothing from the other folder.

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
| 08 | IT8951: the fast drawing path, sending four rows per SPI transfer instead of one, and the [preload](#preloading-the-next-page) itself |
| 09 | IT8951: header for patch 08 |
| 10 | Faster decoding of downloaded 4-bit images |
| 11 | Optional (`.optional.patch`, skipped by the loop below): the [light-grey fix](#the-light-grey-fix) |
| 13 | IT8951: the `preload:` option for the [preload](#preloading-the-next-page) |

Measured on the reTerminal E1003, medians of 10 runs. Each row compares the
same firmware with and without that one change, except the page switch row,
explained [with the preload](#preloading-the-next-page):

| | without | with |
|---|---|---|
| Drawing a full-screen 1872x1404 4-bit image (01-04, 08) | 2,464 ms, pixel by pixel | **62 ms** |
| Sending a full frame to the panel (08), `psram_dma: true`, 20 MHz SPI | 572.5 ms, one row per transfer | **543 ms** |
| Downloading and decoding a 1872x1404 4-bit PNG (10) | 4.73 s | **4.35 s** |
| Switching between two full-screen pages, until the new one is on the glass (08, 13; 5 per direction) | 1,330-1,335 ms | **825-873 ms** |

The pixel-by-pixel figure is these patches' own 4-bit pixel path with the fast
path switched off: stock ESPHome cannot draw a 4-bit image at all.

Unlike the older set, the main set contains only speed-ups. The driver's
`Update took N ms` line stays at VERBOSE, and so does the preload's
`Preload rows ...` line.

### The light-grey fix

Optional patch 11, kept apart because it changes how your screen looks.
Stock ESPHome maps 8-bit grey to the panel's 16 levels by dividing by 16
instead of 17, so greys from the middle up render one step too light and the
lightest grey comes out white. 4-bit images drawn through the fast path are
not affected, since they are copied as they are; everything drawn pixel by
pixel (text, shapes, 8-bit images, 4-bit images that miss the fast path) is.

With the fix, those greys render one step darker than before, as the colour
asks for, and the lightest grey is distinct from white again. If you picked
your colours by eye on stock ESPHome, check them afterwards. It is one line,
and needs patch 08 applied first; see step 3 of [How to apply](#how-to-apply).

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
   for p in "$PATCHES"/*.patch; do
     case "$p" in *.optional.patch) continue ;; esac
     patch -p1 -N -b -r - -d "$SITE" < "$p"
   done
   ```

   `-b` keeps a `.orig` backup of every changed file. `-N` makes an accidental
   second run skip files that are already patched instead of stopping to ask,
   and `-r -` stops `patch` from leaving `.rej` files behind. Patches named
   `.optional.patch` are skipped; step 3 is for those.
3. Optionally, the [light-grey fix](#the-light-grey-fix), in the same shell:

   ```bash
   patch -p1 -N -r - --no-backup-if-mismatch -d "$SITE" < "$PATCHES/11-it8951-grey-levels.optional.patch"
   ```

   No backup on purpose, not even the one `patch` makes by itself when a
   patch applies with an offset: it would overwrite the `it8951.cpp.orig`
   that step 2 made from the stock file.
4. Check it worked: `esphome config your-device.yaml` must accept
   `type: GRAYSCALE_4BIT`. If it says the type is unknown, the patches are not
   applied.

**To undo**, reinstall the same version:
`pip install --force-reinstall --no-deps esphome==2026.11.0`.

**Updating from an earlier version of these patches:** undo first, then
apply. A patch that changed since you applied it does not fit over its old
version: `patch` applies some parts, reports the rest as `FAILED` and moves
on, which leaves ESPHome half-updated.

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
nothing, when it cannot copy the area as it is: clipping set, the display
rotated by 90 or 270 degrees, an odd x or width, the area not fully on the
screen, monochrome mode or `invert_colors` on.

## Preloading the next page

For configs that switch between full-screen images (dashboard pages, for
example). Optional: a config that does not use it behaves exactly as without
it, and nothing is allocated.

A page switch normally sends the whole frame to the panel (about 544 ms at
20 MHz) and then runs the refresh waveform (about 680 ms for GC16). Loading an
image into the IT8951's memory does not change what is on the glass, so the
driver can load the page you will probably show next while the panel is
idle. When you then draw that page, it only sends what differs from the
preloaded copy: whatever you draw on top of it, plus any area another update
overwrote in the meantime.

Measured on the reTerminal E1003, two 1872x1404 pages, 10 switches back and
forth per row, so medians of 5 per direction:

| | to page 1 | to page 2 |
|---|---|---|
| Without the preload | 1,330 ms | 1,335 ms |
| With the preload | **825 ms** | **873 ms** |
| Time spent sending to the panel at the switch | 544 ms to 46 ms | 544 ms to 84 ms |

The row without the preload comes from the first build that had it, with
the preload switched off; the row with it from the final build. Nothing
since changed the path the preload-off run takes: a full repaint with the
preload off still sends in 543-544 ms on the final build. What is left of a
switch is drawing the framebuffer (~65 ms), the overlays and the GC16
waveform itself, which the patches cannot shorten.

### How to use it

Name the page that is usually on screen and the one most likely to come
next. That is all the configuration there is (patch 13):

```yaml
display:
  - platform: it8951
    id: epd
    psram_dma: true
    # ...
    preload:
      home: page_1   # loaded while any other page is on screen
      next: page_2   # loaded while page_1 is on screen
    lambda: |-
      it.image(0, 0, id(current_page) == 1 ? id(page_1) : id(page_2));
      // Anything drawn here, after the page, is sent at the switch.
```

The driver works out which page is up from the last full-screen image it
drew, and loads the other one. For an `online_image` it also follows the
downloads by itself: a page is never preloaded while it decodes, and is
loaded again once the download is done. `next` is optional. With more than
two pages, any page other than `home` counts as away, so `home` is what gets
preloaded then.

The build stops with an error if a `preload:` image is not an opaque
`GRAYSCALE_4BIT` image of exactly the panel's size, or if the display has
`grayscale: false` or its axes swapped. The size is checked when it is known
at build time; an `online_image` without `resize:` only has one once it
arrives, and a wrong size then simply means the preload is never used.

Optionally, a switch in Home Assistant to turn it off and on, for comparing
or if your panel turns out not to keep the preloaded page:

```yaml
switch:
  - platform: template
    name: "Preload next page"
    restore_mode: RESTORE_DEFAULT_ON
    optimistic: true
    turn_on_action:
      - lambda: 'id(epd).set_preload_enabled(true);'
    turn_off_action:
      - lambda: 'id(epd).set_preload_enabled(false);'
```

`RESTORE_DEFAULT_ON` brings back the last state after a reboot or a flash,
so once you turn it off it stays off until you turn it on again.

### For other rules: three lambda calls

Without `preload:`, a config can drive the preload itself:

- `id(epd).set_preload_source(img->get_data_start(), img->get_width_stride(), tag)`
  for the image you expect to draw next. `tag` is any number you choose, to
  name the source later. Setting the same source again does nothing, so you
  can call it freely. Call it after drawing the current page, not before:
  that draw still uses the old preload.
- `id(epd).drop_preload_source(tag)` **before** that image's content changes,
  for example from its `on_download_started`: an `online_image` decodes into
  the same buffer, at the same address, so the driver cannot notice the
  change by itself. Set the source again after the download.
- `id(epd).set_preload_enabled(false)` turns it off at runtime.

`preload:` uses tags 1 (`home`) and 2 (`next`) itself, so use one way or the
other, not both.

### Rules and caveats

- **The page must be drawn whole, first.** The preload is only used by an
  `it.image(0, 0, img)` that covers the entire panel, with the preloaded
  image. The image needs the fast drawing path: opaque, `GRAYSCALE_4BIT`,
  exactly the panel's size, greyscale mode, `invert_colors` off, not rotated
  by 90 or 270 degrees. Anything else is ignored, and the update simply sends
  everything as before.
- **Writing into a page image yourself.** `preload:` follows `online_image`
  downloads, including your own `on_download_finished` handlers, which run
  before it loads the page again. Anything else that changes a page image's
  pixels, or `online_image.release`, it does not see: call
  `id(epd).preload_page_changing(1)` before (1 for `home`, 2 for `next`), and
  `id(epd).preload_page_changed(1)` once the image is whole again. These two
  exist only when the config has an `image:` (ESPHome then defines
  `USE_IMAGE`; `online_image` counts), which any preload config does. With
  the lambda calls, drop the source instead. A missed change is the one mistake
  that shows wrong pixels: the switch would send only the overlays on top of
  the old copy of the page.
- **One preloaded page.** The controller holds the page on screen and one
  more.
- **It relies on the controller keeping what was loaded until the switch**,
  including while it sleeps between updates (`sleep_when_done: true`). That
  holds on the reTerminal E1003. On other IT8951 panels, check a few switches
  by eye: if pieces of the previous page stay visible after a switch, that
  panel loses it, and you should leave the preload off.
- **Background work after every full redraw.** The driver waits for the
  waveform to finish (~0.7 s), then loads the next page (~570 ms of SPI, in
  20 ms slices). Any update you request meanwhile goes first. The main loop
  runs at full speed during that time, which matters if you use light sleep.
- **Memory, from the first preload on:** about 10 KB of PSRAM (two maps of
  one byte per 32x16 block) and 4 KB of internal RAM for a staging buffer.
- **Partial updates get a little faster too** once the staging buffer exists:
  narrow areas go out as one SPI transfer per 4 KB instead of one per row.
- **The `Update took` log line gets shorter** (VERBOSE): when a preload
  follows, the controller is put to sleep after the preload instead of at the
  end of the update, so ~160 ms of waiting moves out of that line. It also
  ends in `N loads`, the number of areas sent; a switch that found the preload
  sends only a few. Both it and the `Preload rows 0-1404 done` line need
  `logger: level: VERBOSE`; a per-tag `logs:` entry cannot go above the
  global level.
- Monochrome mode (`grayscale: false`) does not support the preload: the
  build refuses `preload:`, and the lambda calls do nothing.

## Tips

- `psram_dma: true` on the display (shown above) needs no patch and makes
  sending each frame to the panel about a third faster: ~900 ms to ~575 ms per
  full frame on the reTerminal E1003, measured before 2026.11 and before 08's
  four-rows-per-transfer change.
- Downloaded images decode fastest when they arrive at the size the image is
  configured for (`resize:`, or no `resize:` at all), so serve them at the size
  you draw them.
- The 4-bit type does not work with LVGL.

## ESPHome 2026.9 and 2026.10

The older set in [`esphome-patches/2026.9-2026.10/`](esphome-patches/2026.9-2026.10/)
was checked against 2026.9.0, 2026.9.1 and 2026.10.2. Besides the speed-ups it
also has the light-grey fix and raises the driver's `Update took N ms` log line
to INFO. It also carries decoding speed-ups (05, 06 and part of 07) that
ESPHome 2026.11 has built in.

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
