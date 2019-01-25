---
toc: true
toc_label: "i3"
toc_icon: "beer"
toc_sticky: false
thumbnail: /assets/images/i3/ubuntu-filled.png
---

## Background
After months of using `tmux` as a dev environment I'm starting to see just how much I like the idea of a tiling window manager. I've spent nearly all my time on xfce systems and I'm interested what else is out there. I'm going to walk through the install of [i3](link) starting from a fresh [Ubuntu](http://releases.ubuntu.com/18.04/) install, and hopefully make it pretty by the end.

## Getting Started
To start off I'll provision a new Virtual Machine in [VirtualBox](https://www.virtualbox.org/) with a decent amount of memory and disk space to accomodate software development. 4 gigs of RAM and 32 gigs for storage is probably more than I'll need, I can always adjust this later. I'll start off with this Ubuntu 18.04 iso, which has Gnome on it by default, then just finish the install onto the virtual disk.

### Installing i3-gaps
Off the bat Gnome feels sluggish on VirtualBox and I'll leave everything as is after installing system updates. [i3-gaps](https://github.com/Airblader/i3) is a fork of i3 that powers a lot of [eye-candy](https://reddit.com/r/unixporn) desktops and I'll go straight for it.

Starting with the docs
```
# clone the repository
git clone https://www.github.com/Airblader/i3 i3-gaps
cd i3-gaps

# compile & install
autoreconf --force --install
rm -rf build/
mkdir -p build && cd build/

# Disabling sanitizers is important for release versions!
# The prefix and sysconfdir are, obviously, dependent on the distribution.
../configure --prefix=/usr --sysconfdir=/etc --disable-sanitizers
make
sudo make install
```
Missing dependencies was predicted, those are listed further down in the install docs just `apt install` them

Now we can restart the VM. On the Ubuntu greeter pick an i3 session.

![image-center](/assets/images/i3/sessionpicker.png){: .align-center}

## Configuration

Hitting `Mod+Return` (where `Mod` is the `Super` key by default ) we can launch a terminal. Launching a second and we see there are no gaps! Poking around the `i3-gaps` README it looks like I need to edit `~/.config/i3/config` and add 

```ini
# Disable window titlebars
for_window [class"^.*"] border pixel 0

# Global Gap Settings
gaps inner 30
```

This removes the title bar from applications, apparently necessary. I also added one of the example gaps options so that I can see it take effect on reload. In this same file we can find that `mod+Shift+r` will is the default binding to restart i3, needed to see the config change take effect

There are a multitude of `gaps` arguments, but for now I'll just use `inner` to see how it looks.

![image-center](/assets/images/i3/gaps.png){: .align-center}

Looks like the inner setting worked out. Going on a quick google hunt for an explanation of the `gaps` options I found a [post](https://classicforum.manjaro.org/index.php?topic=27260.0) on the Manjaro forums. It offers up some sensible options for the i3 config file to play around with gaps. It makes it pretty quick to discover a good `gaps inner/outer px` setting, it also seems handy to keep around for on the fly gap resizing.

### Keybind customization
The main reason I wanted to try out `i3` is because I'm starting to get annoyed with `tmux`. `tmux` is my daily driver for managing my terminal, but doesn't play well at all with color settings and I've spent enough work hours fiddling with `set T_Co` and `set -g default-terminal` that moving forward it would be nice to go without that particular fuss. `tmux` is incredibly powerful but having pane management only within the terminal is frustrating.

With that said, my brain is decently hardwired for my custom `tmux` bindings. Possibly against the grain, but `Mod+Shift+q` is just not comfortable to press. Here's my first pass at adjusting keybindings (careful to re-assign any overwrites)

```ini
# These are more similar to what I'm used to in tmux
bindsym $mod+q kill
bindsym $mod+Shift+backslash split h
bindsym $mod+minus split v
bindsym $mod+z fullscreen toggle

# These were jkl; which may be homerow sensible
bindsym $mod+h focus left
bindsym $mod+j focus left
bindsym $mod+k focus left
bindsym $mod+l focus left 
```

### Appearance
I defined a few variables with `set $name-color` to make it a easier to tweak, and used the color codes from the Arc-Dark theme. In my i3 config I added the following

```ini
# Settled on this default
gaps inner 20
gaps outer 10

# Removed the status bar (just shows workspaces for now)
bar {
	colors {
		background $bg-color
		focused_workspace $bg-color $bg-color $text-color
		inactive_workspace $inactive-bg-color $inactive-bg-color $inactive-text-color
		urgent_workspace $urgent-bg-color $urgent-bg-color $text-color
	}
}


# Not too sure how I'll use workspaces, but setting this as an example
# λ here was a lambda unicode character from an installed font
set $workspace1 "1:λ"
...
# Changing how the workspace is displayed on the status bar
bindsym $mod+1 workspace $workspace1
...
bindsym $mod+Shift+1 move container to workspace $workspace1
```
I also found an alternative to `dmenu` called `Rofi` which I installed, switching the keybindings a bit since I'm used to `Win+R` already but don't usually use an application finder on Linux.
```ini
bindsym $mod+r exec rofi -show run -config /home/hoenn/.config/rofi/config
```
The `rofi` config just contains a theme I found on Github, edited to remove the odd alternating striped rows.

```ini
rofi.theme: ~/.config/rofi/themes/arc-dark.rasi
```

The final piece of configuration for now, at least before digging into the `~/.vimrc` I've been carrying for years, was to pretty up the terminal's color scheme. I hadn't installed a different terminal emulator from the default, I imagine that's likely to change. 

I found [pywal](https://github.com/dylanaraps/pywal), this will generate colorschemes from a wallpaper. There's some extra config to have it theme _all the things_ but I decided to keep it local to the terminal, leaving the i3 bar and anything else alone. I fully expect to change my theme far more often when it's this easy.

```ini
# ~/.config/i3/config : regenerate the pywal colorscheme on startup, this also sets the wallpaper
# so we don't need to exec_always feh
exec_always --no-start-up-id wal -i ~/Pictures/wallpaper.jpg

# ~/.zshrc: required for new terminals windows to keep the color theme
(cat ~/.cache/wal/sequences &)

# ~/.xinitrc: would need to add this here, but don't since i3 exec_always takes care of it
wal -R
```


## Conclusion
 Overall I think I vastly overestimated how hard it would be to at least get going with i3, having only heard about it from co-workers and admired from afar. There are a lot of resources to get going, though I had really bad results with the `compton` compositor and couldn't find many easy answers there. Next I'd like to get the rest of my usual kit setup on this new system.

![image-center](/assets/images/i3/last.png){: .align-center}
