# desktopctl

*"A desktop environment for window managers"*

Using a window manager (i3/sway/awesome/etc) should not mean that one has to
completely reinvent the wheel and manage a strange collection of one-liners to
get wifi status or to see if headphones are connected. This project aims to be
a solution to this problem.

Some of these tools are simple one-liners, some are more complex. The idea is to
create a suite of platform-independent (Distro/Desktop Environment independent)
tools to get and set system information.

Similar in nature to [neofetch](https://github.com/dylanaraps/neofetch), but
also allows for *setting* information too (not just *getting*).

### Project status

The code is not particularly robust. It can and will fail in non-obvious ways if
certain utility programs are not installed.

Notably, this program assumes certain permissions are granted to the user. For
example, to change the brightness in Archlinux, the user must be part of the
`video` group; or to non-interactively change the time zone with `timedatectl`
one needs a polkit rule.

### Features

* **Session Authentication** Show session authentication status
* **Backlight Manager** Get and set backlight
* **Battery Manager** Get battery status
* **Bluetooth Manager** Get and set Bluetooth
* **Colour picker** Colour picker
* **CPU frequency** Get CPU frequency
* **Headphone indicator** Get whether headpones are plugged in
* **IP Address** Get local IP address
* **Media Watch Time** Print total media time of listed files
* **Memory** Get memory usage
* **Monitor Directories/Files** Monitor directories/files
* **Check if online** Get current internet connection status
* **Screenshots** Take screenshots
* **Sound Manager** Get and set sound
* **Internet speedtest** Show current internet speed
* **SSH** List ssh clients on local network
* **Time zone** Get and set current time zone
* **Wifi Status** Set wifi on/off

Limited features

* **Configuration file differences** Show configuration file differences (Archlinux only)
* **Power Manager** Set power and charging profiles (Dell only)
* **Reboot check** List programs that need to be restarted (Archlinux only)
* **Xwayland clients** List number of xwayland clients (Sway only)

---

### Configuring

The configuration is read from `~/.config/desktopctl/desktopctlrc`.

---

#### Session Authentication

Show session 

Prints whether the session is [correctly
authenticated](https://wiki.archlinux.org/title/General_troubleshooting#Session_permissions)
using `loginctl`.

Arguments: None

#### Backlight Manager

Get and set backlight

Is configured to give non-linear brightness changes with a very simple
interface. Can easily be bound to keyboard brightness keys.

```
Arguments:
  NUMBER,        Set backlight to NUMBER percentage.
  +NUMBER,       Increase backlight percentage by NUMBER.
  -NUMBER,       Decrease backlight percentage by NUMBER.

  +,             Go to the next backlight intensity level.
  -,             Go to the previous backlight intensity level.

  -p, --print,   Print current backlight percentages.
  -n, --notify,  Send desktop notification with current backlight intensity.
  -h, --help,    Display this help.
```

Changes are made with `xbacklight` under the hood.

The brightness profile can be configured by setting `$BACKLIGHT_LEVELS` in the
configuration file.

The default is

```
BACKLIGHT_LEVELS=(0 1 2 3 4 5 6 7 8 9 10 15 20 25 30 40 50 60 70 80 90 100)
```

(Small steps at lower brightnesses and larger steps at higher brightnesses)

#### Battery Manager

Get battery status

Gives very detailed statistics about the battery usage rate. Useful for building
a status line. Comes shipped with a default status line.

```
Arguments:
  power,       Current power usage (in W)
  voltage,     Current battery voltage (in uV)
  current,     Current battery current (in uA)
  capacity,    Current battery capacity (in %)
  charge,      Current battery capacity (in C)
  charging,    Exit value 0 if charging
  charged,     Exit value 0 if fully charged

  remaining,   If on battery: remaining time until battery depleted
               If charging: remaining time until battery full
               (Human readable string)

  info,        Statusline like information about battery
               (May need correct font configuration)
```

Configure battery icons with `$BATTERY_LEVELS_CHARGING` and `$BATTERY_LEVELS`.

The defaults are set using [Overpass Mono Nerd
Fonts](https://github.com/ryanoasis/nerd-fonts).

```
BATTERY_LEVELS_CHARGING=(󰂆 󰂆 󰂇 󰂈 󰂉 󰂊 󰂊 󰂋 󰂋 󰂅 )
BATTERY_LEVELS=(󰁺 󰁻 󰁼 󰁽 󰁾 󰁿 󰂀 󰂁 󰂂 󰁹 )
```

(You may not be able to see these in browser)

#### Bluetooth Manager

Get and set Bluetooth

```
Arguments:
  on,                           Power bluetooth on
  off,                          Power bluetooth off
  --daemon, daemon,             Interact with the bluetooth daemon
  -i, --indicator,              Print current bluetooth connection indicator
  -l, --list, list,             List connected bluetooth devicees
  -r, --restart, restart,       Restart bluetooth
  -c, --connect, connect,       Connect to bluetooth device
  -d, --disconnect, disconnect, Connect to bluetooth device
  -h, --help, help,             Print this help
```

To display battery status of bluetooth devices, experimental features must be
enabled in `/etc/bluetooth/main.conf` in the `[General]` section by setting

```
Experimental = true
```

Allows configuring bluetooth from the command line whilst aliasing devices with
nice names. For example

```
desktopctl bluetooth connect sony-headphones
```

Instead of

```
bluetooth on
bluetoothctl agent on
bluetoothctl default-agent
# Find out the MAC address of your Bluetooth device
# Suppose it is 00:23:02:C0:A9:6F
bluetoothctl connect 00:23:02:C0:A9:6F
```

These aliases are defined in the configuration file with a function which must
be called `desktopctl.bt.config` and should return from an alias the correct MAC
address.

Here is an example

```
# Inside ~/.config/desktopctl/desktopctlrc
desktopctl.bt.config () {
  case "$1" in
      aukey)         ID=00:23:02:C0:A9:6F ;;
      soundcore)     ID=08:EB:ED:5F:37:28 ;;
      dell-keyboard) ID=ED:DD:9A:5F:57:DB ;;
      dell-mouse)    ID=EE:79:C0:96:C5:C3 ;;
      hr)            ID=DA:80:4B:56:CE:4E ;;
      sony)          ID=38:18:4C:BE:BB:00 ;;
      jlab)          ID=00:23:10:0C:74:47 ;;
      wacom)         ID=DC:2C:26:09:C7:70 ;;
      shokz)         ID=C0:86:B3:57:39:67 ;;
      *)             ID=$1 ;;
  esac
}
````

Bluetooth is also quite fragile. Often one needs to restart to make things work.
This script handles this automatically.

May require polkit rule to allow restarting bluetooth without password

This rule will allow any user in the `wheel` group to restart the systemd
`bluetooth.service`.

```
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        subject.isInGroup("wheel") &&
        action.lookup("unit") == "bluetooth.service") {
            return polkit.Result.YES;
      };
});
```

#### Colour picker

Simple colour picker. Copies value to clipboard in RBG, Hex and SRGB values.

Compatible with wayland and X11.

#### CPU frequency

Prints `<avg CPU clock speed>/<max CPU clock speed>` in human readable form

Data as reported by: `/sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq`

```
Arguments:
  -a, --avg, avg,    Only print average CPU clock speed over all cores
  -m, --max, max,    Only print maximum CPU clock speed over all cores
  -r, --raw, raw,    Do not make human readable

  -h, --help, help,  Display this help
```

Useful for status lines.

#### Headphone indicator

Prints indicator if headphones are plugged in

Uses `pactl` under the hood

Configure the icon in the configuration file by setting `HEADPHONE_INDICATOR`.

Useful for status lines.

#### IP Address

Get local IP address.

#### Media Watch Time

Prints watch times of media with remaining time and percentage.

Requires `column` from `util-linux`.

```
Arguments:
  A list of media files (mp4)
```

Example:

```
$ cd tv-shows/planet-earth
$ desktopctl mediawatchtime *mp4
Item  |   Title                                      |   Duration  |   Watched  |   % Watched  |   Remaining
1     |  S01E01-planet-earth-from-pole-to-pole.mp4:  |  0:59       |  0:59      |  9.00        |  9:44
2     |  S01E02-planet-earth-mountains.mp4:          |  0:57       |  1:56      |  18.00       |  8:47
3     |  S01E03-planet-earth-fresh-water.mp4:        |  0:58       |  2:55      |  27.00       |  7:48
4     |  S01E04-planet-earth-caves.mp4:              |  0:57       |  3:52      |  36.00       |  6:51
5     |  S01E05-planet-earth-deserts.mp4:            |  0:57       |  4:50      |  45.00       |  5:53
6     |  S01E06-planet-earth-ice-worlds.mp4:         |  0:58       |  5:49      |  54.00       |  4:54
7     |  S01E07-planet-earth-great-plains.mp4:       |  0:58       |  6:48      |  63.00       |  3:55
8     |  S01E08-planet-earth-jungles.mp4:            |  0:58       |  7:46      |  72.00       |  2:56
9     |  S01E09-planet-earth-shallow-seas.mp4:       |  0:58       |  8:45      |  81.00       |  1:57
10    |  S01E10-planet-earth-seasonal-forests.mp4:   |  0:58       |  9:44      |  90.00       |  0:58
11    |  S01E11-planet-earth-ocean-deep.mp4:         |  0:58       |  10:43     |  100.00      |  0:00
```

#### Memory

Print memory usage (human readable).

```
Options
  -r, --raw, raw,  Display in raw bytes (not human readable).
```

Useful for status lines

#### Monitor Directories/Files

Prints size, number of files in directory, and changes since last polling.

Argument: Directory or file to monitor

```
Arguments:
  dir,             Directory or file to monitor
  -i, --interval,  Time interval to re-compute file sizes and changes in seconds
                   (Default 60)
```

E.g.

```
$ rsync /path/to/src/dir /path/to/dest/dir
```

Forgot to add progress options! ... instead of cancelling the transfer, just
monitor the destination directory directly

```
$ desktopctl mondir -i 5 /path/to/dest/dir
mondir: Monitoring '/path/to/dest/dir' with interval 5s.

2024-03-30-15:23:14     Total: 188G     Files: 349222
2024-03-30-15:23:20     Total: 188G     Files: 349266   Diff: 500MiB Files diff: 44    Speed: 100MiB/s
2024-03-30-15:23:26     Total: 189G     Files: 349339   Diff: 525MiB Files diff: 73    Speed: 105MiB/s
...
```

Of course this is also useful for tools that do not offer progress statistics.

#### Check if online

Get current internet connection status

```
Arguments:
  -t, --timeout,   Set timeout in seconds
  -q, --quiet,     Do not print. Only uses exit value
``` 

Note: Options must be specified separately i.e. `-q -t 30` and not `-qt30` or
`-qt 30`.

Useful in scripts, to start backups or synchronise emails only when connected to
the internet.

#### Screenshots

Take screenshots

Supports taking a partial selection of the screen

Supports wayland/x11

Screenshots are copied to the clipboard

```
Arguments:
  --select,  Will ask user to make selection of area to screenshot
  --blurred, (x11 only) Will blur screenshot
```

Blurred screenshots are nice for creating a lockscreen with current screen
contents blurred.

#### Sound Manager

Get and set sound

```
Arguments:
  +,               Increase volume
  -,               Decrease volume
  sink,            Print current default output sink
  ismuted,         Print "yes" if muted all devices are muted, else no
  notify,          Send a desktop notification of current volume
  get-volume, get  Print current volume
  mute,            Mute current default sink
  toggle,          Toggle mute status
```

Uses `pactl` under the hood

Useful for displaying current volume in status line or when binding keyboard
buttons.

#### Internet speedtest

Show current internet speed

`curl`s a file from Linode

```
http://eu-central-1.linodeobjects.com/speedtest/100MB-speedtest
```

#### SSH

List ssh clients on local network

Useful for finding a headless Raspberry Pi after setting it up.

#### Time zone

Get and set current time zone.

Uses [ip-api.com](http://ip-api.com/) to get the time zone based on IP address.

Uses systemd's `timedatectl` to set the time zone. To do this non-interactively
one needs the following polkit rule

```
# /etc/polkit-1/rules.d/timedate.rules
polkit.addRule(function(action, subject) {
  if (action.id == "org.freedesktop.timedate1.set-time") {
    return polkit.Result.YES;
  }
});
```

#### Wifi Status

Set wifi on/off. Show wifi connection status, signal strength. Useful for status
lines.

```
Arguments:
  ssid,       Print connected ssid
  icons,      Print ssid with icons
  only-icons  Only print icons
  off,        Turn wifi off
  on,         Turn wifi on
```

Configuration values

```
WIFI_ICON_CONFIGURING=󰤫
WIFI_ICON_LOW_STRENGTH=󰤟
WIFI_ICON_MED_STRENGTH=󰤢
WIFI_ICON_HIG_STRENGTH=󰤨
WIFI_ICON_DISCONNECTED=󰤯
WIFI_ICON_OFF=󰤮
# nm for Network Manager and iwd for IWD
WIFI_BACKEND=nm
WIFI_DEVICE=wlan0
```

---

### Limited  Features

#### Config Diff

Uses configured `DIFFTOOL` (from configuration file) to display changes of files
that have been modified on disk from how they were originally shipped with the
package manager.

```
Arguments:
  path, Path to configuration file
```

Example: See changes from the default in the bluetooth configuration file, call

```
desktopctl config_diff /etc/bluetooth/main.conf
```

Only works on Archlinux.

#### Power Manager

```
Charging Profiles:
  ac,        Primarily AC Charging, Auto performance
  express,   Express charging, Low performance

Performance:
  auto,     Automatically adjust performance
  cool,     Set performance profile to cool
  desktop,  Highest performance settings
  laptop,   Laptop performance settings
  low,      Set perforamnce to power saving
  quiet,    Set performance profile to quiet
```

#### Reboot check

Prints information about what services need to be restarted, and whether the
kernel is old

Used to generally inform, whether the system should be rebooted

Can be wrapped with pacman hook to give information after update

```
Options:
  -d, --detailed,  Print more detailed information
```

#### Xwayland clients

Count xwayland clients

Arguemnts:
  format,  A format string in which '%n' will replace number of Xwayland clients
           if there are any. If there are none, nothing will be printed.
