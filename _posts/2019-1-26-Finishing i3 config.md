---
toc: true
toc_label: "i3"
toc_icon: "beer"
toc_sticky: false
thumbnail: /assets/images/i3/terminalico.png
---
<link href="https://cdn.rawgit.com/Killercodes/281792c423a4fe5544d9a8d36a4430f2/raw/36c2eb3e0c44133880485a143717bda9d180f2c1/GistDarkCode.css" rel="stylesheet" type="text/css">

In the previous post: I set up a workspace variable with `set` and created an icon for workspace 1. I'd like to use workspace 2 to host VSCode. I'd always run this program in full screen, particularly because I use it for its markdown preview capabilities. 

## Workspace
```ini
# Unrelated, but I prefer not to change focus when I move my mouse
focus_follows_mouse no

# Set an icon and create the workspace 2 variable
# Just like before I actually pasted in the glyph for (pen-unicode) 
set $workspace2 "2:(pen-unicode)"
# Always open VSCode in workspace2
assign [class="Code"] $workspace2
```

To find `class="Code"` I used `xprop` in the terminal and used the entry for `WM_CLASS(STRING)`

I haven't thought of any other "set" workspaces I would want, but if I wasn't working in a VM I would likely create a separate workspace for a fullscreen browser. The predicitability for opening applications with `assign` is really handy for keeping things consistent. It's something I end up doing manually on reboot in Windows that could be avoided if I could save where I keep my application windows.

### Theming and revisiting Compton
Running in VirtualBox seems to come with a host of graphical issues. As far as I can tell I'm not going get the `compton` effects I want without tearing. 

To launch `compton` (and reload `pywal`) on restart I use `exec_always
```ini
# I keep a folder with one image as a current wallpaper
# it's a bit cleaner to just have current.jpg, but this
# ends up being more comfortable: New wallpaper? Move the previous
# out of /current and move the new one in
exec_always --no-startup-id wal -i ~/Pictures/current/ -q

exec_always compton --config ~/.config/compton/config
```

and my minimal `compton` config:
```ini
active-opacity=0.95;
inactive-opacity=0.85;

shadow = false;
fading = false;
```

I also decided to let `pywal` generate a rofi theme, which is real easy to setup. When ran a theme is generated in `~/.cache/wal/colors-rofi-dark.rasi`

## Running through old dotfiles
While I do keep a git repo of my dotfiles, I'm pretty bad at keeping it up to date as I tweak things. Since `pywal` is handling color schemes and I'm not planning to work directly in `tmux` much, I'm going to trim the fat a bit.

I'll rely on [Plugged](link) to get a few features I'm used to.

<script src="https://gist.github.com/Hoenn/9a30c9cd6c8b986a31d7e5aecc69c07f.js"></script>

These settings are porbably pretty standard but I've grown very used to leveraging `Ctrl+P`, buffers, and appreciate the visual indication the airline brings. `dylanaraps/wal.vim` is a great plugin for catching the `vim` colorscheme up to pywal's generated color scheme, even gets airline looking right!

### bash or zsh?
At this point I'm giving `zsh` a try. I decided to load download `oh-my-zsh` to see what a final product might look like. I'm using the `oxide` external theme, it's very simple providing environment info and branch/state indicators.

 I have a lot of `alias` defined in my old `~/.bashrc` but I won't copy those over into `zsh` until I actually see the need that had me define them.

 I don't have any particular reason to use `zsh` over `bash`, but in the spirit of having a side project, it'll be nice to see what I'm missing out on. I ended up installing plugins for bash to accomplish some of the "out of the box" features I have on `zsh`, but I'm ignorant of any differences so far that would annoy me.

 ## Conclusion

 A bit of a shorter post, but there wasn't that much to do, certainly not much I wasn't familiar with so far. I like the current state of this machine and it was a fun experiment to have something to write about, even if a bit simple and redundant to recite available resources. 
 
 Configuring a DE or WM, getting your favorite programs installed and running... it's an inspiring ritual that has me excited to get to work.