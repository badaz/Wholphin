# Ugoos SK1 audio sink workaround

This branch is upstream Wholphin plus a workaround for playback freezing on a black screen on the Ugoos SK1 when audio is passed through (E-AC3) over HDMI. It is not proposed upstream: it is rebased onto each Wholphin release.

This file is the single reference for this work: everything needed to pick it up on another machine is here.

## Status (2026-10-28)

The bug no longer shows with Wholphin 1.0.9 or later, even without this workaround: media3 1.11.1, which Wholphin 1.0.9 ships, recovers on its own. The firmware did not fix it. Tested on firmware 2.2.0 (build `eng.ugoos.20260903.125945`) with episodes of season 1 of *Carrie* (E-AC3):

| Build                                          | media3 | `-38` errors     | Recovered on its own                                |
| ---------------------------------------------- | ------ | ---------------- | --------------------------------------------------- |
| Store Wholphin 1.0.9, no workaround            | 1.11.1 | 12 over 8 starts | all, within 70 to 340 ms                            |
| Upstream v1.0.3 + `releaseTest`, no workaround | 1.10.1 | 2 over 2 starts  | first only; the second froze for good, as on 2.0.7  |

On 2.2.0, `AudioFlinger` still refuses the output as often as on 2.0.7, for TrueHD as well as E-AC3. With media3 1.10.1, after the refusal the HAL output is closed and media3 never asks for it again. A likely cause, not checked in media3's code, is the 1.11.1 fix "release count doesn't decrease when playback thread is no longer alive": media3 waits for pending releases to reach zero before retrying.

The workaround stays as a safety net in case a later media3 breaks this retry again. It costs nothing when media3 recovers, since it only acts after 2 s stuck in buffering.

## Setting up a machine

```
git clone -b sk1-audio-sink-workaround git@github.com:badaz/Wholphin.git wholphin-sk1
cd wholphin-sk1
git remote rename origin fork
git remote add origin https://github.com/damontecres/wholphin.git
git fetch --tags origin
git branch -u fork/sk1-audio-sink-workaround
keytool -genkeypair -keystore wholphin-test.keystore -alias wholphintest -storepass wholphintest -keypass wholphintest -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=Wholphin Test"
adb connect 192.168.2.82:5555
```

This gives the same layout as the original machine: `origin` is upstream `damontecres/wholphin`, `fork` is `badaz/Wholphin`, and a plain `git push` goes to the fork. The keystore is gitignored, so each machine generates its own. A build signed with a different keystore cannot update the one already installed on the box: uninstall "Wholphin (Test)" first (`adb uninstall com.github.damontecres.wholphin.test`). Building needs the Android SDK; Gradle installs the missing platforms itself.

## The bug

On the SK1, playback sometimes never starts: the screen stays black (or frozen on the last frame) with no error. It happens mostly when starting a film, sometimes after a seek.

The SK1 (Amlogic S928X, Dolby MS12 audio HAL) exposes a single `E_AC3` `DIRECT|COMPRESS_OFFLOAD` HDMI output. Media3 releases and recreates its `AudioTrack` at playback start and on every seek. If the new track is requested while the previous one is still being released, `AudioFlinger` refuses it:

```
AudioTrack: createTrack_l(...): AudioFlinger could not create track, status: -38
```

`-38` is `INVALID_OPERATION`. Media3 reports an `AudioSink.InitializationException`, the audio renderer is left without a sink and never reports itself ready, so video has no clock to render against. The player sits in `STATE_BUFFERING` with no error, and the user sees a black screen.

There are two variants:

- At playback start (position around 0), media3 1.10 never recovers on its own: the freeze is permanent. This is the common, user-visible case. Media3 1.11.1 recovers (see [Status](#status-2026-10-28)).
- After a mid-playback seek, media3's internal retry usually succeeds within about 70 ms, so it is often invisible. It was reproduced with seeks about 1.2 s apart on a shallow buffer.

The tell that this is the bug and not the network: the player is stuck buffering with plenty already buffered ahead (17 s and 7.5 s were observed).

## The fix

`PlaybackViewModel.onAudioSinkError` reacts to `AudioSink.InitializationException` only:

1. Wait 2 s, since the sink often recovers on its own within about a second.
2. If the player is still in `STATE_BUFFERING` with at least 3 s buffered ahead, seek 1 s away from the current position, backwards when possible and forwards near position 0. The seek makes media3 initialize the sink again.
3. Give up after 3 attempts. The counter resets on `STATE_READY`.

A permanent black screen becomes a spinner of about 2 s. It recovered 3 out of 3 real startup freezes in a release build.

Things that do not work, and why:

- Disabling audio offload (`AudioOffloadPreferences`, or Wholphin's "disable hardware audio offload" setting) has no effect: E-AC3 passthrough never goes through media3's offload path.
- `seekTo(currentPosition)` is a no-op in ExoPlayer, so the recovery seek has to be offset. Near position 0 it must seek forwards, since clamping to 0 is the same no-op.

## Reproducing and monitoring

Checked on the box on 2026-10-28.

A continuous `adb logcat` over Wi-Fi drops after a few minutes (the box logs a lot, and the connection blips). Instead, reproduce, then dump the box's log buffer right away; it holds about 20 minutes:

```
adb -s 192.168.2.82:5555 logcat -d -v time > box.log
grep -E 'Playing [0-9a-f-]{36}|status: -38|adev_open_output_stream:3098|adev_close_output_stream_new|out_set_volume:1136|onAudioSinkError|Audio sink is stalled|Giving up on audio sink recovery' box.log
```

How to read it:

- `I/Wholphin: Playing <item id>` marks each playback start.
- `audio_hw_hal_primary: [adev_open_output_stream:3098] ... format:E_AC3(0xa000000) flags:0x11` is the HAL opening the passthrough output; the format says what is actually sent (`E_AC3`, `AC3`, `DOLBY_TRUEHD`).
- `AudioTrack: createTrack_l(...): AudioFlinger could not create track, status: -38`, preceded by `APM_AudioPolicyManager: getOutputForDevices() could not find output ... flags 0x1`, means the output was refused. Other statuses are a different problem (`-12`, for example, is the system running out of memory for the track).
- Recovered: within a few hundred milliseconds, another `adev_open_output_stream` for the same format, then `out_set_volume:1136 ... internal_format:E_AC3`, which shows the new track playing.
- Frozen: after the `-38`, an `adev_close_output_stream_new` and then nothing for that format. `adb shell dumpsys media_session` then shows `state=BUFFERING(6), position=0, ... error=null`. That state is only reliable while frozen: during normal playback it can be stale.
- With the `releaseTest` build, `onAudioSinkError` should follow, then `Audio sink is stalled, seeking to …` about 2 s later, then playback resumes. `Audio sink recovered on its own` means media3 recovered without help, which is common after a seek.
- `Giving up on audio sink recovery after 3 attempts` means the workaround failed: the screen stays black.
- With the store app, a `-38` line followed by a black screen that never starts is the unpatched bug.

To reproduce, at human speed (see below):

1. Use a build with media3 1.10, since 1.11.1 recovers on its own: upstream v1.0.3 without the workaround. Build it in a separate worktree, `git worktree add --detach ../wh-103 v1.0.3`, cherry-pick the commit that adds the `releaseTest` build type, copy `wholphin-test.keystore`, then build and install as in [Building the test app](#building-the-test-app). It installs as "Wholphin (Test)" and needs signing in to the Jellyfin server.
2. Use an E-AC3 title: any episode of season 1 of *Carrie* works. Check in the log that the output opens as `format:E_AC3`.
3. At playback start: start the episode, wait for the picture (or 15 s), stop, and start again, a few seconds apart. With media3 1.10 it froze on the second start.
4. After a seek: right after the film starts, while the buffer is still short, seek a few times about 1 s apart. Seeks about 1.2 s apart reproduced it on firmware 2.0.7; 1.6 s apart did not.
5. To check the workaround itself, rebase it onto v1.0.3 the same way and repeat: the freeze should turn into a spinner of about 2 s.

Seen from the TV, the bug is a black screen (or a frozen frame after a seek) with the loading spinner and no error message.

## Debugging on the box

The box is at `192.168.2.82:5555` over adb.

- Its userspace is 32-bit (`ro.product.cpu.abilist=armeabi-v7a,armeabi`): install the `armeabi-v7a` APK, the `arm64-v8a` one fails with `INSTALL_FAILED_NO_MATCHING_ABIS`.
- In release builds, R8 obfuscates class names, so Timber logs everything under the tag `Wholphin`, and only INFO and above. Grep log messages, not tags:

  ```
  adb -s 192.168.2.82:5555 logcat -s Wholphin | grep -iE 'audio sink|onAudioSinkError'
  ```

  When the workaround kicks in, `onAudioSinkError` is followed by `Audio sink is stalled, seeking to …`. The `status: -38` line comes from the system `AudioTrack` tag, so this filter misses it: use the log dump in [Reproducing and monitoring](#reproducing-and-monitoring).
- Read the firmware version with `adb shell getprop ro.build.version.incremental`.

Test at human speed. An automated loop of playback restarts and rapid seeks kernel-panicked the box (`sys.boot.reason=kernel_panic`), with no usable trace, and lost an installed app. The Amlogic audio driver is fragile exactly along the path being tested. Ordinary film starts reproduce the bug readily, so daily use is a better test than synthetic stress. When seeking repeatedly, seek backwards in bursts and forwards one step at a time, so whoever is watching does not lose their place.

## Building the test app

The `releaseTest` build type is a minified, non-debuggable build with the application id suffix `.test` and the name "Wholphin (Test)", so it installs alongside the store app. It is signed with `wholphin-test.keystore` at the repository root (see [Setting up a machine](#setting-up-a-machine)).

```
./gradlew :app:assembleDefaultReleaseTest
adb -s 192.168.2.82:5555 install -r app/build/outputs/apk/default/releaseTest/Wholphin-default-releaseTest-<version>-armeabi-v7a.apk
```

Keep the store app installed: it is the unpatched comparison.

## Updating to a new Wholphin release

1. `git fetch --tags origin`
2. `git rebase <new tag>`
3. Check that upstream has not started handling audio sink errors itself (`git grep -n onAudioSinkError <new tag> -- app/src`), and read the media3 release notes for changes to `DefaultAudioSink` retries.
4. Build `releaseTest`, install it and use it for a while.
5. `git push --force-with-lease --force-if-includes`

## When the workaround can go

With Wholphin 1.0.9 (media3 1.11.1), the workaround is no longer needed (see [Status](#status-2026-10-28)). Keep it while it rebases cleanly, and drop it if it starts conflicting.

The output refusal itself is in the vendor audio driver and survives firmware 2.2.0, despite "Fixed some audio passthrough issues" in 2.0.8 and "Audio passthrough fixes" in 2.2.0. To check a later firmware ([reflashing guide](https://ugoos.com/amlogic-firmware-reflashing-guide); 2.0.7 changed how to enter recovery), start E-AC3 titles and count the `-38` lines: if there are none over several starts and seeks, the driver is fixed.

After a media3 upgrade in Wholphin, check again with the store app: several starts of an E-AC3 title, then the log. Any `-38` not followed by a reopened output means media3 stopped recovering, and the workaround is needed again.
