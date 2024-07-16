# Using KDE Plasma with i3wm

This repository has useful information for configuring KDE Plasma to use i3 (rather than KWin) as your window manager. I might also write a script to automatically do it (or an AUR package that you can install to automatically apply the tweaks).

*Note: This guide is VERY hastily written, also primarily for Arch Linux. This is primarily based on my own experience configuring this, and does not represent all use cases / system configurations. While some things may be consistent across systems / distros / display managers / Plasma versions, your mileage may vary. Definitely double check package names as a good first step.*

# Install Packages

You may want to configure a display manager before you go too far. In my case, I use `sddm`:

```sh
pacman -S sddm
```

Install your desktop environment and window manager:

```sh
pacman -S plasma i3
```

Install `feh` for a working wallpaper on i3:
```sh
pacman -S feh
```

# Setting Your Window Manager

Plasma uses KWin as its window manager by default. You can switch this to i3 using one of two methods: first, you can configure a systemd unit (as of Plasma >= 5.25) or as a launch script. There are a couple of advantages to the latter method:

- This window manager configuration is available for all users (if you have a shared device)
- This will show up as an option in your display manager

## systemd

First, mask the default service:

```sh
systemctl --user mask plasma-kwin_x11.service
```

Then, create the following custom service file (at `~/.config/systemd/user/plasma-i3.service`):
```properties
[Install]
WantedBy=plasma-workspace.target

[Unit]
Description=Plasma Custom Window Manager
Before=plasma-workspace.target

[Service]
ExecStart=/usr/bin/i3
Slice=session.slice
Restart=on-failure
```

Then restart systemctl and enable the new service:

```sh
systemctl --user daemon-reload
systemctl --user enable plasma-i3.service
```

## Launch Script

If you are running Plasma >= 5.25, you will need to run the following command:
```sh
# Plasma 5
kwriteconfig5 --file startkderc --group General --key systemdBoot false
# Plasma 6
kwriteconfig6 --file startkderc --group General --key systemdBoot false
```

Then, create the following session entry (at `/usr/share/xsessions/plasma-i3.desktop`):

```properties
[Desktop Entry]
DesktopNames=KDE
Name=Plasma (i3)
Comment=Plasma with i3 Window Manager
Exec=/usr/local/bin/plasma-i3.sh
TryExec=/usr/local/bin/plasma-i3.sh
Type=Application
X-LightDM-DesktopName=plasma-i3
DesktopNames=plasma-i3
Keywords=tiling;wm;windowmanager;window;manager;kde;
```

Then the following script (at `/usr/bin/plasma-i3.sh`):

```sh
#!/bin/sh
export KDEWM=/usr/bin/i3
exec /usr/bin/startplasma-x11
```

Make sure it is executable:
```sh
chmod a+x /usr/local/bin/plasma-i3.sh
```

Log out and then log back in, making sure to set your desktop environment to your new "Plasma (i3)" entry.

# Configuring i3

Launch `i3` first to generate the default config file.

## Working with Plasma

Add the following lines to your `~/.config/i3/config`...

to automatically close the wallpaper and set it using `feh` (note the commented lines, select the one that is correct for your version of plasma):

```
# >>> Plasma Integration <<<
# Try to kill the wallpaper set by Plasma (it takes up the entire workspace and hides everything)
exec --no-startup-id wmctrl -c Plasma
# uncomment this line if you are using plasma < 5.27
#for_window [title="Desktop — Plasma"] kill; floating enable; border none
# uncomment this line if you are using plasma >= 5.27
#for_window [title="Desktop @ QRect.*"] kill; floating enable; border none
# set background using feh instead
exec --no-startup-id feh --bg-color black

# Compositor (Animations, Shadows, Transparency)
exec --no-startup-id picom -cCFb
```


to keep Plasma windows from tiling:

```
# >>> Window rules <<<
# >>> Avoid tiling Plasma popups, dropdown windows, etc. <<<
# For the first time, manually resize them, i3 will remember the setting for floating windows
for_window [class="yakuake"] floating enable;
for_window [class="lattedock"] floating enable;
for_window [class="plasmashell"] floating enable;
for_window [class="Kmix"] floating enable; border none
for_window [class="kruler"] floating enable; border none
for_window [class="Plasma"] floating enable; border none
for_window [class="Klipper"] floating enable; border none
for_window [class="krunner"] floating enable; border none
for_window [class="Plasmoidviewer"] floating enable; border none
for_window [title="plasma-desktop"] floating enable; border none
for_window [class="plasmashell" window_type="notification"] floating enable, border none, move position 1450px 20px
no_focus [class="plasmashell" window_type="notification"]

# >>> Avoid tiling for non-Plasma stuff <<<
for_window [window_role="pop-up"] floating enable
for_window [window_role="bubble"] floating enable
for_window [window_role="task_dialog"] floating enable
for_window [window_role="Preferences"] floating enable
for_window [window_role="About"] floating enable
for_window [window_type="dialog"] floating enable
for_window [window_type="menu"] floating enable
for_window [instance="__scratchpad"] floating enable
```

### Volume Controls
By default, i3 controls the volume using PulseAudio directly. You can configure i3 to pass those controls to KDE's dbus tools (which will display a nice dialog when you change volume) by replacing those lines with the following:

```
# Use dbus to control volume
bindsym XF86AudioRaiseVolume exec --no-startup-id qdbus org.kde.kglobalaccel /component/kmix invokeShortcut "increase_volume"
bindsym XF86AudioLowerVolume exec --no-startup-id qdbus org.kde.kglobalaccel /component/kmix invokeShortcut "decrease_volume"
bindsym XF86AudioMute exec --no-startup-id qdbus org.kde.kglobalaccel /component/kmix invokeShortcut "mute"
bindsym XF86AudioMicMute exec --no-startup-id qdbus org.kde.kglobalaccel /component/kmix invokeShortcut "mic_mute"
```

### Locking and Logout
i3's logout utility does not properly terminate the X session on its own, and neither does Plasma's. For now, you can replace i3's logout with plasma's:

```
# exit i3 (logs you out of your X session)
# Uncomment this for Plasma 6
#bindsym $mod+Shift+e exec --no-startup-id qdbus6 org.kde.LogoutPrompt /LogoutPrompt org.kde.LogoutPrompt.promptLogout
# Uncomment this for Plasma 5
#bindsym $mod+Shift+e exec --no-startup-id qdbus-qt5 org.kde.ksmserver /KSMServer org.kde.KSMServerInterface.logout -1 -1 -1
# Uncomment this for Plasma 4
#bindsym $mod+Shift+e exec --no-startup-id qdbus org.kde.ksmserver /KSMServer org.kde.KSMServerInterface.logout -1 -1 -1
```

A current workaround for fully terminating your X session is to manually do so through the command line (for shutting down use `sudo shutdown`, for rebooting use `sudo reboot`, if you want to log out completely restart your display manager in `systemctl`, etc.).

### i3bar
You can choose to disable i3bar if you would like and just use KDE's panels by deleting the `bar { ... }` lines.

### dmenu/rofi
Select the application you prefer for your launcher. Personally, I prefer `rofi`, but you can stick with `dmenu` if you think that's fine. Make sure that you have your preferred launcher installed (use one of the following):

```sh
pacman -S dmenu
pacman -S rofi
```

# Plasma Config Tweaks

These tweaks may be useful to make your Plasma experience a little bit less hellish.

## Keyboard Shortcuts

**DISABLE ALL THE TASK MANAGER QUICK LAUNCH KEYS. THIS WILL MAKE YOUR LIFE SO MUCH EASIER. GOOD LORD IT WAS ANNOYING TO TRY AND SWITCH DESKTOPS AND HAVE RANDOM WINDOWS POP UP.**

Under Keyboard -> Shortcuts -> plasmashell, uncheck all the shortcuts for "Activate Task Manager Entry #".

You may have issues with the "Show Activity Switcher" shortcut (bound to Meta+Q). I have not experienced this with Plasma 6, but in older versions it probably conflicts with the shortcut to close a window (Meta+Shift+Q). I guess it might have ignored the Shift key.

## Panel

If you notice a black bar on the bottom of your screen, you can right click the panel, click "Show Panel Configuration", then flip the "Floating" switch off to fix it.

# References

- [Using Other Window Managers with Plasma - KDE UserBase Wiki](https://userbase.kde.org/Tutorials/Using_Other_Window_Managers_with_Plasma#Configure_i3)
- [KDE - ArchLinux Wiki](https://wiki.archlinux.org/title/KDE#Use_a_different_window_manager)
- [KDE Plasma with i3wm - maxnatt](https://maxnatt.gitlab.io/posts/kde-plasma-with-i3wm/)
- [heckleson/i3-and-kde-plasma](https://github.com/heckelson/i3-and-kde-plasma)
- [How to logout from command line in plasma 6? - r/kde](https://www.reddit.com/r/kde/comments/1d3ecrs/how_to_logout_from_command_line_in_plasma_6/?rdt=56074)
- [Logout, shutdown, and reboot using the terminal - KDE Forums](https://discuss.kde.org/t/logout-reboot-and-shutdown-using-the-terminal/743/9)