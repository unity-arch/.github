# unity on arch

the unity 7 desktop, packaged for arch linux. sources are the ubuntu unity project
(https://gitlab.com/ubuntu-unity/unity) and the ubuntu archive on launchpad, patched only
where arch differs from ubuntu. no python2, no replacement gtk3.

repo: [unity-arch/packages](https://github.com/unity-arch/packages), pkgbuilds plus a ci
workflow that publishes a pacman repo to the `repo` release.

## status

last updated 2026-09-11. tested on cachyos (arch, kernel 7.2, mesa 26.2, gtk 3.24.52) on
an amd radeon 890m, inside a rootful xwayland window and through a dry run of the real
session path (cinnamon-session as leader, every systemd user unit's command alongside).
a real display manager login has not been done yet.

### working

- compiz 0.9.14.2 (ubuntu's tree) with the unityshell plugin, unity 7.7.1
- launcher, top panel, window decorations, alt tab, spread, keyboard shortcut overlay
- the dash with the home scope, applications lens and files lens
- indicators: application, bluetooth, datetime, keyboard, messages, power, printers,
  session, sound. the ido widgets inside them (volume slider, calendar, switches) render on
  stock gtk3 through a ported menu item factory, see below
- global menu: unity-gtk-module exports gtk2/gtk3 menus, indicator-appmenu shows them,
  hud answers queries
- unity-settings-daemon, unity-greeter (test mode), the lock screen settings
- unity-control-center: appearance, display, time and date (with the world map), text entry,
  details, keyboard, sharing, and the rest of the grid. the theme dropdowns need a theme
  package, and the printers panel has no tile (an upstream gap, `unity-control-center printers`
  still opens it)
- session files: `unity.desktop` in xsessions, `unity-session.target` and the units it
  pulls in, clean logout through cinnamon-session
- the whole chain builds from scratch in a clean `archlinux:base-devel` container

### not working or untested

- a real login through a display manager (plasmalogin/sddm/lightdm). the unit graph
  verifies and the session dry run passes, but nobody has logged in yet
- lock screen and screen unlock through the real pam stack (the pam file is shipped,
  untested)
- indicator menus inside submenus (the messages indicator's per app submenus), the factory
  port only walks section links
- indicator menu items hidden with `hidden-when` may pair up with the wrong widget
- fcitx input method support, dropped everywhere (only fcitx4 api in the code)
- online accounts in the control center (goa), disabled like ubuntu does
- software center and apt integration in the applications lens, disabled, arch has neither
- ubuntu's touch and unity8 leftovers (libunity-api, qtdbusmock, dee-qt) are not built
- multi monitor, hidpi scaling, nvidia, none tested

### what needs improving

- log in for real and fix what falls out, then remove the "untested" line above
- `unity-session` depends on the whole `cinnamon` package for one gsettings schema
  (`org.cinnamon.sounds`, cinnamon-session aborts on logout without it). debian has a
  `cinnamon-common` split, arch doesn't. a smaller fix is welcome
- the indicator menu factory port (ido + libindicator) should walk submenus and anchor
  replacements on something better than position
- ubuntu-unity-settings points at the yaru theme, which lives in the aur. the ambiance
  look is packaged here (`unity-ambiance-settings`), yaru is not yet
- ci builds on a 2 core runner, the chain takes a long time. cache built packages between
  runs or split the workflow per package
- no signing on the pacman repo yet
- version tracking: every pkgbuild pins a commit. a script that diffs against launchpad
  and gitlab heads would make bumps routine

## how it fits together

```
plasmalogin / sddm / lightdm
  └─ /usr/bin/unity-session            imports DISPLAY etc into systemd --user
       └─ unity-session.target
            ├─ unity-session.service   cinnamon-session --session=unity (leader)
            ├─ unity7.service          compiz, loads unityshell from the unity profile
            │    ├─ unity-settings-daemon.service
            │    ├─ unity-panel-service.service
            │    │    └─ indicator-*.service (nine of them)
            │    ├─ unity-gtk-module.service
            │    └─ bamfdaemon.service
            ├─ window-stack-bridge.service (hud)
            └─ xdg-desktop-autostart.target
```

## decisions

- **no patched gtk3.** ubuntu carries `ubuntu_gtk_custom_menu_items.patch` so indicator
  menus can hold custom widgets. we ported the approach the ayatana forks use for debian
  instead: ido exports its own `IdoMenuItemFactory` extension point and libindicator swaps
  `x-canonical-type` items after `gtk_menu_new_from_model`. two patches, everything else
  is stock ubuntu code
- **no patched gsettings-desktop-schemas.** unity reads `ubuntu-lock-on-suspend`, a key
  only ubuntu's schema has. unity is patched to treat the key as optional
- **glewmx** (glew 1.13 multi context build nux needs) installs its headers under
  `/usr/include/glewmx` so it coexists with arch's glew 2.x
- **cinnamon-session** runs the session, same as upstream, because gnome-session dropped
  x11 in 49
- **no python2**, anywhere. everything ubuntu still ships is python3 now
- **ubuntu's accountsservice additions** (`act_user_set_input_sources` and friends) are
  patched out of unity-settings-daemon and indicator-keyboard, the lightdm fallback path
  covers it

## packages

| package | version | notes |
|---|---|---|
| unity | 7.7.1.26.04.20260306 | the shell, two patches (lock key optional, file manager fallback) |
| nux | 4.0.8.18.10.20180623 | ubuntu's quilt series plus an fbo vector resize fix |
| compiz-ubuntu | 1:0.9.14.2.25.10.20250930 | cmake module dir, no live schema compile |
| unity-settings-daemon | 15.04.1.21.10.20220802 | accountsservice input sources dropped |
| unity-session | 49.4 | own entry script, depends on cinnamon for a schema |
| unity-greeter | 25.04.1 | no lightdm conf override, you pick the greeter |
| unity-control-center | 15.04.0.23.04.20230220 | alt tap shortcut reverted (needs ubuntu's gtk), region panel uses glibc locales instead of language-tools |
| unity-scope-home, unity-lens-files, unity-lens-applications | 6.8.2, 7.1.0, 7.1.0 | applications lens ported to zeitgeist 2.0 and xapian 2 |
| ido, libindicator-gtk3 | 13.10.0, 16.10.0 | the menu item factory port |
| indicator-* (nine) | various | whoopsie, fcitx, dbustest and other ubuntu only bits dropped |
| hud, indicator-appmenu, unity-gtk-module, libcolumbus | | global menu stack, hud built without dee-qt |
| libunity, dee, libunity-misc, gsettings-ubuntu-schemas, unity-asset-pool, cmake-extras | | as ubuntu ships them |
| glewmx, geis, grail, frame, xpathselect, libgeonames | | small leaf libraries arch doesn't have (timezonemap comes from arch) |
| ubuntu-unity-settings | 22.10 | gschema override for the yaru look |
| light-themes, ubuntu-mono, humanity-icon-theme, ttf-ubuntu-font-family, ubuntu-wallpapers | 24.04, 0.6.16, 0.869, 26.04.2 | ambiance and radiance with everything they reference |
| unity-ambiance-settings | 1 | gschema override that switches unity and the greeter to ambiance, installs after the yaru one |
| unity-desktop | | meta package pulling in everything above |

## install

```
[unity-arch]
SigLevel = Optional TrustAll
Server = https://github.com/unity-arch/packages/releases/download/repo
```

then `pacman -Sy unity-desktop`, log out, pick "Unity" in your display manager.

the default look is yaru (from the aur, optional). for the classic ubuntu look,
`pacman -S unity-ambiance-settings`, it pulls in ambiance, radiance, ubuntu-mono,
humanity, the ubuntu font and the warty wallpaper, and switches the defaults to them.
`gtk-engine-murrine` from the aur makes gtk2 apps match.

## contributing

pkgbuilds live in `pkgs/<name>/`, patches next to them. `./build.sh -i` builds the chain
in order and installs as it goes. keep patches minimal and named for what they do, and
prefer a configure flag over a patch wherever one exists.

## changing the launcher button (bfb)

the big button at the top of the launcher ships with the arch logo on ubuntu's tile
artwork. unity resolves it as a themed file called `launcher_bfb` (svg first, then png)
and searches, in order:

1. `$XDG_DATA_HOME/themes/<gtk theme>/unity/launcher_bfb.svg` (usually `~/.local/share/themes/...`)
2. `~/.themes/<gtk theme>/unity/launcher_bfb.svg`
3. `/usr/share/themes/<gtk theme>/unity/launcher_bfb.svg`
4. `/usr/share/unity/launcher_bfb.svg`, the package default

`<gtk theme>` is whatever `org.gnome.desktop.interface gtk-theme` is set to (Yaru-dark,
Ambiance, ...). so to use your own, drop a 128x128 svg (or png) at

```
mkdir -p ~/.local/share/themes/$(gsettings get org.gnome.desktop.interface gtk-theme | tr -d "'")/unity
cp my-logo.svg ~/.local/share/themes/<theme>/unity/launcher_bfb.svg
```

and restart unity (`unity --replace`, or log out). a theme package can do the same under
`/usr/share/themes/<Theme>/unity/`. the same lookup is used for the hud icon, so the hud
launcher entry follows your change.

to get ubuntu's logo back, copy `resources/launcher_bfb.svg` from the unity source tree
to one of those locations. the arch logo is a trademark of arch linux, used here under the
arch linux trademark policy.
