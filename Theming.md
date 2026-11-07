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
- **Ported into the Launcher** (2026-11-07): #5686 (a25e3bc4), title bars (9d18ee2b), then
  the rest in one change: the system's dark mode and accent on all three platforms, the
  System and Native themes, the accent setting, the Linux whitelist, Fusion as the macOS
  default, custom themes as palettes, and new theme settings with a selector (see
  [In the Launcher](#in-the-launcher)).
- **Next:** testing on real machines; icons (see [Ideas](#ideas)).

## The theme system before the overhaul

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

- **Windows:** Fusion only, no Native option (decided 2026-11-07, after trying windowsvista
  with Qt 5.6 and 5.15 in ThemeTest). windowsvista is light-only in Qt 5 and ignores most of
  the palette (buttons, checkboxes, progress bars and scroll bars come from Windows' visual
  styles): no dark mode, no accents, and its progress bars stay Windows' green next to
  blue-accented controls. Windows has no single look of its own to match anyway: layers of
  toolkits and their versions side by side.

**Native on macOS and Linux** (decided 2026-11-07): the platform's own style, whatever Qt
starts with, with what that style can take. Known styles:

| Native style | Bright, Dark, Red | Accent | Custom accent | Default |
|---|---|---|---|---|
| Breeze, Kvantum | our palettes (palette-driven, like Fusion) | yes | yes | yes |
| macintosh | Bright and Dark force the app's appearance | the system's, natively | no: AppKit draws checks and buttons with the system accent | no |
| any other (a GNOME desktop's) | no: the platform's look as it is | no | no | no |

What a style can't take is disabled, not hidden, and the choices are kept: switching back to
Fusion brings them back. Handing the palette back to the platform differs by the Qt we run
with: 5.15 gives the platform's back for a palette with no roles set (live, with its
per-class palettes); older Qt copies the current palette instead (seen with 5.6 on Windows,
before Native was dropped there: Dark's window colour and the accent stayed), so there the
palette the platform gave at the start is set back. Checked at run time (`qVersion()`): Linux
builds run on the system's Qt, which can be older than 5.15. An unknown style gets the
platform's look as it is: we don't know what it does with a palette. ThemeTest:
`nativeAbilities()` in `Themes.cpp`.
- **Linux:** the desktop's own style only where it is known to be good and actually present:
  Breeze on KDE-like desktops (the Plasma platform theme is loaded and the style exists), and
  possibly Kvantum (whoever installed it chose a theme on purpose). Everything else gets
  Fusion: Qt's GTK integration varies too much between distributions.
  - On those desktops Native is the default, and must work completely: with the System theme
    the application palette is the platform theme's, all of it (the desktop's colour scheme,
    accent and changes, live), so we set none of our own. Qt 5.15 gives it back when an
    empty palette (no roles set) is set. Bright and Dark are our palettes on Breeze, which
    uses the palette fully, like Fusion. A custom accent goes on top as the only roles set,
    so the rest keeps following the desktop. In every colour group the way the desktop does
    it to its own accent: KDE dims an inactive window's selection (`#3daee9` becomes
    `#1b4155`, same hue, about 65% of the saturation and 38% of the lightness) and practically
    removes a disabled one, by colour effects that aren't a plain mix. ThemeTest
    (`followPlatform`) measures the platform's change in HSL saturation and lightness per role
    and group and applies it to the custom accent, keeping its hue; `--dump-palette` prints
    the platform's colours per group.
  - Seen 2026-11-07 on a KDE fork: the Launcher's current System theme and ThemeTest's
    Fusion only followed the portal's dark/light and accent, not the colour scheme itself
    (Golden Honey Oak stayed our Bright). That's the gap Native closes.
  - Whitelisted styles, by the name of the style Qt starts with: `breeze`, `kvantum` (and
    `macintosh`, where Native is an option, not the default).
  - Following the system: the XDG desktop portal (`org.freedesktop.portal.Settings`,
    namespace `org.freedesktop.appearance`): `color-scheme` (0 no preference, 1 dark,
    2 light) and `accent-color` (r, g, b in 0..1), with a `SettingChanged` signal. One
    interface for KDE, GNOME and their forks, whichever portal backend the desktop runs.
    Verified 2026-11-07 on a KDE fork (X11), which the Launcher's current System theme
    doesn't follow: dark and `#3daee9` read fine. ThemeTest's `SystemTheme_linux.cpp`.
  - It needs QtDBus: one more Qt library on Linux (part of qtbase everywhere; Debian's
    `libqt5gui5` already depends on `libqt5dbus5`), so `libQt5DBus.so.5` joins the lin32 and
    lin64 gate expectations when the Launcher gets it. Calls time out after 2 s, so a
    missing or slow portal can't hang the start.
  - Without a portal, there's nothing to follow: the System theme isn't offered.
- **macOS: Fusion by default, the native style as an option** (decided 2026-11-07, after
  trying both in ThemeTest on a Mac: `run-build.py osx64-5.15.2 ThemeTest <output dir>`).
  - The native macintosh style follows dark mode in Qt 5.15 and looks like a Mac app, but
    it brings its own layout metrics (bigger margins and spacing for focus rings, fixed
    button and combo box heights) that our layouts weren't made for: loose in places,
    cramped or clipped in others. Keeping it the default would mean going through every
    dialog on a Mac.
  - Fusion keeps the layouts as designed and tested everywhere. The System theme and the
    system accent make it follow macOS's colours; the title bar follows through a small
    Objective-C++ shim (the window's `NSAppearance`, aqua or dark aqua, from the palette's
    brightness, or a transparent title bar over the window's background colour).
  - The native style stays as a "Native" theme: Bright and Dark there force the app's
    appearance (`NSApp.appearance`), and Qt's native palette follows it.
  - Qt's Cocoa theme keeps the native controls' right colours in palettes per widget class
    (buttons, item views, labels...). Setting an application palette with any roles clears
    them for good (Qt 5.15 `handlePaletteChanged`); only one with no roles set brings them
    back. A custom accent cleared them (black button text, the list ignoring the accent), so
    ThemeTest takes them after handing back to the platform and puts them back with the
    accent in them. Whatever we do, the native style draws checkboxes, radio buttons and
    default buttons with AppKit's system accent: a custom accent would reach selections and
    links only. So with the native style on macOS the custom accent isn't offered (decided
    2026-11-07): the choice is kept, the system's accent is used, and Fusion brings the
    custom one back.
  - Default on a fresh install: the System theme with the system accent.
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
  - Every theme switch also refreshes all title bars itself: Qt sends PaletteChange only
    when a widget's palette really changes, and Dark to System in a dark Windows (or Bright
    to System in a light one) gives the same palette while the title bars must go from the
    palette's colours to the system's. Without it they kept Dark's caption colour.
  - The title bar changes a moment before the window content (DWM redraws it at once, Qt on
    its next paint). Explorer does the same.
- **System theme (Windows).** `AppsUseLightTheme` under
  `HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize`, followed live through
  the `WM_SETTINGCHANGE "ImmersiveColorSet"` broadcast (a native event filter). With the
  System theme, title bars only switch dark or light and keep Windows' colours, so Windows
  draws the accent on them when the user turned that on.
- **The System theme's colours.** Following only dark or light isn't enough: with our Dark
  palette (window `#31363b`) under Windows' own dark title bar (`#202020`) it didn't look
  like the system. So System uses the system's own colours where we know them
  (`SystemTheme::palette()`), and our Bright or Dark elsewhere:
  - Windows: Qt 5.6 only has the classic system colours, light even in dark mode, so
    Windows 11's own values: dark window `#202020`, fields `#2b2b2b`, buttons `#2d2d2d`,
    white text; light window `#f3f3f3`, white fields, text `#1b1b1b`. They also match the
    default title bars, which System leaves to Windows.
  - Our Bright is those light colours too, everywhere: neutral and clean. Fusion's standard
    palette, which it used before, has a warm beige window (239,235,231) that looked
    yellowish next to Windows.
  - macOS: AppKit's semantic colours (`windowBackgroundColor`, `controlBackgroundColor`,
    `textColor`, `alternatingContentBackgroundColors`) resolved in the current appearance.
    Dark mode's alternate row is white at 5% opacity, drawn over the list: it's mixed over
    the field colour, since a palette colour is drawn opaque (it came out solid white in the
    Launcher's first run, 2026-11-07).
  - Linux (Fusion): the portal has no colours, so our Bright or Dark; KDE-like desktops get
    their real colours through Native.
- **Accent, for every theme:** the theme's own, the system's (followed live), or a custom one
  from Qt's `QColorDialog`. The System theme's own accent is the system's.
  - Windows' accent and its shades: `AccentPalette` (8 × RGBA, Light3 to Dark3) under
    `HKCU\...\Explorer\Accent`; without it, DWM's `AccentColor` (0xAABBGGRR) and shades of
    our own.
  - Applied as the desktops do it: Highlight (selections, checks, progress) is the accent
    itself, HighlightedText black or white by luminance. Links are text on Base, so they
    get the accent's hue made readable there (`Colors::readable`, 4.5:1). An earlier
    version used Windows 11's shades (Light2 on dark, Dark1 on bright); it washed accents out
    on dark themes (red became pale pink) and didn't match KDE or macOS, which fill with the
    accent itself.
- **Checking a theme.** An Enabled/Disabled gallery shows every kind of control twice.
  `ThemeTest --screenshots <dir>` saves the window in every theme as PNGs and quits; with
  `QT_QPA_PLATFORM=offscreen` it needs no desktop, so theme changes can be looked at (and
  compared) without clicking through them.
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
- **Our palettes leak the desktop's colours.** The Bright and Dark palettes
  (`BrightTheme.cpp`, `DarkTheme.cpp`) set only some roles; Qt fills the rest (Light,
  Midlight, Mid, Dark, Shadow, PlaceholderText) from the platform palette, which is the
  desktop's colour scheme on KDE. Palettes must set every role: `QPalette(button, window)`
  derives the shading ones. And a bright palette must not come from the active style's
  `standardPalette()`: Breeze's is the KDE colour scheme (seen in ThemeTest on 2026-11-07,
  Bright turned into Golden Honey Oak).
- **Disabled controls look enabled in dark themes.** `QPalette::setColor` without a group
  sets the disabled colour too, the same as the active one. The Launcher's
  `ITheme::fadeInactive` then mixes roles 50% towards one fade colour, which for Dark is the
  window colour: a disabled field's fill ends up nearly the same as an enabled one (35,38,41
  vs 42,46,50), and Fusion draws indicator outlines from Window, which doesn't change. So a
  disabled unchecked radio button looks enabled. ThemeTest's rule (`deriveDisabled`), the
  same for every palette, after the accent:
  - foregrounds and accent fills (text, links, selected text, Highlight) fade to their
    background, not to grey: they take its hue and saturation, and only their lightness sits
    partway to the background's (halfway for text, 60% for Highlight). A red theme fades to
    red, a dark one to its window's grey. A plain mix makes in-between hues (a blue accent on
    the red theme turned purple, with blue left in disabled controls); fading to grey
    ignores the theme;
  - Base, AlternateBase and Button 60–70% to Window: the theme's own colours, flat.
  With a custom accent on a desktop's palette (Native), a disabled accent is the desktop's
  disabled window hue at the lightness of the desktop's own disabled colour for that role.
  On a strongly saturated theme (Red) the disabled colours come out vivid; the saturation
  taken from the background may need tuning. Checked with ThemeTest's Enabled/Disabled
  gallery.
  What the palette can't fix: QLabel draws rich-text links in the active Link colour when
  disabled, and Fusion's disabled slider groove keeps a light accent fill.
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
| Following dark/light | no API: our registry reader | `AppleInterfaceStyle` and a distributed notification (proven) | the desktop portal's `color-scheme` over QtDBus (proven) |
| Accent | registry (proven) | `NSColor.controlAccentColor`, 10.14+ (proven) | the portal's `accent-color` (proven) |
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

## In the Launcher

Ported from ThemeTest, in this order: the #5686 fix, title bars, then everything else in
one change.

- **Settings** (new; the old `ApplicationTheme` isn't read any more: its System meant the
  platform's style):
  - `Theme`: `system`, `bright`, `dark` or `custom`. Empty until chosen, and the setup
    wizard asks until then, existing installs too. Meanwhile System where the system can be
    followed, else Bright.
  - `ThemeStyle`: `fusion` or `native`. Empty is the platform's default: Native on Breeze
    and Kvantum, Fusion elsewhere.
  - `AccentSource`: `theme`, `system` or `custom`. Empty is the system's where there is one.
  - `AccentColor`: the custom accent, `#0078d4` until picked.
- **Code** (`launcher/ui/themes/`): `ThemeManager` (ThemeTest's `Themes`) owns the themes,
  reads and writes the settings, applies them, and re-applies when the system changes.
  `SystemAppearance` is ThemeTest's `SystemTheme` (that name was taken by the theme class).
  `ITheme` is a palette (every role set), an optional accent, stylesheet and search paths,
  and a title bar style; `ThemeManager` adds the accent and the disabled colours.
- **The selectors** (`ThemeSelectionWidget`, `IconThemeSelectionWidget`, cards from
  `SelectionCard`), on a new Appearance settings page and as a setup wizard step, where
  choices apply at once (in the settings too: the dialog has only Close). Themes: a card per
  theme, a small window painted in its colours; the accent (the theme's own, the system's,
  custom); Fusion or Native where Native is offered. What the style
  can't take is disabled with a tooltip saying why. Icons: a card per set with six of its
  icons, Custom only where `iconthemes/custom` exists.
- **The custom accent** is picked in a dialog of our own (`AccentColorDialog`,
  [#5798](https://github.com/MultiMC/Launcher/issues/5798)): suggested colours with the
  system's accent first, a saturation and brightness field with a hue strip, hex, and a
  preview of the chosen theme with the colour. Not `QColorDialog`: on macOS its native panel
  opens in a corner of the screen, picking from the screen goes through permission prompts,
  and our own needs no more of Qt's translations.
- **Custom themes:** `theme.json`'s colours over Dark's, every role derived from them.
  `widgets` (the widget style), `fadeColor` and `fadeAmount` are ignored: the style comes
  from the settings, disabled colours from the rule. Offered only when customized: older
  versions wrote the Dark theme of the time and its tooltip stylesheet there on every start,
  and files like those don't count. Nothing is written there any more. An editor for it
  would make it worth offering to everyone (later). The Dark theme's tooltip stylesheet is
  now its palette.
- **Breeze with our palettes:** Breeze draws tool bars in the KDE colour scheme's Header
  colours, read from the scheme file, not the palette: Bright on a dark KDE had white tool bar
  text. Like KDE applications with their own colour scheme (KColorSchemeManager), the palette
  is written as a scheme (`cache/breeze.colors`) and the application's
  `KDE_COLOR_SCHEME_PATH` property points to it; cleared for System and Fusion. Our palettes
  also go on the widget classes the platform theme gives palettes of their own.
- **Linux:** links QtDBus (the lin32 and lin64 gate expectations list `libQt5DBus.so.5`).

Parts marked in the code's comments go away with a newer Qt.

## Custom themes: what others did

Custom themes are only offered when customized now, and were limited before (one folder,
`themes/custom`). Before designing them again, what a fork did. Prism Launcher (GPL-3) was
looked at through its user documentation and its theme repository only, not its code: formats
and behaviour (2026-11-07).

- **The format is ours, extended.** Community themes are `themes/<name>/theme.json` with our
  keys (`name`, `widgets`, always "Fusion", `colors` with the 13 roles and `fadeAmount`,
  `fadeColor`), plus:
  - `logColors`: colours per log level (Launcher, Message, Debug, Warning, Error, Fatal,
    FatalHighlight);
  - `qssFilePath`: the stylesheet's name, `themeStyle.css` by default.

  Next to them, `themeStyle.css` (mostly a few lines; some themes restyle everything in
  18 KB) and `resources/`.
- **Any number of themes** (every folder in `themes/`), and icon themes the same way
  (`iconthemes/<name>/index.theme`). Colour theme, icons and the cat are three separate
  choices. The cat is a "cat pack": one image, or a folder with `catpack.json`, a default
  image and variants shown between given dates.
- **Sharing by hand:** a community repository (about 40 themes and 5 icon sets, zips on
  GitHub releases, licensed per file). Users unzip into the folder and restart. No import, no
  editor, no browsing in the launcher. The repository calls itself for Prism "and other
  compatible launchers": these themes nearly work in MultiMC today.

What it means for ours:
- Keep reading the same `theme.json` (accept `qssFilePath`; `widgets` and the fade keys are
  ignored already), and list every folder in `themes/`: the existing community themes work.
- Accept `logColors`, through `Colors::readable` against the log's background.
- Do better where they stop: import a zip from the launcher (button, drag and drop), no
  restart, previews on the cards, and an editor ("Customize…" from any theme), as the old
  idea list (`Ideas2015.md`, Ideas → Themes) had it: built-in and user themes, cloning.
- Which themes are offered: every folder in `themes/`, except an old untouched default
  (`CustomTheme`'s check).

## Findings to fix

Found while testing ThemeTest, recorded first, fixed later.

- **macOS, Fusion, System theme:** a black bar between the title bar and the window content
  (2026-11-07). Gone with any other theme. Guess, not verified: System leaves the title bar
  to macOS (`Style::Brightness`, only the window's appearance set), and macOS 11+ draws a
  separator line under a normal title bar; the other themes use a transparent title bar,
  which has none (`NSWindow.titlebarSeparatorStyle`, 11+, could turn it off). May be
  expected: plain macOS windows without a toolbar get that hairline, other Qt apps too;
  Apple's own apps mostly blend the title bar into the content instead. Left as it is
  (decided 2026-11-07): revisit if Mac users complain.

## Ideas


- Icons that follow the palette's brightness. Only Simple has a pair (dark and light icons);
  a dark theme can still get dark icons (#5665). Colored is the default and works on both.

- A contrast setting (a slider) for log colours: `Colors::readable` already takes the target.
- A title bar colour in custom themes' `theme.json`, for themes whose stylesheet paints window
  backgrounds itself.

## Open questions

- Which Linux styles go on the whitelist besides Breeze.
