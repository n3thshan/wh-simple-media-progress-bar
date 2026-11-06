# Simple Media Progress Bar

A Windhawk mod that adds a seek bar with elapsed/total time to the media card in
Windows 11 Quick Settings.

Source: <https://github.com/n3thshan/wh-simple-media-progress-bar>.

- Windows 11 **24H2 (build 26100) or newer**, x86-64.
- The bar is a real XAML `Slider` inserted into the card, so it uses the platform
  look, the accent color, the standard animations, and the accessibility tree.
- The native previous / play-pause / next buttons are not modified. They keep
  their exact style and position; the bar is placed in its own row above them.
- Apps that do not expose a seekable timeline (live streams, some browsers)
  simply get the normal card without the extra row.

## Install

The mod is not in the Windhawk catalogue yet, so install it from this repository:

1. Open [`simple-media-progress-bar.wh.cpp`](simple-media-progress-bar.wh.cpp)
   and click **Raw** (or download the file).
2. In Windhawk, open **Create new mod**.
3. Replace the whole contents with the downloaded file.
4. **Compile mod**, then **Install**.
5. Close and reopen Quick Settings once. The panel is built on demand, so the bar
   appears the next time the panel opens.

This mod has its own id, `simple-media-progress-bar`. An older **Media Progress
Bar** install (`media-progress-bar`, or the even earlier
`quick-settings-media-seek`) is a different mod to Windhawk: remove it before
installing this one, or both would inject into the panel at the same time.

If the panel was already open while the mod was enabled, reopening it is enough;
a `ShellHost.exe` restart is not normally required.

## Settings

None. The bar seeks when the thumb is released, which is what players expect
and what avoids the audible stutter that continuous seeking causes.

## Testing checklist

Layout (no player needed):

- [ ] Open Quick Settings with the media card visible: bar, `0:00` on the left,
      duration on the right, buttons centered underneath.
- [ ] Buttons look exactly as before (same size, spacing, hover circle).
- [ ] Card is 36 DIP taller; the quick action buttons below move down by 36;
      nothing is clipped and no scrollbar appears on a normal-size screen.
- [ ] Repeat at 100 %, 125 %, 150 %, 200 % display scaling.
- [ ] Open/close the panel ~50 times, including by clicking outside and by
      pressing Win+A twice quickly: no flicker, no duplicated bar, no crash.
- [ ] Switch between dark and light mode: the time labels stay readable and
      match the artist line (black in light mode, white in dark mode).

Seeking (Spotify, Edge/YouTube Music, Windows Media Player):

- [ ] Drag: the thumb and the elapsed label follow the pointer; playback jumps
      once, on release, without replaying a fraction of the audio.
- [ ] Click on the track at 75 %: playback resumes near 75 % within a fraction of
      a second and the thumb does not jump back.
- [ ] Pause, then seek: the position is still applied, the card stays paused, and
      the bar does not jump back to the old position, not even after 10 seconds.
- [ ] A track of one hour or longer: the elapsed time keeps the `h:mm:ss` form
      before the one-hour mark, so the label never widens mid-track.
- [ ] Next/previous track: duration and thumb reset correctly, no stray seek.
- [ ] An app with no seek support (e.g. a live stream): no extra row, card keeps
      its normal height.
- [ ] Switch the default media session while the panel is open: bar updates.

## Logs

In Windhawk's mod log you should see, in order:

```
Initializing (process 1234)
Waiting for ControlCenter.dll
ControlCenter discovery: ready
Hook ran: deferred=1 element=1 qr=00000000
Update timer started
Media card base height: 178.0
Seek row inserted in row 2 of 4
Control Center view attached
Panel opened: row=1, height=178
```

`Update timer started` is logged only once per process; the timer is stopped
whenever the panel is closed and started again from the open hook. `Hook ran:`
lines only appear a few times and show that the shell actually reached the
discovery hooks. `ControlCenter discovery: failed` means the internal symbol
names did not match the installed `ControlCenter.dll` build — the mod then
does nothing at all and is harmless. `Media card not ready: region=0 root=0
buttons=1` means the media card was located but one of its named parts was not (the
mod retries, then gives up quietly). `Media card base height` different from 178
is fine; the mod always adds its 36 DIP to whatever the card currently uses.

If the panel stops opening, enable the log, reproduce, and look for
`Slow tick: ... ms` or `Slow panel handling: ... ms`. Those two lines mean a XAML
or window query blocked the shell's UI thread, and the mod is the place to look.
`Panel opened: row=1` with no bar visible means the window opened but the card
was rebuilt without the row. `No panel window recognised on this build` is not an
error: it means the shell build does not expose the window the mod looks for, so
the bar keeps its fast poll rate.

The mod logs nothing else. In particular there is no per-tick or per-frame line:
it is either running normally or it says once that something is missing.

## Notes and limitations

- The card grows by 36 DIP while the bar is shown. On very short screens the rest
  of the panel reflows; that is expected.
- The internal `ControlCenter.dll` entry points are hooked from a thread the mod
  starts, once that module has finished loading and had a full second to
  settle; nothing hooks the module during the shell's own start-up. This
  containment exists because earlier builds that worked during start-up
  killed `ShellHost.exe` a few milliseconds later on every cold start.
- While the panel is closed the mod stops its update timer and touches no
  window or XAML. It still reads the playback position once a second in the
  background, so reopening after a long pause shows the current time. Everything
  wakes up from the events that fire when the panel opens, and the bar is
  filled in before the panel finishes animating in.
- The very first panel open after a reboot may show no bar, because the hooks
  arrive after the panel is already built. It appears on the next open.
- The bar hides itself if the session reports no playback position control, or if
  the timeline is empty (start >= end).
- Seeking goes through `GlobalSystemMediaTransportControlsSession.TryChangePlaybackPositionAsync`.
  Some players apply the seek but never publish the new position back; the mod
  keeps counting from the position it asked for instead of trusting the stale
  timeline, so the bar stays in sync either way.
- Only the arm64 and x86-64 shells ship this XAML; the mod is built for x86-64.

## Project

- `@author` is `n3thshan`, `@github` is `https://github.com/n3thshan`.
- `@id` is `simple-media-progress-bar`; the source file
  (`simple-media-progress-bar.wh.cpp`) and project folder match it. Earlier
  builds used `media-progress-bar`, which is why Windhawk treats this one as a
  separate mod.
- `@version` is `1.0.0`.
- Windhawk compiles the single `.wh.cpp` file, so there is no separate build
  step.
- The mod contains no personal paths or machine-specific values.
- Issues and log reports: <https://github.com/n3thshan/wh-simple-media-progress-bar/issues>.

## License

GPL-3.0-only. See [`LICENSE`](LICENSE).
