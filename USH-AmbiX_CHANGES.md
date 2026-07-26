# USH-AmbiX — Modifications to UltrasonicSuperHearing

This is a fork of [leomccormack/Super-Hearing](https://github.com/leomccormack/Super-Hearing), containing modifications to the **UltrasonicSuperHearing** JUCE plugin (in [`Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_`](Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_) and [`ultrasoniclib`](Ultrasonic-Super-Hearing/audio_plugins/ultrasoniclib)). All other content in this repository (Underwater-Super-Hearing, hardware designs, MATLAB scripts, papers) is unmodified and credited entirely to the original author, Leo McCormack.

The modified plugin is renamed **USH-AmbiX** (plugin code `BATA`, was `BATM`) so it does not conflict with the original `UltrasonicSuperHearing` plugin in any host.

## What changed

### 1. Dual-mode output: Binaural or AmbiX

The plugin now has an **Output** selector with two modes:

- **Binaural** (original behaviour) — 2-channel HRTF binauralisation of the pitch-shifted, DoA-estimated ultrasonic signal.
- **AmbiX** (new) — 4-channel 1st-order Ambisonics (ACN channel ordering, SN3D normalisation) encoding of the same pitch-shifted signal, steered to the estimated direction-of-arrival:
  - `W = 1`
  - `X = √3 · x`
  - `Y = √3 · y`
  - `Z = √3 · z`

  where `(x, y, z)` is the estimated unit-vector DoA.

The plugin **always declares 4 output channels** to the host, regardless of mode, so host routing/pinning stays fixed:
- In **Binaural** mode, output channels 1–2 carry L/R and channels 3–4 carry silence.
- In **AmbiX** mode, output channels 1–4 carry W/X/Y/Z.

A colour-coded badge in the plugin UI (amber = Binaural "L R", cyan = AmbiX "W X Y Z") makes the active mode unambiguous at a glance, and updates live.

The output mode is a saved/automatable-off host parameter (`outputMode`), so it persists with the session/preset.

### 2. Two additional pitch-shift options

Added **Down 4 Oct** (factor 0.0625) and **Down 5 Oct** (factor 0.03125) to the existing pitch-shift choices, for a total of 7 options: `none, Down 1 Oct, Down 2 Oct, Down 3 Oct, Down 4 Oct, Down 5 Oct, Use CH7`.

> **Preset compatibility note:** the underlying `ULTRASONICLIB_PITCHSHFT_OPTIONS` enum was renumbered to insert the two new options before `USE_CHANNEL_7` (which moved from enum value 5 to 7). Presets saved with the *original* plugin that had "Use CH7" selected will load into this fork as "Down 4 Oct" instead. This wasn't considered worth a migration path for personal use — flag it if you rely on old presets with that specific option selected.

## Files changed

| File | Change |
|---|---|
| [`CMakeLists.txt`](CMakeLists.txt) | Default to building VST3 instead of VST2 (no SDK required); guard `juce_set_vst2_sdk_path` so configuring without the VST2 SDK doesn't fail. |
| [`.../_UltrasonicSuperHearing_/CMakeLists.txt`](Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_/CMakeLists.txt) | Version bump to 1.1.0, `PLUGIN_CODE` → `BATA`, `PRODUCT_NAME` → `USH-AmbiX`. |
| [`ultrasoniclib.h`](Ultrasonic-Super-Hearing/audio_plugins/ultrasoniclib/include/ultrasoniclib.h) | New `ULTRASONICLIB_OUTPUT_MODE` enum + get/set API; two new pitch-shift enum values. |
| [`ultrasonic_internal.h`](Ultrasonic-Super-Hearing/audio_plugins/ultrasoniclib/src/ultrasonic_internal.h) | `NUM_SH_SIGNALS` (4) and `SQRT3` constants; `output_mode` field on the internal struct. |
| [`ultrasoniclib.c`](Ultrasonic-Super-Hearing/audio_plugins/ultrasoniclib/src/ultrasoniclib.c) | Output buffers/filterbank grown from 2 to 4 channels; synthesis stage branches on output mode (HRTF binaural vs. ACN/SN3D encoding); post-gain and output copy now cover all 4 channels; two new pitch-shift factor cases. |
| [`PluginProcessor.cpp`](Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_/src/PluginProcessor.cpp) | New `outputMode` JUCE parameter (with save/load state); output bus widened from 2 to 4 discrete channels; extra pitch-shift choice labels. |
| [`PluginEditor.h`](Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_/src/PluginEditor.h) | New combo-box member for the output mode selector. |
| [`PluginEditor.cpp`](Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_/src/PluginEditor.cpp) | Third UI row (output mode combo + colour-coded badge); window resized from 500×112 to 500×142. |

## Building

Same as upstream, with the addition that VST3 is now the default (no VST2 SDK download needed):

```bash
git clone --recurse-submodules https://github.com/wongchunhoi9/Super-Hearing.git
cd Super-Hearing
mkdir build && cd build
cmake .. -G Xcode -DBUILD_PLUGIN_FORMAT_VST3=ON -DBUILD_PLUGIN_FORMAT_VST2=OFF   # macOS
cmake --build . --config Release
```

The built plugin appears at:
```
build/Ultrasonic-Super-Hearing/audio_plugins/_UltrasonicSuperHearing_/UltrasonicSuperHearing_artefacts/Release/VST3/USH-AmbiX.vst3
```
and is auto-copied to the system VST3 folder (`COPY_PLUGIN_AFTER_BUILD`). Rescan plugins in your DAW afterwards.

## Usage notes (REAPER)

- **AmbiX mode**: route the plugin's 4 outputs (W, X, Y, Z) to an Ambisonics decoder/binauraliser (e.g. SPARTA AmbiBIN, IEM BinauralDecoder) for monitoring, or print to an ambisonics bus.
- **Binaural mode**: route outputs 1–2 (L, R) to a stereo/headphone bus; outputs 3–4 carry silence and can be ignored.
- Set the host block size to 512 samples to match the plugin's internal `FRAME_SIZE` for lowest latency (it still works at other block sizes via the internal block adapter).

## License

Unchanged from upstream: the `ultrasoniclib` DSP core remains under the permissive ISC License; the JUCE-based plugin wrapper code (`_UltrasonicSuperHearing_`, including the files modified here) remains under GPLv3. See the header of each source file and the main [README.md](README.md#license) for details.
