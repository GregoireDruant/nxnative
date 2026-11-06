# feat(audio): add native 5.1 surround sound support with CoreAudio hardware detection and zero-downmix passthrough

## Summary

This PR introduces native 5.1 surround sound support to the audio subsystem (`dm/host/hle_audio.c` and `dm/host/hle_audren.c`), allowing games to output discrete 6-channel Linear PCM audio when the host is connected to a surround-capable audio setup (e.g. Mac connected via HDMI to an AV receiver / home cinema, multi-channel USB DAC, or virtual multi-channel audio interface). On stereo devices (MacBook built-in speakers, headphones), the existing software downmix matrix is cleanly preserved.

---

## What and why

### The problem
Previously, both the low-level `AudioOut` (`hle_audio.c`) and in-process `AudioRenderer` (`hle_audren.c`) layers were strictly constrained to 2-channel stereo output:
1. In `hle_audio.c`, `device_channels()` unconditionally returned `2` for any 6-channel guest stream (`a->ch == 6`), and `pull_device()` forcibly downmixed the 6 channels to stereo using Q15 coefficients.
2. In `hle_audren.c`, `capture()` always downmixed the `DeviceSink`'s multi-channel inputs into a stereo scratch buffer, and `render_main` opened `dm_pcmout_open(2)` with a hardcoded channel count of 2.
3. Switch audio device queries (`GetActiveAudioDeviceChannelCountForOutput` and `GetActiveChannelCount`) hardcoded `2` channels and reported `"AudioBuiltInSpeakerOutput"`. Consequently, game engines that query device capability at boot or on device change never enabled their internal 5.1 surround pipelines.

When playing on a Mac connected via HDMI to an AV receiver or surround system, all audio was downmixed to stereo, collapsing the soundstage and losing spatial separation for center dialog and surround effects.

### The change
1. **CoreAudio Hardware Channel Detection (`dm_audio_device_channels`)**:
   - On macOS (`!TARGET_OS_IPHONE`), queries CoreAudio HAL via `AudioObjectGetPropertyData` on the default output device (or the AudioUnit's `CurrentDevice`) for `kAudioDevicePropertyStreamConfiguration` on the output scope.
   - Computes the total physical output channels across all streams. If the device supports $\ge 6$ channels, 5.1 surround mode (`6`) is enabled; otherwise, stereo (`2`) is used.
   - Supports manual override or forcing via the `DM_AUDIO_CHANNELS` environment variable (`=6` forces 5.1 surround, `=2` forces stereo downmix; unset auto-detects hardware capability).
   - On iOS (`TARGET_OS_IPHONE`), returns `2` by default (iOS RemoteIO remains stereo).

2. **Passthrough Without Downmix in `hle_audio.c`**:
   - `device_channels(a)` returns `6` when the output device supports $\ge 6$ channels and the stream has 6 channels.
   - Configures the output `AudioUnit` (`DefaultOutput`) with `mChannelsPerFrame = 6` and assigns `AudioChannelLayout` `kAudioChannelLayoutTag_MPEG_5_1_D` (`L - R - C - LFE - Ls - Rs`), which precisely matches the Nintendo Switch / Audren 6-channel layout.
   - In `pull_device()`, when `device_channels(a) == 6`, samples are pulled directly into the output buffer without applying software downmix.
   - If the output device has only 2 channels, the Q15 downmix coefficients (`CF=32768, CC=23167, CL=8224, CB=23167`) remain active so voices and rear sounds are preserved on stereo speakers.
   - `au_open_at` logs whether the output is running in `5.1 surround`, `stereo downmix`, or `stereo`.

3. **6-Channel Rendering in `hle_audren.c`**:
   - Expanded `ar_t` internal buffers (`frame`, `cq`, `pcm`) to `FRAME_MAX * 6` to store up to 6 discrete channels per frame.
   - Added `out_ch` to `ar_t`, initialized to `audren_device_channels() >= 6 ? 6 : 2`.
   - `render_main` opens `dm_pcmout_open(a->out_ch)` and processes `n * ch * sizeof(int16_t)` bytes per block.
   - In `capture()`, when `out_ch == 6`, clamps and copies all 6 channels from `DeviceSink` (`ch[0]..ch[5]`) directly into `dst` without downmixing.
   - `GetActiveAudioDeviceChannelCountForOutput()` and `GetActiveChannelCount()` return `6` when surround is active (and `out_devname` reports `"AudioTvOutput"`, matching the Switch docked behavior), prompting games to render full 5.1 audio.

4. **Weak Linking for Isolated Unit Tests**:
   - In `hle_audren.c`, `dm_audio_device_channels` is declared with `__attribute__((weak))` so standalone tests that compile `hle_audren.c` without `hle_audio.c` (e.g. `t_audren_layout.c`, `t_audren_work.c`) default cleanly to stereo (`2`).

---

## Detailed File Changes

- **`dm/host/hle.h`**:
  - Declared `int dm_audio_device_channels(void);`.
- **`dm/host/hle_audio.c`**:
  - Implemented `mac_device_out_channels()`, `mac_default_output_device()`, and `dm_audio_device_channels()`.
  - Implemented `aout_device_channels()` and updated `device_channels()` to return 6 when 5.1 is supported.
  - Configured `AudioChannelLayout` `kAudioChannelLayoutTag_MPEG_5_1_D` on the `AudioUnit` in `au_create()`.
  - Bypassed downmix in `pull_device()` when `device_channels(a) == 6`.
  - Updated `stats_take()` to pass `device_channels(a)` for accurate zero-pad calculation.
  - Added channel mode logging to `au_open_at()`.
- **`dm/host/hle_audren.c`**:
  - Added weak declaration for `dm_audio_device_channels()`.
  - Sized `ar_t` buffers (`frame`, `cq`, `pcm`) to `FRAME_MAX * 6` and added `out_ch`.
  - Updated `render_main` to open `dm_pcmout_open(ch)` and transfer frame bytes according to `ch`.
  - Updated `ar_open()` to initialize `a->out_ch`.
  - Updated `capture()` to copy 6 discrete channels without downmixing when `out_ch == 6`.
  - Updated `out_devname()` to return `"AudioTvOutput"` when $\ge 6$ channels are active.
  - Updated `h_ad_Channels` and `h_ad_Channels7` to return the detected channel count (6 or 2).
- **`dm/host/tests/t_audio.c`**:
  - Added `test_surround_51_passthrough()` to verify that with `DM_AUDIO_CHANNELS=6`, all 6 discrete channels (FL, FR, Center, LFE, Surround Left, Surround Right) pass through without downmixing.
- **`dm/host/tests/t_audren_layout.c`**:
  - Added `build6()` and `test_51()` to verify 6-channel discrete capture across all supported SDK layouts.
- **`docs/env_switches.tsv`**:
  - Documented `DM_AUDIO_CHANNELS`.

---

## Checked

- [x] **Unit tests**:
  - `python3 -m unittest dm/tools/tests/test_env_switches.py` passed (`OK`).
  - Standalone C verification of `pull_device` and `capture()` in both 5.1 passthrough mode and stereo downmix mode passed.
  - Verified that on stereo devices / default settings, all existing stereo behavior and downmixing formulas are 100% preserved.
- [x] **Platform Scope**:
  - CoreAudio hardware detection logic is wrapped in `#if !TARGET_OS_IPHONE`. iOS targets remain unaffected and continue running in stereo.
