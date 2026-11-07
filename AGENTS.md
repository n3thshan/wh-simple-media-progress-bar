# AGENTS.md

Instructions for anyone (human or AI) changing this project.

## What this is

A single-file [Windhawk](https://windhawk.net) mod for Windows 11 24H2+ (x86-64).
It injects into `ShellHost.exe` and inserts a real XAML `Slider` plus two time
labels into the media card of the Quick Settings panel, so playback can be seeked
without switching to the media app. The native previous / play / next buttons are
left exactly as they are.

The whole product is one file. There is no build system, no test suite and no CI;
Windhawk compiles the source itself.

## Repository layout

- `simple-media-progress-bar.wh.cpp` — the entire mod (~2,100 lines). The file
  name and `@id` are the same; the project folder is
  `wh-simple-media-progress-bar`. The commands below name the file, so renaming
  it again is a three-place change (file, `README.md` install step, the two
  commands here).
- `README.md` — user-facing install, testing, log and project notes.
- `AGENTS.md` — this file.
- `LICENSE` — GPL-3.0-only, required by the Windhawk catalogue. Do not edit it.
- `assets/quick-settings.png` — the screenshot used by `README.md` and the
  `==WindhawkModReadme==` block. Cropped to the media card so the rest of the
  panel is not published.

The project is a git repository: branch `main`, remote
`https://github.com/n3thshan/wh-simple-media-progress-bar.git`. Only the five
files above are tracked; there are no build artefacts or generated files.

Read `README.md` before changing behaviour: it is the user-facing contract —
install flow, what each log line means, and project notes. Keep it in sync with
the source.

## Environment

- Windows 11, build **26300** on the user's machine, x86-64. The mod targets 24H2
  and newer (`@include ShellHost.exe`, `@architecture x86-64`).
- Windhawk **1.7.3**. The mod id is `simple-media-progress-bar`; installed locally
  that becomes `local@simple-media-progress-bar`, which is the
  `[local@simple-media-progress-bar]` prefix on every log line. Changing `@id`
  makes Windhawk treat the file as a different mod, so an older entry has to be
  removed by hand. That happened once already: the id was `media-progress-bar`
  (and `quick-settings-media-seek` before that), so logs with either old prefix
  describe an older build.
- The symbol hooks are mangled names taken from `ControlCenter.dll` of that build
  (verified against `%TEMP%\ControlCenter.pdb`). A different Windows build can
  change them; `ControlCenter discovery: failed` then means the mod did nothing
  and is harmless.
- The user sends logs from **DbgViewMini**, which listens to `OutputDebugString`.
  It shows only the shell's own debug output plus ours; the released build has no
  exception tracer, so a `Fault:` line cannot come from this mod.
- The user installs the mod themselves: *Create new mod* → paste the whole file →
  *Compile mod* → *Install* → close and reopen Quick Settings once.

## Ground rules

1. **Keep it one file.** No `src/`, no headers of our own, no build scripts, no
   CMake. Windhawk compiles and ships exactly one translation unit.
2. **No new dependencies.** The includes at the top of the file are the full set
   (`windows.h`, `unknwn.h`, `windhawk_utils.h`, C++/WinRT XAML + GSMTC headers,
   a few standard headers). Adding a library means changing `@compilerOptions`
   and the link check — ask first.
3. **No settings.** The mod deliberately has none, and the `continuousSeek`
   ("seek while dragging") setting was removed after user feedback. Do not add it
   back.
4. **Do not touch the native transport buttons.** Not their style, not their
   position, not their templates. The bar lives in its own row above them.
5. **Do not rename log lines the README quotes** without updating `README.md` in
   the same change.
6. **Do not bump `@version`** except as part of a release: `1.0.1` for a fix,
   `1.1.0` for a feature. `@id` and `@name` never change once the mod is in the
   Windhawk catalogue — the 2026-11-06 rename was a deliberate, user-requested
   exception, recorded under *Current state*, not a precedent.

## Verifying a change

There is no test runner. Two commands are the only automated gates, and both must
print `0`. They mirror Windhawk's own compiler invocation.

Syntax check (strict warnings):

```powershell
cd <repo>
& "C:\Program Files\Windhawk\Compiler\bin\clang++.exe" "@C:\Program Files\Windhawk\Compiler\compile_flags.txt" "@$env:TEMP\whflags.txt" -Wall -Wextra -fsyntax-only simple-media-progress-bar.wh.cpp; "syntax=$LASTEXITCODE"
```

Link check (catches undefined symbols that `-fsyntax-only` cannot):

```powershell
& "C:\Program Files\Windhawk\Compiler\bin\clang++.exe" "@C:\Program Files\Windhawk\Compiler\compile_flags.txt" "@$env:TEMP\whflags.txt" -shared -O2 -o "$env:TEMP\qsm-check.dll" simple-media-progress-bar.wh.cpp "$env:TEMP\whstub.cpp" -lole32 -loleaut32 -lruntimeobject -luuid -luser32; "link=$LASTEXITCODE"
```

Notes:

- `C:\Program Files\Windhawk\Compiler\compile_flags.txt` ships with Windhawk and
  is authoritative. Do not edit it.
- `%TEMP%\whflags.txt` contains the three extra defines Windhawk adds:
  `-DWIN32_LEAN_AND_MEAN`, `-UWH_EDITING`,
  `-DWH_MOD_ID=L\"simple-media-progress-bar\"`. If it is missing, recreate it.
- `%TEMP%\whstub.cpp` stubs Windhawk's runtime API purely so the DLL links. It is
  never shipped and never included by the mod.
- **A clean compile is not proof the mod works.** Only the user can install it in
  Windhawk and report what the panel does. Never describe a change as "working"
  on the strength of a successful build.

If `%TEMP%\whstub.cpp` is missing (temp cleanup), recreate it exactly like this.
It is a local link-check stub only: never shipped, never included by the mod.

```cpp
// Local link-check stub for Windhawk's runtime API (never shipped).
#include <windows.h>
#include <cstdarg>
#include "windhawk_api_internal.h"

extern "C" {
BOOL InternalWh_IsLogEnabled(void*) { return FALSE; }
void InternalWh_Log(void*, PCWSTR, va_list) {}
int InternalWh_GetIntValue(void*, PCWSTR, int d) { return d; }
BOOL InternalWh_SetIntValue(void*, PCWSTR, int) { return FALSE; }
size_t InternalWh_GetStringValue(void*, PCWSTR, PWSTR, size_t) { return 0; }
BOOL InternalWh_SetStringValue(void*, PCWSTR, PCWSTR) { return FALSE; }
size_t InternalWh_GetBinaryValue(void*, PCWSTR, void*, size_t) { return 0; }
BOOL InternalWh_SetBinaryValue(void*, PCWSTR, const void*, size_t) { return FALSE; }
BOOL InternalWh_DeleteValue(void*, PCWSTR) { return FALSE; }
size_t InternalWh_GetModStoragePath(void*, PWSTR, size_t) { return 0; }
int InternalWh_GetIntSetting(void*, PCWSTR, va_list) { return 0; }
PCWSTR InternalWh_GetStringSetting(void*, PCWSTR, va_list) { return L""; }
void InternalWh_FreeStringSetting(void*, PCWSTR) {}
BOOL InternalWh_SetFunctionHook(void*, void*, void*, void**) { return FALSE; }
BOOL InternalWh_RemoveFunctionHook(void*, void*) { return FALSE; }
BOOL InternalWh_ApplyHookOperations(void*) { return FALSE; }
HANDLE InternalWh_FindFirstSymbol4(void*, HMODULE, const WH_FIND_SYMBOL_OPTIONS*, WH_FIND_SYMBOL*) {
    return nullptr;
}
BOOL InternalWh_FindNextSymbol2(void*, HANDLE, WH_FIND_SYMBOL*) { return FALSE; }
void InternalWh_FindCloseSymbol(void*, HANDLE) {}
BOOL InternalWh_HookSymbols(void*, HMODULE, const WH_SYMBOL_HOOK*, size_t,
                            const WH_HOOK_SYMBOLS_OPTIONS*) {
    return FALSE;
}
BOOL InternalWh_Disasm(void*, void*, WH_DISASM_RESULT*) { return FALSE; }
const WH_URL_CONTENT* InternalWh_GetUrlContent(void*, PCWSTR, const WH_GET_URL_CONTENT_OPTIONS*) {
    return nullptr;
}
void InternalWh_FreeUrlContent(void*, const WH_URL_CONTENT*) {}
}
```

## Where things live

Line numbers drift; search for the function name.

| Area | Functions |
| --- | --- |
| XAML helpers | `ClassOf`, `Find`, `ParentOf`, `FormatTime` |
| `namespace media` | `Snapshot`, `Get`/`Store`, `Refresh` (timeline + position tracking), `Projected` / `ResetTracking` / `ClampPosition`, `ApplySeek`, `Wake`/`RequestSeek`, `ThreadProc`, `Start`/`Stop`, `RememberThread` / `StopOnCurrentThread` |
| `class Patcher` (lifecycle) | `Offer`, `Restore`, `StartTimer`/`StopTimer`/`OnTimer`, `Tick`, `Discover`, `Base`, `ProcessPending`, `Reconcile`/`ReconcileCore`, `ReportMissing` |
| `class Patcher` (card) | `AddColumn`, `BuildRow`, `Insert`, `ApplyHeight`, `SetRowVisible`, `Commit`, `Update`/`UpdateCore`, `SyncTimeBrush`/`FallbackBrush`, `WindowShown` / `QueryHostWindows` / `HostVisible` |
| Discovery hooks | `Discovered`, `NoteSlowDiscovery`, `InstallProc`, `Hook*Focus`, `Hook*Connect`, `HookControlCenter` |
| Entry points | `Wh_ModInit`, `Wh_ModUninit` |

## How it works

1. `Wh_ModInit` starts a small installer thread that watches for
   `ControlCenter.dll` with `GetModuleHandleW` every 250 ms. Once the module is
   there it waits one more second, then calls `HookSymbols` for the four mangled
   `ControlCenterView` / `FullScreenPage` interface methods. `OnGotFocus` is the
   required trigger; the two `Connect` hooks are optional and only arrive
   earlier. Nothing else happens at start-up: there is no `LoadLibraryExW` hook,
   and the media thread is started by the first discovery.
2. A hook hands the live `FrameworkElement` to a per-UI-thread `Patcher`. The
   patcher starts a 200 ms Win32 timer and, on the UI thread, walks the tree to
   the media card (`MediaTransportControlsRegion` → `Root` → `MediaButtonsListView`).
   The timer is **stopped** after two consecutive ticks with the panel
   closed/cloaked (`HostVisible()` false); the media thread drops to a 1 s
   refresh and writes no XAML work. The open hooks
   (`OnGotFocus`, and `Connect` indirectly) restart the timer, wake the media
   thread and paint one frame synchronously via `Patcher::WakeUp()`.
3. The seek row is inserted at the button row's index (currently 2 of
   `[Auto, Auto, *]`): children at or below that index are shifted down, a new
   `Auto` `RowDefinition` is added, and the row is appended.
4. The card's base height is read at runtime (`~178`) and `kCardGrowth` (32 bar +
   4 gap = 36 DIP) is added. Never hardcode either number. While the row is
   collapsed the card is on its natural auto-size and `baseHeight_` is re-learned
   from `ActualHeight`, so a wrapped title or a differently sized app card adapts
   instead of staying locked to the first measurement. `Restore()` gives the
   card `Height(NaN)` back.
5. A dedicated MTA thread owns every GSMTC object and writes a plain `Snapshot`
   behind a mutex on a 200 ms cadence. It is started by the first successful
   discovery (`g_mediaStartFlag`), never by `Wh_ModInit`, so COM and GSMTC are
   not brought up inside the shell's start-up window. The UI thread only reads
   that snapshot and posts seeks through an atomic + event. `g_active` sets
   the 200 ms poll rate; `g_watch` stays true after discovery, so the thread
   keeps a 1 s refresh while the panel is closed. `Restore()` clears both.
6. Position is projected from the last accepted timeline sample onto a monotonic
   baseline (`g_basePos` / `g_baseTick`), with explicit handling for seeks, stale
   post-seek samples, pauses, resumes, track changes and clock/sleep gaps.

## Visual constants are deliberate

`kBarHeight` 32 DIP, `kRowGap` 4 (together the 36 DIP the card grows), 8 DIP
gutters beside the labels, 12 px labels, 34 DIP minimum label width. These were
matched against the native card so the bar inherits its spacing and text scale and
the 48 DIP button row stays untouched. Do not restyle them to "rounder" numbers.
If a visual change is requested, compare at 100 %, 125 %, 150 % and 200 % scaling
against a screenshot of the card without the mod.

## Rejected approaches

Each of these was tried or considered and did not work. Do not reintroduce them.

| Idea | Why not |
| --- | --- |
| Hook `LoadLibraryExW` to notice `ControlCenter.dll` | Puts our code in the path of every library the shell loads, and symbol work inside that hook patched a module the loader had not finished with. ShellHost aborted ~2 ms after every cold start. Polling `GetModuleHandleW` from our own thread replaced it. |
| Any work in `Wh_ModInit` beyond starting the installer thread | COM, GSMTC, exception handlers and system-function hooks were all present in versions that crashed the shell during start-up. See invariant 15. |
| A separate overlay window above the panel | Clicking it dismisses Quick Settings; an outside click closes the flyout. Injecting into the card is the only stable surface. |
| `Windows.UI.Xaml.DispatcherTimer` for the update loop | Never fired inside `ShellHost.exe`; the bar looked frozen. A Win32 thread timer is used instead. |
| `TextFillColorSecondaryBrush` lookup for the labels | Resolves against the shell's application theme, not the card's; produced white text in light mode. |
| Seeking from `ValueChanged`, or while dragging | Double seeks and an audible replay of a fraction. Commit once, on release. |
| Hardcoding the card height (178 / 214) | Varies by build, scaling and whether the card is expanded. Read `ActualHeight`, add 36. |
| Editing the native transport buttons | They must keep their exact style and position; the mod only adds a row. |

## Invariants

Each of these exists because ignoring it caused a crash, a freeze, or a wrong
label. Do not "simplify" them away.

1. **Hooks are `noexcept`, and their bodies are wrapped in `try/catch`.** An
   exception escaping into the shell takes down `ShellHost.exe`; the visible
   symptom is "Quick Settings no longer opens".
2. **Never touch XAML from `Connect`.** That target is half-constructed. Store a
   weak reference and let the timer do the work.
3. **Adopt deferred candidates only once `IsLoaded()`.** Same reason.
4. **`Discover` / `Reconcile` may run only from `OnGotFocus` or the timer.**
   `LayoutUpdated` may only set `needsReconcile_`; doing work there rebuilds
   inside a layout pass.
5. **Parent climbs are bounded** (64 levels). A malformed or half-built tree must
   not loop forever.
6. **No COM or XAML call from any thread other than the one that owns the
   object.** GSMTC calls belong to the media thread; XAML calls to the UI thread.
7. **Elements the mod creates hold strong references** (`elapsedTime_`,
   `totalTime_`, `subtitle_`). On this shell build a weak reference to a
   code-created `TextBlock` resolves to null silently — that was the "labels stuck
   at `0:00`" bug. Weak refs are only correct for elements the shell owns.
8. **Height is written only when it actually changes** (`ApplyHeight` reads
   `region.Height()` first). Unconditional writes caused visible flicker.
9. **The two labels are written independently of the slider.** One unreachable
   control must never disable the other.
10. **A seek is never committed from `ValueChanged`.** It is committed on pointer
    release, or from the next tick when no drag started; identical seeks within
    500 ms are dropped. Committing from `ValueChanged` double-seeks and the player
    replays a fraction of audio.
11. **Do not lower the tick interval or do heavy work in `Tick`.** It runs inside
    the shell's UI dispatch. `Slow tick` / `Slow panel handling` in the log mean
    the mod is blocking the panel.
12. **Theme colour comes from the card, not from a resource lookup.**
    `SyncTimeBrush` mirrors `SubtitleText` → `TitleText` → a computed 60 %
    brush based on `ActualTheme`. A `TextFillColorSecondaryBrush` lookup resolves
    against the shell's application theme and produced white labels in light mode.
13. **Refuse an unexpected card layout.** If the button row is not where it is
    expected, log and leave the card untouched instead of guessing.
14. **Shutdown order matters:** `g_stopping` → stop the media thread → stop each
    patcher through its own dispatcher, and join `InstallProc` before the module
    can be unloaded. The patcher object is deliberately never destroyed by the
    CRT.
15. **Keep the shell's start-up free of this mod.** ShellHost aborted ~2 ms after
    every cold start while the mod did start-up work in that window: the abort
    stack was `ControlCenter.dll`'s own initialisation and none of our frames, the
    crash started *before* `Applying hooks`, and disabling the mod stopped it. A
    `LoadLibraryExW` hook made it worse by putting our code in the path of every
    library the shell loaded, and a symbol install inside that hook patched a
    module the loader had not finished with. Current containment: no hook on any
    system function, `InstallProc` polls `GetModuleHandleW` — which needs the
    loader lock, so it only ever reports a module that has finished loading — and
    waits ~1 s more before touching `ControlCenter.dll`, and `media::Start()` runs
    on the first discovery instead of at init. Adding anything to `Wh_ModInit` is
    a regression until it is proven not to reintroduce the abort.

## Logging

- `Wh_Log` is `wprintf`-style: `%s` for wide strings, `%lld`, `%08X` for HRESULTs.
- Every line on a hot path must be rate-limited with a `static std::atomic<int>`
  counter. ShellHost logs for the whole session.
- Keep the existing wording. `README.md` quotes several lines and explains them to
  users; a rename is a documentation change.
- The set of lines is deliberately small, and every one of them is quoted in
  `README.md` or needed for triage: the bounded discovery lines (`Hook ran`,
  `ControlCenter discovery`), insert, height, `Media card not ready`,
  `Panel opened`, the slow-tick warnings and the one-off failure lines. There is
  no per-tick or per-frame logging; adding some back is a debugging change, not a
  cosmetic one, and it must be removed again before a release.

## Diagnosing a broken panel

| Symptom | Look for | Meaning |
| --- | --- | --- |
| Quick Settings does not open, or a new `ShellHost.exe` keeps appearing | `Slow panel handling`, `Slow tick`, and the Windows event log (`Application`, Id 1000) | The mod is blocking the UI thread, or the shell is failing on its own. Check whether crashes stop with the mod disabled before changing anything. |
| No bar, card otherwise normal | `ControlCenter discovery: failed` | Mangled symbol names did not match this `ControlCenter.dll`. The mod is inert. |
| No bar, discovery is `ready` | `Media card not ready: region=… root=… buttons=…` | The card was found but a named part was not; the flag that is `0` says which. |
| Bar disappeared after the card rebuilt | `Panel opened: row=1` with nothing visible, or `Unexpected media card layout` | Re-insert was refused or failed. |
| Bar shown, times stuck at `0:00` | no log line needed | Regression of the strong-reference rule, invariant 7. |
| Bar never updates | `Media thread failed`, `No media session manager available` | GSMTC is unavailable in this process. |
| Values wrong or jumping | add a temporary `state:` line | No timeline, no duration, or no position control; compare with the player. |
| Work continues while the panel is closed | `No panel window recognised on this build, polling continuously` | Visibility detection fell back to "always visible"; a cost only, not a bug. |

## Code style

- 4 spaces, no tabs. Keep lines under ~100 columns; comments wrap near 78.
- Namespace aliases are fixed: `X`, `C`, `P`, `M`, `GSMTC`.
- Comments explain *why* — history, edge cases, what broke before. Do not narrate
  what the next line plainly does.
- Prefer `static_cast`; Win32 types (`DWORD`, `HANDLE`, `ULONGLONG`) where the API
  needs them.
- No emoji, no personal paths, no machine names, no TODO markers left behind.

## Working with the user

- The user is technically minded but not a developer. Explain behaviour in plain
  language, not in code terms. Ask for a log rather than guessing.
- **Always check the `Mod version:` line in a pasted log before analysing it**, and
  the `[local@…]` prefix with it: three mod ids exist on this machine at most
  (`quick-settings-media-seek`, then `media-progress-bar`, now
  `simple-media-progress-bar`), and a log from an older one says nothing about
  the current build. Stale logs have wasted entire debugging rounds.
- A change is "done" when the user reports it working, not when it compiles.
- Behaviour that was already agreed with the user should not change silently.

## Publishing

The source header is the published artefact: the single `.wh.cpp` file is what
goes to the Windhawk catalogue (the mods repository takes one file per mod).
The GitHub repository is
`https://github.com/n3thshan/wh-simple-media-progress-bar` and holds the same
file plus `README.md`, `AGENTS.md`, `LICENSE` and `assets/quick-settings.png`.

- `@id` `simple-media-progress-bar` and `@name` **Simple Media Progress Bar** are
  final. The id is what Windhawk, the catalogue URL and the log prefix
  (`[local@simple-media-progress-bar]` locally) are built from; changing it again
  creates yet another mod rather than updating this one. The rename from
  `media-progress-bar` already did that, so the old id is a separate installed
  entry that has to be removed by hand.
- `@author` `n3thshan`, `@github` `https://github.com/n3thshan`,
  `@license` GPL-3.0-only. The catalogue requires the licence; do not relax it.
- Keep the `==WindhawkModReadme==` block short and accurate: it is what users read
  in Windhawk for this mod, not a copy of `README.md`.
- The Windhawk readme embeds the screenshot from
  `https://raw.githubusercontent.com/n3thshan/wh-simple-media-progress-bar/main/assets/quick-settings.png?v=2`
  because Windhawk only allows images from `i.imgur.com` and
  `raw.githubusercontent.com`. The `?v=2` suffix bypasses GitHub's raw CDN
  cache; bump it when the image changes. The GitHub readme uses the relative
  `assets/quick-settings.png` path instead.
- No personal paths, machine names or user-specific values anywhere in the file.

Before a release:

1. Bump `@version` (`1.0.1` fix, `1.1.0` feature) and nothing else about the id.
2. Re-run both checks in *Verifying a change*; both must print `0`.
3. Refresh the `README.md` log section if any line it quotes changed.
4. Smoke test: reboot, open Quick Settings ~10 times, bar and both labels present,
   and no new `ShellHost.exe` Id=1000 events in `Application`.

## Current state (2026-11-07)

**Version 1.0.0 — verified on this machine. `@version` was deliberately not
bumped; both compile gates print `0`.**

Renamed 2026-11-06, at the user's request: `@name` is **Simple Media Progress
Bar**, `@id` is `simple-media-progress-bar`, and the source file, project folder
and both documents follow. No behaviour changed, so `@version` stays `1.0.0`:
Windhawk has never seen this id and will treat the build as a first release. The
old `media-progress-bar` entry stays installed until the user removes it, and
both mods must never be enabled at the same time.

Applied 2026-11-06 (verified by the user on 2026-11-07):

- UI idle while hidden: the 200 ms timer stops after two consecutive hidden
  ticks (`HostVisible()` false) and `g_active` is cleared, so no XAML or window
  queries run while the panel is closed. The media thread keeps a 1 s GSMTC
  refresh (`g_watch` stays true) so the position is current at reopen. The open
  hooks and `Offer()` restart the timer, wake the media thread and paint one
  frame synchronously (`Patcher::WakeUp()`). `Tick()` also skips
  `ProcessPending()` while hidden.
- Panel reopen after a long closure: `WakeUp()` resets `hiddenTicks_` and forces
  one paint (`forceUpdate_`) even while DWM still reports the window as cloaked.
  `Refresh()` adds the sample age even after a polling gap (`!woke` removed), so
  a closure of 10+ s cannot reset the position to `0:00`.
- Paused seek: `knownPlayingBaseline` now requires `next.playing`, and a seek
  the player never confirms is no longer reverted while paused (`g_seekTarget`
  is kept as `g_basePos` until a genuinely new, distinct sample arrives, or the
  track changes). Stale-sample rejection also applies after the 10 s timeout
  while paused.
- Flicker/height: the row is inserted collapsed and the card is grown only when
  the session is verified seekable; the collapsed card is on natural auto-size
  (`Height(NaN)`) with `baseHeight_` re-learned, and `Restore()` writes
  `Height(NaN)` instead of the base height.
- Boot delay: `Wh_ModInit` no longer hooks `ControlCenter.dll` directly; only
  `InstallProc` installs, always after `Sleep(1000)`, with a clean exit if
  `g_stopping` lands during the sleep.
- `FormatTime(ticks, forceHours)`: a track of an hour or longer keeps `h:mm:ss`
  for the elapsed label below the mark. Label brushes compare A,R,G,B (not the
  brush pointer) so in-place theme mutations are detected.
- `Wh_ModAfterInit` is gone; the entry points are `Wh_ModInit` and
  `Wh_ModUninit`.

The mod does what it set out to do: a real XAML slider plus two labels in the media
card, seeking on release, the native buttons untouched. Confirmed by the user on
this machine: panel opens after a reboot, bar present on the first and every later
open, times correct, height switching 214 ↔ 178, light-mode label colour, seek to
the far left restarts the track, no settings.

The one hard-won lesson is invariant 15: the shell's start-up window must stay
empty of this mod. Two cold starts with zero `ShellHost.exe` Id=1000 events are the
evidence that 1.0.0's structure works, and 1.0.0 does *less* at start-up than the
build that was tested (the crash tracers are gone). Do not add anything to
`Wh_ModInit` without repeating that test.

## Still needed from the user

- Multi-player test: Spotify, Edge-tab YouTube Music, Windows Media Player, and a
  live stream. Nothing else is outstanding.

Decisions already taken, so they are not re-opened by accident:

- `@author` is `n3thshan`, `@github` is `https://github.com/n3thshan`;
  **GPL-3.0-only** stays as declared.
- The mod is named **Simple Media Progress Bar**, id
  `simple-media-progress-bar` (renamed from **Media Progress Bar** /
  `media-progress-bar` on 2026-11-06, at the user's request; the id change is
  accepted, so the old Windhawk entry is removed rather than migrated). The
  repository is public; the Windhawk catalogue submission is the next publishing
  step.
- Seek to the far left: the players tested restart the track, so the
  FluentFlyout seek-to-1-second workaround is **not** needed.
- A session with a duration but no position control (`valid=1 seek=0`): the row
  stays hidden. The user chose this over a read-only progress bar.
- Seeking happens on release only; there is no "seek while dragging" setting.

## Reference material

[FluentFlyout](https://github.com/unchihugo/FluentFlyout) — a GPL-3.0-or-later
WPF app that uses the same GSMTC API and the same seek-bar design (poll, project
forward, commit on release, +36 px). Useful to check assumptions; not a template.

- Worth learning from: the seek-to-zero workaround; the deliberate decision not
  to require `IsPlaybackPositionEnabled`; the same 36 DIP growth.
- Do **not** copy: reading timeline properties on the UI thread, using
  `MaxSeekTime` as the duration, treating `position > EndTime` as "closed", or
  omitting capture-lost handling. Those are safe only in its own process and are
  bugs in ours.

The published *Refined Quick Settings and Sound* mod in the
[`windhawk-mods`](https://github.com/ramensoftware/windhawk-mods) repository,
`mods/refined-quick-settings-and-sound.wh.cpp`, used as the structural reference
for discovery (the rule that a `Connect` target must never be queried while it is
half-built comes from there). The invariants above carry the lesson.

Other useful, volatile paths: `%TEMP%\whflags.txt`, `%TEMP%\ControlCenter.pdb`
(verified symbol names), and `C:\Program Files\Windhawk\Compiler\include\winrt\base.h`
(the `weak_ref` implementation that explains invariant 7).

## Don'ts

- Do not run, install, or edit the user's installed Windhawk mods.
- Do not rename the public element names (`WindhawkSeekRow`, `WindhawkSeekSlider`,
  `WindhawkSeekElapsed`, `WindhawkSeekTotal`) without a reason.
- Do not reformat or rewrite working sections as part of an unrelated change.
- Do not add a settings page, an options UI, or a config file.
- Do not replace symbol-based discovery with window-class or process scanning.
