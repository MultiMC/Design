# Theming overhaul

Notes for reworking how the Launcher looks: which widget style and palette it uses on each
platform, following the system's dark mode and accent colour, themed title bars, and the bugs
around all of it. What is decided, what is proven, and what is still open.

## Goals

- One consistent look by default: our own palettes on Fusion, the same on every platform,
  except where a platform's own style is known to be good (see [Per platform](#per-platform)).
- Follow the system: a System theme that switches between bright and dark with the OS, live,
  and an accent colour that can follow the OS too.
- Title bars that match the theme, not a white bar over a dark window.
- Theme switches that apply fully and immediately, to every window.
- Readable text on every theme, measured rather than guessed.
- Fusion is decent but dated. It stays the default for now; a style of our own may replace it
  later (see [Our own style](#our-own-style)).

## Where things are

- **Done:** Rainbow (the KDE colour library) is replaced by `launcher/Colors.{h,cpp}`
  (Launcher ce4376b3): WCAG luminance and contrast, `mix`, and `readable()`, which makes a
  colour just light or dark enough for a contrast target (4.5:1 by default). Log colours go
  through it.
- **Proven in a proof of concept:** `ThemeTest/` (next to the Launcher checkout; built with
  `hopper/launcher/run-build.py win32 ThemeTest <output dir>`). Verified on Windows 11 on
  2026-11-07, see [ThemeTest](#themetest). Not tested on Windows 10 (no machine).
- **Next:** decide macOS by trying both options; then port ThemeTest's parts into the
  Launcher.

## The current theme system

`launcher/ui/themes/`: each theme is an `ITheme` with a Qt style name (`qtTheme()`), an
optional palette and an optional stylesheet. `ITheme::apply` sets the style, then the palette,
then the stylesheet. Themes:

- **System:** the platform's own style (windowsvista on Windows, macintosh on macOS, whatever
  the Linux desktop gives) with its standard palette.
- **Bright, Dark:** Fusion with our palettes; Dark adds a tooltip stylesheet.
- **Custom:** `themes/<id>/theme.json` and `themes.css` in the data folder: a palette, a
  stylesheet and a free choice of widget style.

Icon themes (pe_dark, pe_light, flat, ...) are chosen separately from the widget theme, so a
dark theme can end up with dark icons.

## Per platform

The default everywhere is Fusion with our palettes (Bright, Dark, System following the OS),
the accent setting, and title bars following the theme. Exceptions:

- **Windows:** no exception. The native windowsvista style is light-only in Qt 5 and ignores
  most of the palette (buttons, checkboxes, progress bars and scroll bars come from Windows'
  visual styles), so it can't do dark mode or a custom accent. It could stay as an opt-in
  "Native" theme, with title bars left to Windows.
- **Linux:** the desktop's own style only where it is known to be good and actually present:
  Breeze on KDE-like desktops (the Plasma platform theme is loaded and the style exists), and
  possibly Kvantum (whoever installed it chose a theme on purpose). Everything else gets
  Fusion: Qt's GTK integration varies too much between distributions.
- **macOS: open, to be decided by trying both.**
  - *Fusion*, like everywhere else. The title bar needs a small Objective-C++ shim: the
    window's `NSAppearance` (aqua or dark aqua) from the palette's brightness, or a
    transparent title bar over the window's background colour for the palette's colour.
  - *The native macintosh style.* Qt 5.15 follows dark mode with it, and it looks like a Mac
    app. Our Bright and Dark themes would then force the app's appearance
    (`NSApp.appearance`) rather than set palettes; custom palettes on top of it look odd.
  - ThemeTest has a style switch on macOS to compare the two on a real Mac
    (`run-build.py osx64-5.15.2 ThemeTest <output dir>`).
  - Toolchain note: `@available(macOS ...)` doesn't link with our osxcross toolchain (no
    compiler-rt, so no `__isPlatformVersionAtLeast`). Version checks go through
    `NSProcessInfo isOperatingSystemAtLeastVersion:`, see ThemeTest's `MacOS.h`.

Custom themes become a palette (and a small stylesheet) on top of the default style. Their
free choice of widget style goes away or is limited: most of what follows only works with a
style that uses the palette.

## What depends on the widget style

| Style | Uses the palette | Title bar | Accent |
|---|---|---|---|
| Fusion | fully | follows the palette | everywhere: selections, checks, progress, sliders, links |
| windowsvista | barely | stays light (its palette is the system's light one): leave it to Windows | only where Qt paints from the palette (text selection, item views, links) |
| Windows (classic) | fully | follows the palette | everywhere |
| macintosh | partly, follows the app's appearance | follows the appearance | from the system |
| Breeze (Linux) | its own colour schemes | the window manager's | the desktop's |

Themes should say what they support, for example `ITheme::titleBarStyle()` (leave it to the
system, dark or light only, or the palette's colours) and `ITheme::supportsAccent()`.

## ThemeTest

What the proof of concept does (Qt 5.6.3, the Launcher's win32 toolchain, still starts on
XP):

- **Title bars (Windows).** `DwmSetWindowAttribute`, looked up at run time with
  `GetProcAddress` (`dwmapi.dll` doesn't exist on XP):
  - Windows 10 1809+: `DWMWA_USE_IMMERSIVE_DARK_MODE` (attribute 20, 19 before build 18985):
    dark or light only.
  - Windows 11 (build 22000+): `DWMWA_CAPTION_COLOR`, `DWMWA_TEXT_COLOR`,
    `DWMWA_BORDER_COLOR`: any colour. Caption = the palette's Window, text = WindowText,
    border = the caption 40% darker, so overlapping windows don't merge.
  - Dark or light: dark when the Window colour's WCAG luminance is below 0.179, where white
    and black text have the same contrast.
  - An event filter on the application applies it to every top-level window when it gets its
    native window, is shown, or gets a new palette: our windows, `QMessageBox`, Qt's own
    `QFileDialog`. Native file dialogs aren't Qt windows and keep the system's look.
  - The title bar changes a moment before the window content (DWM redraws it at once, Qt on
    its next paint). Explorer does the same.
- **System theme (Windows).** `AppsUseLightTheme` under
  `HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize`, followed live through
  the `WM_SETTINGCHANGE "ImmersiveColorSet"` broadcast (a native event filter). With the
  System theme, title bars only switch dark or light and keep Windows' colours, so Windows
  draws the accent on them when the user turned that on.
- **Accent, for every theme:** the theme's own, the system's (followed live), or a custom one
  from Qt's `QColorDialog`.
  - Windows' accent and its shades: `AccentPalette` (8 × RGBA, Light3 to Dark3) under
    `HKCU\...\Explorer\Accent`; without it, DWM's `AccentColor` (0xAABBGGRR) and shades of
    our own.
  - Applied like Windows 11 does: Highlight = Light2 on dark backgrounds, Dark1 on bright
    ones; Link = Light3 or Dark2; HighlightedText black or white by luminance.
- **Platform split.** `TitleBar` and `SystemTheme` are interfaces with one file per platform
  (`_win.cpp`, `_other.cpp` stubs), no `#ifdef`s.

## Bugs

Open issues the overhaul fixes or touches, by cause:

- **Theme switching.** [#5686](https://github.com/MultiMC/Launcher/issues/5686): buttons and
  combo boxes keep the old theme's colours. Cause found in ThemeTest: Qt 5.6's Fusion caches
  painted controls in `QPixmapCache` keyed by state and size, not palette
  (`QStyleHelper::uniqueName`). Fix: `QPixmapCache::clear()` in `ITheme::apply`. Newer Qt
  puts the palette in the key. The `setStyleSheet("/* */")` repolish hack in
  `InstanceWindow.cpp` probably belongs to the same family.
- **Contrast in dark themes.** [#5665](https://github.com/MultiMC/Launcher/issues/5665)
  (invisible arrow icon: icon theme chosen apart from the palette),
  [#4086](https://github.com/MultiMC/Launcher/issues/4086) (fixed font colour on Linux),
  [#2476](https://github.com/MultiMC/Launcher/issues/2476) (log search highlight). Hard-coded
  colours that ignore the palette: the black pen in `InstanceView.cpp`, the Legacy FTB and
  ATLauncher list colours.
- **Fusion.** [#1791](https://github.com/MultiMC/Launcher/issues/1791): message box icons are
  Qt's generic ones; a small `QProxyStyle` returning the platform's standard icons would fix
  it.
- **Native styles.** [#1790](https://github.com/MultiMC/Launcher/issues/1790): wrong dialog
  backgrounds on macOS with the System theme.
- **Title bars and following the system.**
  [#5783](https://github.com/MultiMC/Launcher/issues/5783) (dark title bar),
  [#3899](https://github.com/MultiMC/Launcher/issues/3899) (Windows 10 dark theme: the System
  theme).
- **Scaling and DPI** (a separate track, mostly Qt 5.6's high-DPI limits):
  [#5791](https://github.com/MultiMC/Launcher/issues/5791),
  [#3666](https://github.com/MultiMC/Launcher/issues/3666),
  [#3380](https://github.com/MultiMC/Launcher/issues/3380),
  [#2747](https://github.com/MultiMC/Launcher/issues/2747),
  [#2206](https://github.com/MultiMC/Launcher/issues/2206),
  [#4305](https://github.com/MultiMC/Launcher/issues/4305).

## What our Qt versions allow

| | Windows (Qt 5.6.3, XP) | macOS (Qt 5.15.2) | Linux (the distribution's Qt 5) |
|---|---|---|---|
| Following dark/light | no API: our registry reader | the platform theme follows the appearance; we read it for Fusion | the platform theme if any; the portal's `color-scheme` over D-Bus |
| Accent | registry | `NSColor.controlAccentColor` (10.14+) | portal `accent-color`, newer desktops |
| Title bars | DWM (proven) | `NSWindow.appearance`, or a transparent title bar | X11: the window manager's (Mutter respects `_GTK_THEME_VARIANT=dark`, KDE has a per-window colour scheme property); Wayland: Qt 5's own decorations ignore the palette |
| Known problems | Fusion's cache bug; integer-only high-DPI scaling; per-monitor DPI changes | | depends on the version |

## What even current Qt limits

Qt 6 removes much of the platform code (versions from memory, to verify when upgrading):

- `QStyleHints::colorScheme()` and change notification (6.5): replaces `SystemTheme`'s
  bright/dark part.
- `QPalette::Accent`, read from the system (6.6): replaces the accent reading.
- `QStyleHints::setColorScheme()` (6.8): forces dark or light per app, including the Windows
  title bar.
- The `windows11` style (6.7, Windows 11 only): native-looking, with dark mode.
- Fractional high-DPI scaling that works.

What stays ours even then:

- Custom title bar and border colours: Qt only does dark or light, so the DWM colour code stays.
- Fusion is still the only cross-platform style Qt ships.
- Stylesheets still fight styles and palettes and are slow: themes should rely on palettes.
- Title bars on Wayland are up to the compositor or Qt's decoration plugin.

## Our own style

A style of our own means subclassing `QCommonStyle` (or `QProxyStyle` over Fusion) and drawing
every control. Big, but well bounded, and independent of the rest: with Fusion as the default
everywhere, it can replace Fusion later without changing anything else.

## Porting to the Launcher

Roughly in this order, each a release of its own:

1. The #5686 fix (`QPixmapCache::clear()` in `ITheme::apply`).
2. `TitleBar` (Windows, then macOS) and the title bar event filter; the title bar style per
   theme.
3. `SystemTheme` (Windows first) and a System theme that follows it; the current System theme
   becomes "Native" where it is kept.
4. The accent setting (theme's own, system, custom) in the settings.
5. Icon themes following the palette's brightness (a light and a dark variant per set).
6. The Linux whitelist and the macOS decision.
7. Custom themes as palettes on the default style.

Parts marked in ThemeTest's comments go away with a newer Qt.

## Ideas

- A contrast setting (a slider) for log colours: `Colors::readable` already takes the target.
- A title bar colour in custom themes' `theme.json`, for themes whose stylesheet paints window
  backgrounds itself.

## Open questions

- macOS: Fusion or the native style (testing both).
- Which Linux styles go on the whitelist besides Breeze.
- Keep a "Native" theme on Windows at all?
