# autorandr

Automatically select a display configuration based on connected devices

## Branch information

This is a compatible Python rewrite of
[wertarbyte/autorandr](https://github.com/wertarbyte/autorandr). Contributions
for bash-completion, fd.o/XDG autostart, Nitrogen, pm-utils, and systemd can be
found under [contrib](contrib/).

The original [wertarbyte/autorandr](https://github.com/wertarbyte/autorandr)
tree is unmaintained, with lots of open pull requests and issues. I forked it
and merged what I thought were the most important changes. If you are searching
for that version, see the [`legacy` branch](https://github.com/phillipberndt/autorandr/tree/legacy).
Note that the Python version is better suited for non-standard configurations,
like if you use `--transform` or `--reflect`. If you use `auto-disper`, you
have to use the bash version, as there is no disper support in the Python
version (yet). Both versions use a compatible configuration file format, so
you can, to some extent, switch between them.  I will maintain the `legacy`
branch until @wertarbyte finds the time to maintain his branch again.

If you are interested in why there are two versions around, see
[#7](https://github.com/phillipberndt/autorandr/issues/7),
[#8](https://github.com/phillipberndt/autorandr/issues/8) and
especially
[#12](https://github.com/phillipberndt/autorandr/issues/12)
if you are unhappy with this version and would like to contribute to the bash
version.

## License information and authors

autorandr is available under the terms of the GNU General Public License
(version 3).

Contributors to this version of autorandr are:

* Adrián López
* andersonjacob
* Alexander Wirt
* Chris Dunder
* Christoph Gysin
* Daniel Hahler
* Maciej Sitarz
* Mathias Svensson
* Matthew R Johnson
* Nazar Mokrynskyi
* Phillip Berndt
* Rasmus Wriedt Larsen
* Simon Wydooghe
* Stefan Tomanek
* stormc
* tachylatus
* Timo Bingmann
* Timo Kaufmann
* Tomasz Bogdal
* Victor Häggqvist

## Installation/removal

You can use the `autorandr.py` script as a stand-alone binary. If you'd like to
install it as a system-wide application, there is a Makefile included that also
places some configuration files in appropriate directories such that autorandr
is invoked automatically when a monitor is connected or removed, the system
wakes up from suspend, or a user logs into an X11 session. Run `make install`
as root to install it.

If you prefer `pip` over your package manager, you can install autorandr with:

    sudo pip install "git+http://github.com/krzysztofmo/autorandr#egg=autorandr"

## How to use

Save your current display configuration and setup with:

    autorandr --save mobile

Connect an additional display, configure your setup and save it:

    autorandr --save docked

Now autorandr can detect which hardware setup is active:

    $ autorandr
      mobile
      docked (detected)

To automatically reload your setup:

    $ autorandr --change

To manually load a profile:

    $ autorandr --load <profile>

or simply:

    $ autorandr <profile>

autorandr tries to avoid reloading an identical configuration. To force the
(re)configuration:

    $ autorandr --load <profile> --force

To prevent a profile from being loaded, place a script call _block_ in its
directory. The script is evaluated before the screen setup is inspected, and
in case of it returning a value of 0 the profile is skipped. This can be used
to query the status of a docking station you are about to leave.

If no suitable profile can be identified, the current configuration is kept.
To change this behaviour and switch to a fallback configuration, specify
`--default <profile>`. The system-wide installation of autorandr by default
calls autorandr with a parameter `--default default`. There are three special,
virtual configurations called `horizontal`, `vertical` and `common`. They
automatically generate a configuration that incorporates all screens
connected to the computer. You can symlink `default` to one of these
names in your configuration directory to have autorandr use any of them
as the default configuration without you having to change the system-wide
configuration.

You can store default values for any option in an INI-file in
`~/.config/autorandr/settings.ini` in a section `config`. The most useful
candidate for doing that is `skip-options`, if you need it.

If you are trying to configure auto-switching on laptop you might run into
problems, because by default internal display is always fingerprinted
even when the lid is closed. Say you wanted to create two profiles: one with
the lid closed and an external display connected and another one with lid open
and the same external display connected. You will end up with two different
profiles, but autorandrw won't be able to detect the correct profile, because
both of the profiles will match. The solution is to exclude internal display
from being fingerprinted when the lid is closed by passing output's name to
--skip-outputs option.

    $ autorandr --skip-outputs eDP-1-1

Here is an example of simple wrapper script for autorandr, which excludes
internal display from being fingerprinted only when the lid is closed.

```bash
#!/usr/bin/env bash
# This a wrapper script for autorandr

INTERNAL_DISPLAY_NAME="eDP-1-1"

exec "autorandr" "$@" $([[ $(cat /proc/acpi/button/lid/LID0/state | awk '{print $2}') == 'closed' ]] \
&& echo "--skip-outputs" "${INTERNAL_DISPLAY_NAME}")
```

## Hook scripts

Three more scripts can be placed in the configuration directory (as 
(as defined by the [XDG spec](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html),
usually `~/.config/autorandr` or `~/.autorandr` if you have an old installation
for user configuration and `/etc/xdg/autorandr` for system wide configuration):

- `postswitch` is executed *after* a mode switch has taken place. This can be
  used to notify window managers or other applications about the switch.
- `preswitch` is executed *before* a mode switch takes place.
- `postsave` is executed after a profile was stored or altered.
- `predetect` is executed before autorandr attempts to run xrandr. 

These scripts must be executable and can be placed directly in the configuration
directory, where they will always be executed, or in the profile subdirectories,
where they will only be executed on changes regarding that specific profile.

Instead (or in addition) to these scripts, you can also place as many executable
files as you like in subdirectories called `script_name.d` (e.g. `postswitch.d`).

If a script with the same name occurs multiple times, user configuration
takes precedence over system configuration (as specified by the
[XDG spec](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html))
and profile configuration over general configuration.

As a concrete example, suppose you have the files

- `/etc/xdg/autorandr/postswitch`
- `~/.config/autorandr/postswitch`
- `~/.config/autorandr/postswitch.d/notify-herbstluftwm`
- `~/.config/autorandr/docked/postswitch`

and switch from `mobile` to `docked`. Then
`~/.config/autorandr/docked/postswitch` is executed, since the profile specific
configuration takes precedence, and
`~/.config/autorandr/postswitch.d/notify-herbstluftwm` is executed, since
it has a unique name.

If you switch back from `docked` to `mobile`, `~/.config/autorandr/postswitch`
is executed instead of the `mobile` specific `postswitch`.

In these scripts, some of autorandr's state is exposed as environment variables
prefixed with `AUTORANDR_`. The most useful one is `$AUTORANDR_CURRENT_PROFILE`.

If you experience issues with xrandr being executed too early after connecting
a new monitor, then you can use a `predetect` script to delay the execution.
Write e.g. `sleep 1` into that file to make autorandr wait a second before
running `xrandr`.

## Changelog

* *2018-01-04* Fixed vertical/horizontal/clone-largest virtual profiles
* *2018-03-07* Output all non-error messages to stdout instead of stderr

**autorandr 1.4**

* *2017-12-22* Fixed broken virtual profile support
* *2017-12-14* Added support for a settings file
* *2017-12-14* Added a virtual profile `off`, which disables all screens

**autorandr 1.3**

* *2017-11-13* Add a short form for `--load`
* *2017-11-21* Fix environment stealing in `--batch` mode (See #87)

**autorandr 1.2**

* *2017-07-16* Skip `--panning` unless it is required (See #72)
* *2017-10-13* Add `clone-largest` virtual profile

**autorandr 1.1**

* *2017-06-07* Call systemctl with `--no-block` from udev rule (See #61)
* *2017-01-20* New script hook, `predetect`
* *2017-01-18* Accept comments (lines starting with `#`) in config/setup files

**autorandr 1.0**

* *2016-12-07* Tag the current code as version 1.0.0; see github issue #54
* *2016-10-03* Install a desktop file to `/etc/xdg/autostart` by default
