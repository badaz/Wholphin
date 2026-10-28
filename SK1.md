# Ugoos SK1 audio sink workaround

This branch is upstream Wholphin plus a workaround for playback freezing on a black screen on the Ugoos SK1 when audio is passed through (E-AC3) over HDMI. It is not proposed upstream: it is rebased onto each Wholphin release.

This file is the single reference for this work: everything needed to pick it up on another machine is here.

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

- At playback start (position around 0), media3 never recovers on its own: the freeze is permanent. This is the common, user-visible case.
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

## Debugging on the box

The box is at `192.168.2.82:5555` over adb.

- Its userspace is 32-bit (`ro.product.cpu.abilist=armeabi-v7a,armeabi`): install the `armeabi-v7a` APK, the `arm64-v8a` one fails with `INSTALL_FAILED_NO_MATCHING_ABIS`.
- In release builds, R8 obfuscates class names, so Timber logs everything under the tag `Wholphin`, and only INFO and above. Grep log messages, not tags:

  ```
  adb -s 192.168.2.82:5555 logcat -s Wholphin | grep -iE 'audio sink|onAudioSinkError'
  ```

  When the workaround kicks in, `onAudioSinkError` is followed by `Audio sink is stalled, seeking to …`. The `status: -38` line comes from the system `AudioTrack` tag, so look for it in the unfiltered logcat.
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

The defect is in the vendor audio driver. The box runs firmware 2.0.7. Ugoos's changelogs announce "Fixed some audio passthrough issues" in 2.0.8 and "Audio passthrough fixes" in 2.2.0, with no detail and no user reports either way. To check, flash the latest firmware ([guide](https://ugoos.com/amlogic-firmware-reflashing-guide); 2.0.7 changed how to enter recovery), then start E-AC3 films with the unpatched store app. If several starts go through without `status: -38` or a black screen, the firmware fixed it and this branch is no longer needed.
