# Volume Dimmer (pulse_effect)

Keeps every LED in a segment at its selected primary color, dim in quiet and
brighter with louder audio. Attacks are immediate; brightness falls smoothly
back to the minimum. RGB and white channels are scaled together. Global and
segment brightness still apply, so they determine the available headroom.

## Enable

Append `pulse_effect` to your build environment's `custom_usermods`, alongside
`audioreactive`. For example, in the ignored `platformio_override.ini`:

```ini
[env:custom_build]
extends = env:esp32dev
custom_usermods = audioreactive auto_save pulse_effect
```

Preserve any existing board settings and other usermods. Build that environment
with `pio run -e custom_build`, install the firmware for your controller, and
select **Volume Dimmer** in the Effects list. No core source changes are needed.
This targets the current main-branch effect API (void callbacks), not older WLED releases.

## Controls

| Control | Default | Behavior |
| --- | --- | --- |
| Decay | 32 | Full-scale release time, 50–2090 ms; higher is slower (default 306 ms). |
| Sensitivity | 171 | Logarithmic audio multiplier: about 1/256x to 16x; 171 = 1x, 0 = off. |
| Minimum brightness | 26 | Quiet brightness, approximately 10% of the selected color. |
| Maximum brightness | 255 | Loud brightness; values below Minimum are treated as Minimum. |
| Noise floor | 2 | Additional gate on AudioReactive volume: slider 0–31 maps to threshold 0–255. |

All controls except Noise floor use 0–255. The primary color is uniform across
the segment; palettes do not affect it. The effect works on strips and matrices.

The Noise floor slider stays 0–31 because WLED stores this control in five bits.
Its threshold now extends to 255, with finer adjustment at the low end:

| Slider | Audio threshold |
| --- | --- |
| 0–8 | 0–8 (unchanged) |
| 12 | 18 |
| 16 | 43 |
| 20 | 80 |
| 24 | 132 |
| 28 | 197 |
| 31 | 255 |

Sensitivity now doubles approximately every 21 slider steps: 128 ≈ 0.24x,
171 = 1x, 192 = 2x, 213 = 4x, 234 = 8x, and 255 = 16x. Small responses retain
fractional precision until rendering. The minimum nonzero gain is about 0.0037x.
The transfer above the noise floor remains linear in audio amplitude; the slider
itself is logarithmic. High gain can still saturate loud music at Maximum.

Existing presets retain their sensitivity value too: an old value of 128 now
means about 0.24x instead of 1x. Set it to 171 to recover the old unity gain.
New selections default to 171.

Existing presets retain their noise-floor slider position, so values above 8 now suppress
more audio and should be retuned. The default of 2 is unchanged. This is an
amplitude threshold, not a frequency filter: it cannot distinguish crickets
from music at the same processed volume. Raising the threshold also reduces
the response to music; Sensitivity still controls amplification above it.

Audio comes from AudioReactive's existing microphone or audio-sync input. It
uses `volumeRaw` (int16_t data slot 1), which includes AudioReactive squelch and
gain/AGC but bypasses its smoothed-volume limiter. The effect provides its own
release envelope and does not read the microphone directly or modify audio data.
Set AudioReactive squelch above ambient noise and tune gain before adjusting
Sensitivity. Disable AGC if you want differences in actual loudness preserved.

Without an enabled AudioReactive usermod, the effect settles at Minimum; it
never substitutes simulated music. Switching effects resets the per-segment
envelope. Maximum below Minimum produces constant Minimum brightness.

## Implementation references

Based on WLED's `usermods/user_fx/user_fx.cpp` registration pattern,
`usermods/audioreactive/audio_reactive.cpp` data exchange, and the core
`color_blend` RGBW helper, inspected at upstream commit `92064ae3`.
The design implements the Volume Dimmer described in “Audio Reactive WLED Effects.”
