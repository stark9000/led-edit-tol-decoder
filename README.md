# LED Edit .tol Decoder

Reverse-engineered reader and players for **LED Edit `.tol`** pixel animation files.
Includes a Java 8 Swing desktop player and an Arduino/ESP32 sketch that plays `.tol` files from an SD card with FastLED.

![Pink & Round 4 in the Rings view](docs/images/pink_round4_rings.png)

## Features

**Desktop player (Java 8 Swing)**
- Playlist with drag and drop, add files or a whole folder, auto-next or loop
- Play / pause / stop, frame stepping, seek slider, adjustable FPS
- Views: Rings (wheel), Spokes, Grid, Single strip
- Hub↔Rim flip, rotate, colour order (RGB, GRB, BRG...), LED size and glow
- All LED Edit wiring modes: `SINGLE_ROW`, `RETURN_ROW`, `SINGLE_COLUMN`, `RETURN_COLUMN`, with start (hub/rim) and direction (CW/CCW)
- Wiring path overlay and a raw preview that shows what an unmapped strip would display
- Export wired `.tol` files or a `wiring_map.h` lookup table for the Arduino
- Save the current frame as PNG

**Arduino player (ESP32 / Mega 2560)**
- Streams `.tol` files frame by frame from an SD card, so file size is not limited by RAM
- Plays every `.tol` file on the card in turn
- Optional wiring map for any physical chain order

## The .tol format

A `.tol` file is not compressed or encrypted: a 14-byte header followed by every frame as raw R, G, B bytes.
All numbers are big-endian.

| Offset | Size | Example | Meaning |
| --- | --- | --- | --- |
| 0 | 4 | `00 00 04 15` = 1045 | Frame count |
| 4 | 2 | `00 0a` = 10 | Dimension A (rings / rows) |
| 6 | 2 | `00 3c` = 60 | Dimension B (pixels per ring / columns) |
| 8 | 2 | `00 23` = 35 | Unknown |
| 10 | 2 | `00 16` = 22 | Unknown |
| 12 | 1 | `03` | Bytes per pixel |
| 13 | 1 | `00` | Reserved |
| 14 | … | | Frames: A × B × 3 bytes each |

- File size = `14 + frames × A × B × 3`, exactly.
- Pixel `p` of frame `f` starts at `14 + (f × A × B + p) × 3`.
- Channel values are **7-bit (0–127)**. Scale to 8-bit with `(v << 1) | (v >> 6)`.
- On a wheel (60 spokes × 10 rings), each group of 60 pixels is one ring, and the **outer rim ring is stored first**:
  `ring = 9 − (i / 60)`, `spoke = i % 60`.

The full step-by-step write-up, with pictures, is in the [tutorial](docs/LED_Edit_tol_Tutorial.docx).

## Desktop player

Requires Java 8 or newer.

Run the prebuilt jar:

```
java -jar desktop/dist/TolPlayer.jar
```

Or build from source:

```
javac -d out desktop/src/TolPlayer.java
jar cfe TolPlayer.jar TolPlayer -C out .
java -jar TolPlayer.jar "Round 3.tol"
```

Keys: **Space** play/pause, **← / →** step one frame.

## Arduino player

**Board:** ESP32 (recommended) or Arduino Mega 2560. A Uno or Nano does not have enough RAM for 600 pixels plus the SD library.
**Libraries:** FastLED, SD.

1. Format the SD card as FAT32 and copy the `.tol` files to the root.
2. Open `arduino/LedEditTolPlayer/LedEditTolPlayer.ino` and set:

   | Setting | Default | Notes |
   | --- | --- | --- |
   | `DATA_PIN` | 5 | Pixel data pin |
   | `SD_CS_PIN` | 4 | SD chip select (ESP32 VSPI: SCK 18, MISO 19, MOSI 23) |
   | `LED_TYPE` | WS2811 | |
   | `COLOR_ORDER` | RGB | GRB for most WS2812B |
   | `MAX_LEDS` | 600 | Must be ≥ A × B |
   | `FRAME_MS` | 40 | 40 ms = 25 fps |
   | `LOOPS_PER_FILE` | 3 | Repeats before the next file |

3. Upload, and open the serial monitor at 115200 baud to see which file is playing.

### Matching your wiring

The `.tol` file stores the picture ring by ring. If your LEDs are chained in a different order, use one of these:

- **Wired files:** in TolPlayer choose your wiring, click **Export wired .tol...**, and copy those files to the SD card. Keep `USE_WIRING_MAP 0`.
- **Lookup table:** click **Export map .h...**, put `wiring_map.h` next to the sketch, and set `USE_WIRING_MAP 1`. Keep the original `.tol` files on the card.

Tip: turn on **Raw preview** in TolPlayer and step through the wiring modes. The one that matches what your wheel shows today is your current wiring.

## What about the .led files?

LED Edit also writes `.led` files for hardware SD-card controllers. Their layout is known:

- `rzbv001` marker and a 512-byte header
- frame count at bytes 12–13
- 6,144-byte frames (8 outputs × 256 pixels × 3, bit-plane layout)
- 40 sectors of padding at the end

The pixel bytes, however, are scrambled with a per-file substitution that has not been decoded.
To play those animations, open the project in LED Edit and save it as `.tol`.

## Open questions

- Header bytes 8–11 (35 and 22 in every sample file). One may be the playback speed.
- Contributions with more sample files, especially different sizes or speeds, are welcome.

## Disclaimer

This is an independent project and is not affiliated with the makers of LED Edit.
The format was worked out from sample files for interoperability. No LED Edit code or files are included.
