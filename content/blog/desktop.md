---
title: "bspwm - minimalistic and efficient desktop"
date: 2022-04-19
description : "My desktop setup with bspwm."
slug : "bspwm-desktop"
tags : [ "bspwm", "desktop" ]
categories : [ "FreeBSD" ]
---

I would like to describe the concept of my ideal desktop. The main values are simplicity - which means only necessary services and dependencies installed; efficiency - desktop actions that can be executed by keyboard shortcuts; and customisable - so I can add or remove whatever functionality I want.

The concept is very different from what I have been using my whole life. So what changed my mind? The idea came from a series called [Ricing your \*nix desktop](https://telegra.ph/Ricing-your-nix-desktop-epI-01-18), published on FreeBSD Telegram channel. That's where I got inspired to build my own lightweight desktop, instead of relying on pre-configured ones like KDE, GNOME, etc. It helped me to overcome the knowledge gap I had and it also introduced me to the idea of tiling managers. Because I was already happy with using `vim` fully from keyboard, I was quite keen to see what would happen if I started controlling the whole desktop like this. And the result was fantastic!

## Tech Stack
Before we start this brief blog post about how I created my dream desktop, let's quickly have a look at the technical stack:

* [FreeBSD](https://www.freebsd.org) - operating system (13-STABLE)
* [X.Org](https://x.org/wiki/) - display server
* [slim](https://sourceforge.net/projects/slim.berlios/) - login manager
* [bspwm](https://github.com/baskerville/bspwm) - tiling window manager
* [sxhkd](https://github.com/baskerville/sxhkd) - X hotkey daemon
* [polybar](https://github.com/polybar/polybar) - toolbar

## Display Server and OS
I have already explained my reasons for choosing FreeBSD over macOS in previous blog post. You can very easily follow these instructions on any GNU/Linux distribution, if you know the basics like package manager, usual file locations, etc. There's no particular benefit of using FreeBSD over GNU/Linux with these tools, it's just a matter of personal preference.

I chose X.Org display server instead of Wayland. Same with the OS, it's my personal choice, since you can easily find tiling managers on Wayland too (like `sway` for example). I have briefly experimented with Wayland, but found no advantages in using it over X.Org server, only more problems. I know how X.Org server works, I can easily configure it and it does everything I need. There's plenty of material to read about cons & pros of both solutions on the internet, so I won't go into more details here.

## Login Manager
Slim is simple to install and configure login manager. I chose it over the others, because I like its FreeBSD black & red colour scheme that's available as an extra package. It matches nicely with the colours of my ThinkPad laptop. It also doesn't pull in any heavy dependencies on QT/KDE, GNOME, etc. Here's how to install it:

```
# pkg install slim slim-freebsd-black-theme
# sysrc slim_enable=YES
```

Because I compiled my own version of the package without D-Bus support, I didn't have to activate it (normally done via `sysrc dbus_enable=YES`). It's because D-Bus service doesn't provide any functionality I need and only adds unnecessary complexity and potential security weaknesses as well (like PolKit).

I also changed the `current_theme`'s value by switching to `slim-freebsd-black-theme` in the configuration file located at `/usr/local/etc/slim.etc`. Here's how it looks on my laptop: 

![slim black theme sporting nice FreeBSD logo with username input field](https://bytebucket.org/rigoletto-freebsd/slim-freebsd-black-theme/raw/962413abb6abe7622bdfd554bde1591547254ee7/src/preview.png)

## X.Org Session
So far pretty simple and standard stuff. The truly fun part starts with defining the after-login process, which is required to start the window manager (WM) and configure X.Org server to my liking. You can find my `.xinitrc` below:

```
# merge all .Xresources
xrdb -merge ~/.Xresources

# fonts
xset +fp /home/marian/.local/share/fonts
xset +fp /usr/local/share/fonts/tamsyn-font
xset +fp /usr/local/share/fonts/powerline-fonts
xset fp rehash

# keyboard mapping
xmodmap ~/.Xmodmap

# set custom background
~/.fehbg

# cursor
xsetroot -cursor_name left_ptr

# keyboard layout
setxkbmap -layout us,cz -option grp:ctrls_toggle
setxkbmap -option "terminate:ctrl_alt_bksp"

# launch selected WM
if [ -x "$1" ]; then
	exec "$1" # provided by /usr/local/share/xsessions/*.desktop file
else
	exec bspwm # no WM selected, use default session
fi
```

I will skip several configuration commands here, like the `.Xresources` and `.Xmodmap` files, to keep the blog post reasonably "short" as I promised in the beginning. If you want to see what these strange `*.desktop` files contain, here's my hand written example for bspwm.desktop file:

```
[Desktop Entry]
Name=bspwm
Exec=/usr/local/bin/bspwm
```

You might also wonder why I have support for multiple WMs there? Sometimes I'm in a retro mood and I want to enjoy the look&feel of NeXTSTEP, so I like to hit `F1` on the login screen and choose WindowMaker session instead. I could also install `wdm` (WINGs Display Manager) to have the login manager in the same style, but I never bother, because I would have to reconfigure and restart the whole laptop.


## Window Manager
It's time to get introduced to the core components of my desktop - `bspwm` and `sxhkd`. What sort of fresh madness are these acronyms, you ask? `bspwm` stands for Binary Space Partitioning tiling window manager, `sxhkd` stands for Simple X Hotkey Daemon.

There you go. Now, with you being properly educated, let's install the packages with `# pkg install bspwm sxhkd` and move straight to their configuration.

### bspwm configuration
Here's the content of `~/.config/bspwm/bspwmrc` configuration file. The file must be executable, because bspwm is configured by `bspc` commands. The spacing between arguments doesn't make a difference, it's just there to make it prettier.

```
#!/bin/sh

# keyboard daemon
~/.config/sxhkd/start.sh

# polybar
~/.config/polybar/start.sh

# eDP-1 (internal display), DP-1 (external display)
bspc monitor eDP-1 -d 1 2 3 4 5 6 7 8 9 10
bspc monitor DP-1  -d 1 2 3 4 5 6 7 8 9 10

bspc config border_width        2
bspc config window_gap          5

bspc config split_ratio         0.5
bspc config borderless_monocle  true
bspc config gapless_monocle     true
bspc config single_monocle		true

# app configurations
bspc rule -a URxvt:nnn      state=pseudo_tiled center=true
bspc rule -a URxvt:htop     state=floating     center=true
bspc rule -a *:*:Calculator state=floating     center=true
bspc rule -a URxvt:mixertui state=floating     center=true
bspc rule -a URxvt:cal      state=floating     center=true
```

I use 10 desktops in the following order: web, email, RSS, music, with the last desktop assigned to a password manager. I use the rest for various apps, like `vim` for writing and note taking, `RetroArch` for gaming, etc.

I start `sxhkd` and `polybar` only as part of `bspwm` initialisation process, so it doesn't run when I start WindowMaker. The content of my bspwm configuration file comes mostly from the example file located at `/usr/local/share/examples/bspwm/bspwmrc`.

### sxhkd start script
`sxhkd` daemon allows to bind various actions like desktop switching or changing size and position of the windows to global shortcuts. It can also run various apps like `urxvt`, `musikcube`, etc. directly, using memorable shortcuts. I use `dmenu` for launching the remaining apps that I run infrequently.

I will show here only the start up script itself, leaving the lengthy and mostly copy&paste configuration file aside:

```
#!/bin/sh

# kill the current instance first
pkill sxhkd

# wait until polybar dies
while pgrep -u $UID -x sxhkd >/dev/null; do sleep 1; done

# start new instance
sxhkd &
```

It might look pretty simple to a seasoned sysadmin, but it took me a while to get it right. The main point here is to kill all the child processes like `sxhkd` and `polybar` when bspwm is restarted.

I took most of the sxhkd configuration from the example file at `/usr/local/share/examples/bspwm/sxhkdrc`.

## polybar
Polybar is responsible for creating and customising status bars. bspwm itself does not provide this functionality out of the box and leaves the choice of the tool on the user. I agree with this idea of being focused only on one thing and do it well. I started with `lemonbar` instead, but I found it quite cumbersome and not flexible for my needs (I couldn't figure out how to use emoji there).

Because I spent countless hours on polishing my status bar, I must expose my latest iteration of it here. The configuration file itself is stored in `~/.config/polybar/config.ini`:

```
[bar/main]
	width = 100%
	height = 22

	font-0="Noto Sans:size=12;2"
	font-1="Wuncon Siji:size=4;2"

	modules-left = bspwm
	modules-right = keyboard bhyve volume battery date

	wm-restack = bspwm
	enable-ipc = true

[module/keyboard]
	type = internal/xkeyboard
	format = <label-layout>
	format-prefix = ""
	format-prefix-padding = 2
	format-prefix-foreground = #ff5370

[module/bhyve]
	type = custom/script
	exec = ~/.config/polybar/bhyve.sh
	interval = 5

	format-prefix = ""
	format-prefix-padding = 2
	format-prefix-foreground = #ff5370

[module/volume]
	type = custom/script
	exec = ~/.config/polybar/volume.sh
	interval = 5

	format-prefix = ""
	format-prefix-padding = 2
	format-prefix-foreground = #ff5370

	click-left = "urxvt -name mixertui -e mixertui"

[module/battery]
	type = custom/script
	exec = ~/.config/polybar/battery.sh
	interval = 5

	format-padding = 2
	format-prefix-foreground = #ff5370

	click-left = ~/.config/polybar/power-menu.sh

[module/date]
	type = internal/date
	interval = 30

	date = %d. %m.
	time = %H:%M

	label = "%{F#ff5370}%{F-} %date% %{F#ff5370}%{F-} %time%"
	label-padding = 2

[module/bspwm]
	type = internal/bspwm

	format = <label-state> <label-mode>
	label-focused-foreground = #ff5370
```

That's a really long file, isn't it? Let's dissect it piece by piece together. 

### Fonts
`polybar` doesn't provide any support for icons, but it does support emojis. I'm using two fonts here - Noto and Siji. Noto can be installed from packages via `# pkg install noto-basic noto-emoji`. Siji is not available in ports, so it must be installed manually as described on [Siji home page](https://github.com/stark/siji).

### bhyve monitor script
I love using bhyve for running various VMs on my laptop. Here's an extremely simple monitor script that tells me how many VMs I run at the moment.

```
#!/bin/sh

if [ -d "/dev/vmm" ]
then
	INSTANCES="$(ls /dev/vmm | awk 'END{print NR}')"
else
	INSTANCES=0
fi

echo "$INSTANCES"
```

### volume monitor
`polybar` volume module does not support FreeBSD audio system, so I created my own implementation of it using `mixer` command and some dark `awk` magic, which I barely understand. Yes you are right, I took it mindlessly from the internet:

```
#!/bin/sh

GETVOL="$(mixer vol | awk '{ print $7 }' | grep -o '[^:]*' | uniq)"
echo "$GETVOL%"
```

### power menu
It took me a while to persuade myself that spending 15 minutes on implementing power menu is a better idea than simply hit Meta-Enter and type `doas poweroff` on the terminal every time I want to switch off the laptop. I couldn't figure out how to use Polybar's pop-up menu, so I resigned to a quick & dirty trick with `jqmenu`.

```
#!/bin/sh

printf "Restart,doas reboot\\nShutdown,doas poweroff\\n" | jgmenu --simple --at-pointer
```

The important note here is that `doas` configuration must allow the user to run these two commands without password. I'm not crazy enough to bother with implementing an authentication dialogue window for it here.

### battery monitor script
Another `polybar` module I had to implement by myself, because it works only on GNU/Linux. Luckily, it's brain dead simple to get the information from the system, provided you know where to look. Here comes a true masterpiece of my shell script programming capabilities, hang on to your hats:

```
#!/bin/sh

CHARGE="$(sysctl hw.acpi.battery.life | awk '{ print $2 }')%"
STATE="$(sysctl hw.acpi.battery.state | awk '{ print $2 }')"

case $STATE in
	0) # full
		LABEL=""
		VALUE="$CHARGE"
		;;
	1) # discharging
		LABEL=""
		VALUE="$CHARGE"
		;;
	2) # charging
		LABEL=""
		VALUE="$CHARGE"
		;;
	5) # discharging - critical
		LABEL=""
		VALUE="$CHARGE"
		;;
	6) # charging - critical
		LABEL=""
		VALUE="$CHARGE"
		;;
	7) # no battery
		LABEL=""
		VALUE=""
		;;
	*) # error
		VALUE="ERR"
		;;
esac

echo "%{F#ff5370}$LABEL%{F-} $VALUE"
```

## The end
Congratulations for reaching the end of this monstrously long blog post. Didn't I say I will try to keep it short? Yeah sorry, that was just a poor marketing trick to force you to read the whole thing. Anyway, I hope that it will help somebody to get inspired by it and perhaps to try something different on their desktop. See you next time!

