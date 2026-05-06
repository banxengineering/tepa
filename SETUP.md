# Waveshare ESP32-S3 Touch AMOLED 1.32" — Arduino IDE Setup
## TEPA — Animated Eyes + Focus Timer

---

## Hardware

- **Board:** Waveshare ESP32-S3-Touch-AMOLED-1.32
- **Chip:** ESP32-S3-PICO-1-N8R8 (8MB Flash, 8MB OPI PSRAM)
- **Display:** 1.32" round AMOLED, 466×466, CO5300/SH8601 driver, QSPI interface
- **Touch:** CST820, I2C (SDA=GPIO47, SCL=GPIO48)
- **Power pin:** GPIO18 must be set HIGH to power the display

---

## Arduino IDE Board Settings

| Setting | Value |
|---|---|
| Board | ESP32S3 Dev Module |
| USB CDC On Boot | **Enabled** ← critical for Serial to work |
| CPU Frequency | 240MHz (WiFi) |
| Flash Mode | QIO 80MHz |
| Flash Size | 8MB (64Mb) |
| Partition Scheme | **Huge APP (3MB No OTA/1MB SPIFFS)** ← critical |
| PSRAM | **OPI PSRAM** ← critical |
| Upload Mode | UART0 / Hardware CDC |
| Upload Speed | 921600 |
| USB Mode | Hardware CDC and JTAG |

> ⚠️ **Partition Scheme must be Huge APP.** The full TEPA firmware (display + audio codec + WiFi + LLM) exceeds the default 1.28 MB partition. If you see `text section exceeds available space in board` during compilation, this is the cause. Go to **Tools → Partition Scheme → Huge APP (3MB No OTA/1MB SPIFFS)** and recompile.

---

## Required Libraries

Install from Arduino IDE → Sketch → Include Library → Manage Libraries:

- **Arduino_GFX Library** by Moon On Our Nation

---

## Display Wiring (internal — no wiring needed)

All display pins are internal to the board:

| Signal | GPIO |
|---|---|
| CS | 10 |
| SCLK | 11 |
| D0 | 12 |
| D1 | 13 |
| D2 | 14 |
| D3 | 15 |
| RST | 8 |
| Power | 18 |
| Touch SDA | 47 |
| Touch SCL | 48 |
| Touch RST | 7 |

---

## API Key Setup (Voice + AI Chat)

TEPA uses **one free Groq API key** for everything — Speech-to-Text (Whisper), LLM chat (Llama), and TTS (Orpheus). Free tier resets monthly and is more than sufficient for normal hobby use.

### Step 1 — Create your Groq API key

1. Go to [https://console.groq.com](https://console.groq.com) and sign up (free)
2. Click **API Keys → Create API Key** and copy it
3. In the Arduino sketch folder, create a file called `secrets.h`:

```cpp
#pragma once
#define GROQ_API_KEY    "gsk_your_key_here"
#define GROQ_LLM_MODEL  "llama-3.3-70b-versatile"
#define GROQ_STT_MODEL  "whisper-large-v3-turbo"
#define GROQ_TTS_MODEL  "canopylabs/orpheus-v1-english"
#define GROQ_TTS_VOICE  "austin"
```

4. `secrets.h` is in `.gitignore` — it will **never** be committed to GitHub

### Step 2 — Accept Orpheus TTS model terms

> ⚠️ **Required before TTS works.** The Orpheus voice model requires a one-time terms acceptance in the Groq console.

1. While logged into your Groq account, visit:  
   **[https://console.groq.com/playground?model=canopylabs%2Forpheus-v1-english](https://console.groq.com/playground?model=canopylabs%2Forpheus-v1-english)**
2. Look for a **yellow/orange banner at the top** of the page — click **Accept** or **Agree**
3. Done — acceptance is instant, no reflash needed

Without this step, TTS calls return `HTTP 400 model_terms_required`.

### LLM model options (all free on Groq)

| Model | Speed | Best for |
|-------|-------|----------|
| `llama-3.3-70b-versatile` | Fast | Best personality, default |
| `llama-3.1-8b-instant` | Fastest | Quick replies, lowest latency |
| `gemma2-9b-it` | Fast | Warm conversational tone |

### TTS voice options (`GROQ_TTS_VOICE`)

Groq Orpheus supports exactly these six voices:

- **Female:** `autumn` · `diana` · `hannah`
- **Male:** `austin` · `daniel` · `troy`

---

## Key Code Patterns

### Display Init (correct — Canvas approach)
```cpp
// Panel is the raw CO5300 driver; Canvas wraps it for buffered drawing.
// col_offset=9 is the confirmed working value for this board revision.
// Rotation=2 on the Canvas gives correct upright orientation (USB-C at bottom).
Arduino_DataBus *bus  = new Arduino_ESP32QSPI(10, 11, 12, 13, 14, 15);
Arduino_CO5300  *panel = new Arduino_CO5300(bus, 8, 0, 466, 466, 9, 0);
Arduino_Canvas  *gfx   = new Arduino_Canvas(466, 466, panel, 0, 0, 2);

pinMode(18, OUTPUT);
digitalWrite(18, HIGH);  // power on display
delay(200);

gfx->begin(80000000);     // initialises panel automatically at 80MHz
panel->setBrightness(255); // call on panel*, not gfx*
panel->fillScreen(0x0000); // pre-clear hardware buffer before canvas takes over
gfx->fillScreen(0x0000);
gfx->flush();              // push canvas buffer to panel
```

### Drawing and updating
```cpp
// All drawing calls go to the canvas (in-memory, no flicker).
gfx->fillScreen(BLACK);
gfx->fillRoundRect(...);
// ...
gfx->flush();  // REQUIRED after every complete frame — nothing shows without this
```

### What NOT to do
```cpp
// WRONG — 'false' sets width=0, nothing draws
new Arduino_CO5300(bus, rst, 0, false, 466, 466);

// WRONG — brightness method on base pointer
Arduino_GFX *gfx = ...;
gfx->Display_Brightness(255); // doesn't exist on base class
panel->setBrightness(255);    // correct: call on Arduino_CO5300*

// WRONG — calling panel->begin() separately when using Canvas
panel->begin();  // Canvas::begin() calls this internally; calling twice breaks init

// WRONG — col_offset=6 leaves a green sliver at 9 o'clock
new Arduino_CO5300(bus, rst, 0, 466, 466, 6, 0);
// col_offset=0  → sliver moves to 9 o'clock (larger)
// col_offset=12 → sliver moves to 3 o'clock
// col_offset=9  → centered, both edges hidden by bezel ✓
```

---

## Lessons Learned

1. **USB CDC On Boot must be Enabled** — without it, Serial Monitor shows nothing and the board appears dead
2. **Constructor parameter order matters** — `false` was silently setting width=0; always use `466, 466`
3. **setBrightness(255) is required** — AMOLED has no auto-backlight; call on `Arduino_CO5300*` not base pointer
4. **col_offset=9** — this board revision needs offset 9 to center content; offset 6 gives a green sliver at 9 o'clock, offset 12 moves sliver to 3 o'clock, offset 9 hides both edges behind the bezel
5. **Use Arduino_Canvas, not direct CO5300** — Canvas provides a standard top-left coordinate system, handles rotation correctly, and eliminates flicker by buffering the full frame in PSRAM
6. **Canvas::begin() initialises the panel** — do not call `panel->begin()` separately; it will break init
7. **gfx->flush() is mandatory** — nothing appears on screen without it after each frame
8. **Rotation=2 on the Canvas** — gives correct upright orientation with USB-C port at the bottom
9. **begin(80000000)** — always pass explicit 80MHz clock speed
10. **GPIO18 HIGH** — must be set HIGH before display init or display stays off
11. **OPI PSRAM must be selected** — Canvas allocates its 466×466×2 = 434KB framebuffer in PSRAM; without OPI PSRAM selected in Arduino IDE the allocation fails silently
12. **panel->fillScreen(BLACK) before gfx->flush()** — pre-clears the hardware buffer so uninitialized display memory doesn't bleed through on first frame
13. **Smart redraw beats full clear for eye animation** — only erase pixels that actually changed to reduce flicker; Canvas double-buffering also helps significantly

---

## Touch (CST820) Gesture Codes

| Code | Gesture |
|---|---|
| 0x01 | Swipe up |
| 0x02 | Swipe left |
| 0x03 | Swipe down |
| 0x04 | Swipe right |
| 0x05 | Single tap (if the controller reports it) |

On this Waveshare 1.32" panel, read **two bytes starting at register 0x01**: gesture ID, then finger count (0x02). Only act on gesture if finger count > 0. (Reading from 0x00 misaligns fields and breaks swipes.)

Many CST8xx builds report swipes reliably but not tap as **0x05**. The firmware also treats a **short touch release** (about 30–900 ms) with **no swipe** during the stroke as the same as tap (**0x05**) for app picker and settings.

---

## Servo (S90 on XIAO ESP32-C3 — separate project)

```cpp
const int SERVO_PIN = 5;  // GPIO5 = D3 on XIAO board label
const int PWM_FREQ  = 50;
const int PWM_RES   = 14; // 14-bit max on ESP32-C3 (NOT 16-bit)

uint32_t usToDuty(uint32_t us) {
  return (uint32_t)((us * 16383UL) / 20000UL);
}
// setup: ledcAttach(SERVO_PIN, PWM_FREQ, PWM_RES);
// CW:    ledcWrite(SERVO_PIN, usToDuty(1000));
// CCW:   ledcWrite(SERVO_PIN, usToDuty(2000));
```
