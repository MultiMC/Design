# Instance icons overhaul

Notes for reworking instance icons: where they come from, how they're stored, named, loaded,
shown and chosen. Started 2026-11-17 from the open icon issues, a few new ideas and the old idea
list. Nothing here is decided yet unless it says so; it's a place to see how the pieces fit
together, and to find a design that fixes several of them at once.

Not in scope: mod icons in the mods list (#3982), MultiMC's own application and window icon
(#4561, #2980), toolbar and theme icons (#2747; the icon themes in `iconthemes/`), and Qt not
following GNOME's scale factor (#4305). They share the word, not the system.

## Goals

- Every instance icon looks right at every size the launcher shows it, whatever its source
  image (size, shape, margins, pixel art or picture).
- Icons from modpack platforms and imported instances can never replace or rename the user's
  own icons, the built-in ones, or each other's.
- Memory and time don't grow with the size of the images: a folder of photos or a 4000 pixel
  pack logo costs about as much as a 64 pixel icon.
- Icon sets can be added and shared as packs, like themes, icon themes and languages.
- The icon picker shows where each icon comes from, in groups, like the instance view.
- Aim to keep backwards compatibility. New formats and keys are fine, as long as
  they're either additive or provide a clear upgrade path.

## How it works now (Launcher develop, 2026-11-17)

### One flat list of keys

- An instance stores one string, `iconKey` in `instance.cfg` (`default` when unset).
- `IconList` (`launcher/icons/IconList.{h,cpp}`) maps keys to `MMCIcon`s
  (`launcher/icons/MMCIcon.{h,cpp}`). A key is a file's base name, in one namespace for
  everything.
- **Built-in icons:** the base names of the files in `:/icons/multimc/{32x32,50x50,128x128,scalable}/instances/`
  in the resources (brick, creeper, steve...), loaded through the icon theme loader (XdgIcon);
  `logo` and `default` are the MultiMC logo (`:/logo.svg`).
- **User icons:** every file in the icons folder (setting `IconsDir`, `icons/` in the data
  folder by default), watched for changes. Key: the file's base name.
- **Overrides:** an `MMCIcon` holds an image per type (`Builtin`, `FileBased`); a user file
  with a built-in icon's name takes its place (`MMCIcon::replace`), and deleting the file brings
  the built-in one back.

### Where icons come from

| Source | Key | Where the image goes |
|---|---|---|
| Built in | the file's name (`steve`) | in the resources |
| The icon picker's Add | the file's name | copied into the icons folder as it is (`installIcons`) |
| Modrinth | `modrinth-<project id>` | icons folder (`NewInstanceDialog::importIconNow`, `installIcon`) |
| Technic | `technic_<logo file name>` | icons folder |
| FTB App import | `ftb_<pack id>` | icons folder |
| Legacy FTB | the pack's logo file name, no prefix | icons folder |
| ATLauncher | the pack's safe name, no prefix | icons folder |
| Imported instance (zip) | the `iconKey` in its `instance.cfg` | the icon file found next to it (`IconUtils::findBestIconIn`) goes into the icons folder; **an existing icon with that key is deleted first** (`InstanceImportTask`) |

- **Export:** a non-built-in icon's file is copied into the instance folder
  (`ExportInstanceDialog`'s `SaveIcon`), so the zip carries `iconKey` and a file of that name.
  Other launchers read this too.
- **The game:** `icon.png` in the game folder, written at launch at up to 128 pixels, shape kept
  (`IconList::saveIcon`). macOS's Dock (`-Xdock:icon`) and the legacy applet frame use it.

### Loading and showing

- Each icon file is a `QIcon(path)`: Qt reads the header for its size, then the whole image the
  first time it's drawn, and keeps it decoded at full size for as long as the icon exists. The
  picker draws every icon, so opening it decodes them all.
- In 2019 (8369dbdc, #2584 and #1663) added icons were shrunk to 512 pixels when installed.
  That's gone: `installIcon` copies files as they are, and `MAX_SIZE` in `IconList.cpp` is
  defined and unused.
- **Shown:** `IconUtils::fitted` (2026-11-17): transparent margins trimmed, fit to the box with
  its shape kept, small pixel art enlarged sharp, cached per size (Theming.md, "Instance
  icons"). The instance view, the side panel (up to 160×80) and nothing else yet.
- **The picker** (`IconPickerDialog`): a `QListView` in icon mode over the whole `IconList`, the
  instance view's delegate, Add, Remove, Open folder. One flat list.

## The problems

### Names collide

- **#3068:** an imported instance with an icon named like a built-in one (`steve`) replaces
  the built-in icon for every instance.
- **#4058** (closed as a duplicate of #3068): two packs with the same `iconKey` and different
  icons: the newer one replaces the older's for good.
- Importing deletes any existing icon with the pack's key before installing its own: a user
  icon called `skyblock` is gone after importing a pack whose icon is `skyblock`.
- ATLauncher and Legacy FTB icons have no prefix at all; the others use three different ones
  (`modrinth-`, `technic_`, `ftb_`).

### Memory and size

- **#2317:** pointing the icons folder at a Pictures folder took MultiMC from 11 MB to 4.6 GB,
  and froze it while loading.
- **#2811:** a 2362×2362 pack icon crashed the edit dialogs.
- **#2584, #1663** (closed in 2019): very large icons crashed; the fix (shrink on install) is
  gone.

### Looks

- Different margins and shapes made icons look different sizes; pixel art was blurry when
  enlarged; the side panel didn't enlarge small icons. Fixed in the view and the panel
  (2026-11-17, `IconUtils::fitted`).
- **#3820:** ATLauncher icons are crushed in the macOS Dock while the game runs: `icon.png` is
  written with the icon's shape (say 128×64), and the Dock squashes it into a square.
- How to choose between smooth and sharp (nearest pixel) scaling (the user's TODO, 2026-11): by
  the source's size, since nothing in the content tells pixel art from pictures (Theming.md).

### Storage

- **#2845:** with the icons folder moved elsewhere (`IconsDir`), the picker shows the new
  folder's icons but Add puts new ones in the old one.
- **#3702:** icons downloaded from platforms get no file extension.

### Platforms

- **#4029:** installing an FTB pack doesn't set its icon.
- **#3714:** Legacy FTB pack icons sometimes don't appear.

### Wanted

- **#4368** (and #2039, closed when it moved to the idea list): choose the icon new instances
  get. The idea list adds: a button in the picker, and a check mark over the current default.
- **#2101** (closed, in the idea list): rename custom icons in the picker. The idea list asks:
  what about instances using the old name?
- **#5117:** add an instance icon from a URL.
- **#4960:** desktop shortcuts with the instance's icon.
- **#1031** (closed, done): exported packs carry their custom icon (see Export above).

## Ideas

- **Icon packs** (the user, 2026-11-17): sets of instance icons as packages, installed,
  shared and removed like themes, icon themes and languages: a folder each, imported and
  exported as zips (`ThemeArchive`'s way), with who made them and their license
  (`theme.json.license`, REUSE, as the community themes have). Later, from a store on hopper,
  like the theme store.
- **Namespaces** (the user, 2026-11-17): each platform's icons in a namespace of their own, so
  they can't touch the user's icons or each other's.
- **Groups in the picker** (the user, 2026-11-17), like the instance view's: Built in, one per
  platform, one per installed icon pack (and Yours).
- **Folders as categories** (the 2015 idea list, Ideas → Application core → Icons): folders in
  the icons folder (`icons/Blocks`, `icons/Items`) show as categories: Default, Custom,
  Custom/Blocks, Custom/Items.
- **More built-in icons** (the 2015 idea list): instance block icons; replace the old textures
  with better ones, not Mojang's assets.
- **Default icon** with a check mark in the picker, **rename** in the picker (the 2015 idea
  list, #2039, #2101).

## A design to start from

A first sketch, to be argued with. Each part says which problems it takes care of.

### 1. Sources, and keys that say where an icon comes from

An icon has a source, and its key is the source and a name within it:

| Source | Key | Folder |
|---|---|---|
| Built in | `steve` (as now) | resources |
| Yours | `skyblock` (as now) | `icons/` |
| An icon pack | `pack:<pack id>/<name>` | `iconpacks/<pack id>/` |
| A platform | `<platform>:<pack id>` (`modrinth:AANobbMI`, `technic:tekkit`, `ftb:23`, `atlauncher:SkyFactory4`) | `icons/.platforms/<platform>/` (or the meta cache?) |
| An imported instance | `instance:<instance id>` | the instance's own folder (its icon file, as exported) |

- Keys without a `:` are what they are today: yours if `icons/` has the file, else built in.
  Nothing existing changes meaning.
- Platform and import keys can't collide with yours or each other's: **#3068, #4058**, and the
  delete-before-import go away.
- **Open:** keep the built-in names unprefixed and let your icons still override them (as now),
  or stop overriding (a separate `builtin:` key)? Overriding is a feature some people use.
- **Open:** migrate the old platform keys (`modrinth-…`, `technic_…`, `ftb_…`) into namespaces,
  or leave them as user icons? ATLauncher and Legacy FTB ones can't be told from yours.
- **Open:** an imported instance's icon living in its own folder means it travels with the
  instance (copy, export) by itself. Additively: the new key goes where only new versions look,
  and export keeps writing a plain `iconKey` and a file of that name as well; or `iconKey` takes
  the new form outright, and older versions and other launchers show the default icon for it.

### 2. Icon packs

- `iconpacks/<id>/` in the data folder: the images, and a small `pack.json` (name, description)
  with its `.license` (REUSE), like themes. Folders inside are categories (the 2015 idea).
- Built-in icons become the first pack ("MultiMC"), in the resources.
- Import and export as zips, drag and drop, through `ThemeArchive`; later from the store.
- **Open:** what happens to instances using a pack's icon when the pack is removed: the default
  icon, or keep a copy?

### 3. Loading: small, lazy, capped

- Our own `QIconEngine` for icon files: reads only the header when listed; decodes on first use,
  straight to at most 512 pixels on the long side (`QImageReader::setScaledSize`), and keeps only
  that. About 1 MB per icon at most, whatever the file; nothing decoded until shown: **#2317,
  #2811**.
- The picker shows what's in view only (it already does; with lazy decoding it stays cheap).
- **Open:** also skip files over a size in bytes (a 200 MB TIFF), or images over some pixel
  count, as broken rather than icons?

### 4. One way to fit icons

`IconUtils::fitted` everywhere an instance icon is shown or written: the view and the panel
(done), the picker, the dialogs' icon buttons, the console window, desktop shortcuts (#4960),
and the game's `icon.png`, written centred in a 128×128 square: **#3820**.

### 5. The picker

- Groups: Default, Yours (and your folders), each icon pack, each platform. Collapsible, like
  the instance view's groups.
- Set as default (check mark): **#4368, #2039**. Rename yours: **#2101** (instances using it
  follow the new name). Add from a file or a URL: **#5117**.
- Add copies into the icons folder that's actually set: **#2845**. Files get their real
  extension from their content: **#3702**.

### 6. Platform icons in one place

One function for "this platform's icon for this pack": downloads into the platform's namespace,
returns the key. The platform pages and install tasks call it, so a platform can't forget its
icon (**#4029**) or name it its own way. **#3714** needs its own look.

## To review field by field before building

Per the working rules, these get reviewed with the user one field at a time before code:

- the key format (separator, allowed characters, what's unprefixed);
- the folder layout (`icons/`, `icons/.platforms/`, `iconpacks/`) and what lives in the data
  folder versus the cache;
- `pack.json`;
- what an exported instance carries, and what older versions and other launchers make of it.

## Order

Small, independent steps first, each useful alone:

1. Lazy, capped loading (3): fixes the memory issues without changing any key.
2. Fitting everywhere, the game's `icon.png` square (4): #3820.
3. Picker fixes that don't need namespaces: the right folder (#2845), extensions (#3702),
   default icon (#4368), rename (#2101).
4. Namespaces for platforms and imports (1, 6), with migration decided.
5. Icon packs and picker groups (2, 5).
