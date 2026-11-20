# ESPHome patches for IT8951 e-paper displays

Author note: Generated using Claude Opus 5.5.

Patches for ESPHome that make IT8951 e-paper panels render faster, plus a
4-bit greyscale image type (`type: GRAYSCALE_4BIT`, half the memory of
`GRAYSCALE`). Tested on the Seeed Studio reTerminal E1003 only; other IT8951
panels should work.

Measured on the reTerminal E1003 with the 2026.11 set, medians of 10 runs,
each row with and without that change:

| | without | with |
|---|---|---|
| Drawing a full-screen 1872x1404 4-bit image: row copy instead of pixel by pixel (01-04, 08) | 2,464 ms | **62 ms** |
| Sending a full frame, 20 MHz SPI, `psram_dma: true`: four rows per transfer instead of one (08) | 572.5 ms | **543 ms** |
| Downloading and decoding a 1872x1404 opaque 4-bit PNG at its own size (10) | 4.73 s | **4.35 s** |
| Page switch until on the glass, with the next page [preloaded](#preloading-the-next-page) (08, 13, opt-in) | 1,330-1,335 ms | **825-873 ms** |

The draw blocks ESPHome's main loop, and with it touch, for its whole
duration. Its "without" figure is the patched 4-bit pixel path with the fast
path off: stock ESPHome cannot draw 4-bit at all.

## The setup these were written for

A reTerminal E1003 showing Home Assistant dashboards:

```
HA view (1872x1404) -> hass-lovelace-kindle-screensaver (PNG, every 30 min)
  -> online_image, GRAYSCALE_4BIT, one buffer per page
  -> lambda: it.image(0, 0, page), then overlays
  -> IT8951: full refresh, or only the overlay's area
```

- Pages are laid out in HA (HTML/CSS) and rendered by
  [hass-lovelace-kindle-screensaver](https://github.com/sibbl/hass-lovelace-kindle-screensaver)
  at `/` and `/2`. The firmware draws no widgets.
- Both pages stay decoded in PSRAM: 1.25 MiB each at 4 bit, 2.5 at 8 bit.
  Polled every 10 min (304 if unchanged). A page switch is a repaint, not a
  ~4 s decode.
- Taps are answered with overlays (a render takes ~20 s): 4-bit crops of the
  render, drawn over the page on HA state changes and refreshed alone.

None of this is required: the 4-bit type and fast path help any config that
draws large images. Full config:
[two-page example](#example-a-two-page-home-assistant-dashboard).

## Versions

| ESPHome | Folder |
|---|---|
| 2026.11.0 | [`esphome-patches/2026.11/`](esphome-patches/2026.11/) |
| 2026.9.x, 2026.10.x | [`esphome-patches/2026.9-2026.10/`](esphome-patches/2026.9-2026.10/), see [below](#esphome-20269-and-202610) |

Exact versions only. Apply every patch in one folder (`.optional.patch` only
if wanted), nothing from the other.

## The 2026.11 patches

Checked against 2026.11.0. 05 and 06 are absent: 2026.11 does that work
itself.

| Patch | What it does |
|---|---|
| 01 | Display hook for the fast drawing path |
| 02 | 4-bit greyscale image type |
| 03 | Draws 4-bit images; hands whole images to the fast path |
| 04 | `GRAYSCALE_4BIT` in YAML. Greyscale PNGs load as opaque (stock treats them as transparent, which rules out the fast path) |
| 07 | `online_image` writes 4-bit |
| 08 | IT8951: fast drawing path, four rows per SPI transfer, the [preload](#preloading-the-next-page) |
| 09 | IT8951: header for 08 |
| 10 | Faster decoding of downloaded 4-bit images (opaque, at their own size) |
| 11 | Optional: the [light-grey fix](#the-light-grey-fix) |
| 13 | IT8951: the `preload:` YAML option |

Speed-ups only, unlike the older set: `Update took N ms` and
`Preload rows ...` stay at VERBOSE.

### The light-grey fix

Optional patch 11, because it changes how the screen looks. Stock ESPHome maps
8-bit grey to the panel's 16 levels by dividing by 16, not 17: greys from the
middle up render one step too light, and the lightest grey as white. This
affects everything drawn pixel by pixel (text, shapes, 8-bit images, 4-bit
images off the fast path); fast-path 4-bit images are copied unchanged.

With the fix those greys render as specified, one step darker than before.
Recheck colours picked by eye on stock ESPHome. Apply it after 08 (step 3
below), so 08's `it8951.cpp.orig` backup stays stock.

## How to apply

The patches modify the installed ESPHome, not your project, so they need a
pip or uv install. The HA add-on and the Docker image ship their own copy,
replaced on every update.

1. Pin an exact version: `pip install esphome==2026.11.0`. **Any upgrade or
   reinstall removes the patches.**
2. With ESPHome's venv active, from this repo's root (Git Bash, macOS, Linux):

   ```bash
   PATCHES=esphome-patches/2026.11
   SITE=$(python -c "import esphome, os; print(os.path.dirname(os.path.dirname(esphome.__file__)))")
   for p in "$PATCHES"/*.patch; do
     case "$p" in *.optional.patch) continue ;; esac
     patch -p1 -N -b -r - -d "$SITE" < "$p"
   done
   ```

   `-b` keeps `.orig` backups, `-N` skips already-patched files on a rerun,
   `-r -` leaves no `.rej` files.
3. Optional, the light-grey fix:

   ```bash
   patch -p1 -N -r - --no-backup-if-mismatch -d "$SITE" < "$PATCHES/11-it8951-grey-levels.optional.patch"
   ```

   No backup: one made on an offset apply would overwrite the stock
   `it8951.cpp.orig` from step 2.
4. Verify: `esphome config` must accept `type: GRAYSCALE_4BIT`.

**Undo:** `pip install --force-reinstall --no-deps esphome==2026.11.0`.

**Updating these patches:** undo, then apply. Applying over an older version
fails some hunks and leaves ESPHome half-patched.

## How to use

Set the type and draw as usual:

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

The fast path needs all of these; otherwise the image draws correctly at the
old speed:

- opaque image;
- fully on screen, no clipping region set;
- even x and width (with `mirror_x`, multiples of 8 are fastest);
- axes not swapped (no 90/270 rotation, no `swap_xy`);
- greyscale mode, `invert_colors` off.

Part of a 4-bit image: `it.draw_gray4_packed(x, y, w, h, src, src_stride)`,
with `src` = `image->get_data_start()` plus a byte offset (so an even source
x) and `src_stride` = `image->get_width_stride()`. Same conditions except
opacity, as bytes are copied as they are; returns `false` and draws nothing
when they fail.

## Preloading the next page

Optional, for configs that switch between full-screen images. Unused, it
allocates nothing and changes nothing.

A switch normally sends the whole frame (~544 ms at 20 MHz), then runs the
GC16 waveform (~680 ms). Loading into IT8951 memory does not change the glass,
so the driver loads the likely next page while the panel is idle. At the
switch it sends only what differs from that copy: overlays, and any area
another update overwrote meanwhile.

Two 1872x1404 pages, medians of 5 switches per direction:

| | to page 1 | to page 2 |
|---|---|---|
| Without preload | 1,330 ms | 1,335 ms |
| With preload | **825 ms** | **873 ms** |
| SPI transfer at the switch | 544 to 46 ms | 544 to 84 ms |

"Without" is the first preload build with the preload off; that path has not
changed since (still 543-544 ms). The rest of a switch is the framebuffer draw
(~65 ms), the overlays and the waveform.

### Configuration (patch 13)

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

- The page on screen is the last full-screen image drawn; the other is loaded.
- `online_image` downloads are followed: no preload while a page decodes, a
  reload once it is done.
- `next` is optional. With more than two pages every page but `home` counts as
  away, so `home` is preloaded.
- The build fails if a `preload:` image is not opaque `GRAYSCALE_4BIT` at
  exactly the panel size, or with `grayscale: false` or swapped axes. An
  `online_image` without `resize:` has no size until it arrives; a wrong one
  then just disables the preload.

Runtime switch, e.g. for panels that lose the preloaded page:

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

`RESTORE_DEFAULT_ON` keeps an off across reboots and flashes.

### Driving it from lambdas

Instead of `preload:` (which uses tags 1 and 2 itself; use one or the other):

- `id(epd).set_preload_source(img->get_data_start(), img->get_width_stride(), tag)`:
  the image you expect to draw next; `tag` is any number naming it.
  Idempotent. Call it after drawing the current page, which still uses the
  old preload.
- `id(epd).drop_preload_source(tag)` **before** the image changes, e.g. in
  `on_download_started`: `online_image` decodes into the same buffer at the
  same address, so the driver cannot notice. Set it again afterwards.
- `id(epd).set_preload_enabled(false)`: off at runtime.

### Caveats

- **Only a full-panel `it.image(0, 0, img)` of the preloaded image** on the
  fast path uses the preload (opaque, `GRAYSCALE_4BIT`, panel size,
  greyscale, `invert_colors` off, no 90/270 rotation). Anything else sends
  everything as before.
- **Writing into a page image yourself.** `preload:` sees `online_image`
  downloads, including your `on_download_finished` handlers (they run before
  the reload), but not other pixel writes or `online_image.release`. Wrap
  those in `id(epd).preload_page_changing(n)` and
  `id(epd).preload_page_changed(n)` (1 = `home`, 2 = `next`; they exist when
  `USE_IMAGE` is defined, which any preload config has). With the lambda
  calls, drop the source instead. A missed change is the one way to show
  wrong pixels: overlays sent onto the stale page.
- **One preloaded page**: the controller holds the visible page and one more.
- **The controller must keep the preloaded page**, also while asleep
  (`sleep_when_done: true`). True on the E1003. On other panels, check a few
  switches: leftovers of the previous page mean leave it off.
- **Background work after every full redraw**: wait for the waveform
  (~0.7 s), then load the page (~570 ms SPI in 20 ms slices). Your updates go
  first. The main loop runs at full speed meanwhile, which matters with light
  sleep.
- **Memory**, from the first preload: ~10 KB PSRAM (two maps, one byte per
  32x16 block) and a 4 KB internal staging buffer. That buffer also sends
  narrow partial updates as one SPI transfer per 4 KB instead of one per row.
- **Logs**: `Update took` drops ~160 ms, because the controller now sleeps after
  the preload, not after the update. It ends in `N loads` (areas sent; few on
  a preloaded switch). It and `Preload rows 0-1404 done` need a global
  `logger: level: VERBOSE`; a per-tag `logs:` entry cannot exceed it.
- **Not in monochrome** (`grayscale: false`): `preload:` is refused, the
  lambda calls do nothing.

## Example: a two-page Home Assistant dashboard

[The setup above](#the-setup-these-were-written-for), trimmed: two pages, a
page button, one tile driven by a HA light. Board config (`esphome`, `esp32`,
`psram`, `api`, `http_request`) omitted. Passes `esphome config` on patched
2026.11.0.

```yaml
substitutions:
  # hass-lovelace-kindle-screensaver, serving page 1 at / and page 2 at /2.
  renderer: "http://192.168.1.20:5050"

globals:
  - id: page
    type: int
    initial_value: "1"
  # Set by everything that needs the whole page drawn; cleared by the lambda.
  - id: redraw_page
    type: bool
    initial_value: "true"

spi:
  - id: epaper_spi
    clk_pin: GPIO7
    miso_pin: GPIO8
    mosi_pin: GPIO9

image:
  - platform: online_image
    id: page_1
    url: "${renderer}/"
    format: PNG
    type: GRAYSCALE_4BIT
    resize: 1872x1404
    update_interval: never
    on_download_finished:
      - if:
          condition:
            lambda: "return !cached && id(page) == 1;"
          then:
            - lambda: "id(redraw_page) = true;"
            - it8951.update: epd

  - platform: online_image
    id: page_2
    url: "${renderer}/2"
    format: PNG
    type: GRAYSCALE_4BIT
    resize: 1872x1404
    update_interval: never
    on_download_finished:
      - if:
          condition:
            lambda: "return !cached && id(page) == 2;"
          then:
            - lambda: "id(redraw_page) = true;"
            - it8951.update: epd

  # 240x246 crops of the light tile, cut from a render in each state.
  - platform: file
    file: images/light_on.png
    id: light_on
    type: GRAYSCALE_4BIT
  - platform: file
    file: images/light_off.png
    id: light_off
    type: GRAYSCALE_4BIT

display:
  - platform: it8951
    id: epd
    model: seeed-reterminal-e1003
    spi_id: epaper_spi
    psram_dma: true
    grayscale: true
    update_mode: GC16
    update_interval: never
    auto_clear_enabled: false
    transform:
      mirror_x: true
      mirror_y: false
    preload:
      home: page_1
      next: page_2
    lambda: |-
      auto *img = id(page) == 1 ? id(page_1) : id(page_2);
      if (img->get_data_start() == nullptr) {
        return;  // not downloaded yet
      }
      if (id(redraw_page)) {
        it.image(0, 0, img);
      }
      if (id(page) == 1) {
        it.image(1320, 1128, id(light).state ? id(light_on) : id(light_off));
      }
      id(redraw_page) = false;

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # The first fetch: the interval below cannot do it, Wi-Fi is not up yet.
  on_connect:
    - component.update: page_1
    - component.update: page_2

interval:
  - interval: 10min
    # Without this it also fires at boot, before Wi-Fi, and that fetch fails.
    startup_delay: 10min
    then:
      - component.update: page_1
      - component.update: page_2

binary_sensor:
  - platform: homeassistant
    id: light
    entity_id: light.living_room
    on_state:
      - if:
          condition:
            lambda: "return id(page) == 1;"
          then:
            - it8951.update: epd

  - platform: gpio
    id: next_page
    pin:
      number: GPIO4
      mode: INPUT_PULLUP
      inverted: true
    on_press:
      - lambda: |-
          id(page) = id(page) == 1 ? 2 : 1;
          id(redraw_page) = true;
      - it8951.update: epd
```

Notes:

- `auto_clear_enabled: false`: the framebuffer persists, so the lambda can
  draw only the tile. The driver refreshes only the drawn area, snapped to
  32 px columns.
- `redraw_page` is only set outside the lambda, never cleared: the driver
  merges pending updates into one paint, so the flag must accumulate. A
  "tile only" flag would drop a page switch merged with a light change.
- Crops are cut from a render at the tile's position (1320, 1128), so they
  match pixel for pixel. x and width are multiples of 8 (fast path). Re-cut
  after styling changes.
- `!cached`: no repaint on a 304.
- The light trigger checks the page: an update that draws nothing still
  counts towards `full_update_every`.
- `full_update_every` (default 30, max 255) forces a full-screen GC16. Raise
  it for frequent overlays.
- `mirror_x: true`: the E1003 panel is mirrored.
- Touch: a `touchscreen` binary sensor calls a HA action; the overlay follows
  the state HA sends back, not the tap.

## Tips

- `psram_dma: true` needs no patch: ~900 to ~575 ms per full frame on the
  E1003 (measured before 2026.11 and before 08's four-row transfers).
- Serve downloaded images at their configured size (`resize:`, or none):
  decoding is fastest then.
- The 4-bit type does not work with LVGL.

## ESPHome 2026.9 and 2026.10

[`esphome-patches/2026.9-2026.10/`](esphome-patches/2026.9-2026.10/), checked
against 2026.9.0, 2026.9.1 and 2026.10.2. Also contains the light-grey fix,
`Update took N ms` at INFO, and decode speed-ups (05, 06, part of 07) that
2026.11 has built in.

- **On 2026.10**, patch 08 reports `Hunk #8 FAILED` / `1 out of 9 hunks
  FAILED`: that watchdog tweak is already upstream, the rest applies. Any
  other failure means the wrong version; undo.
- **No fix for greyscale PNGs loading as transparent**: use RGB PNGs for
  `platform: file` (`online_image` is unaffected). Convert in place (Pillow
  ships with ESPHome), skipping truly transparent images:

  ```bash
  python -c "import sys; from PIL import Image; Image.open(sys.argv[1]).convert('RGB').save(sys.argv[1])" icon.png
  ```

## License

GPLv3, the same license as ESPHome's C++ code, which these patches modify.
