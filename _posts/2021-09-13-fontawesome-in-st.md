---
title: "How to install fontawesome glyphs in st terminal emulator"
header:
  image: /assets/images/2021-09-13-fontawesome-banner.png
tags:
  - Linux
  - fonts
  - suckless
  - st
---

If you're like me and new to the Linux scene you may be wondering how all the l33t users manage to have their nice glyphs in their terminal editors. I am particular to [suckless software](https://suckless.org/) so I needed these glyphs to work in [st](https://st.suckless.org/). This turned out to be a headache so here I am documenting the process.

# Requirements
For those following along at home all you are going to need is to make sure to have st with the [font2](https://st.suckless.org/patches/font2/) patch installed. If you don't know what that means you can follow my guide on patching suckless software which will be made in the future, in the meantime you can use an already patched version of st such as [Luke Smith's fork of st](https://st.suckless.org/patches/font2/). Luke's fork has great instructions on how to install it.

Once you have st installed you also need to make sure to install Font Awesome. On my arch system this looks like:

```console
$ sudo pacman -S ttf-font-awesome
```

# Optional Font Config
I highly recommend this part as it makes the rest of the steps easier.

When Linux needs a font it tries to match a string to a font name using the utility `fc-match`. You can try running it alone to see what your systems default font is. What we want is for `fc-match fontawesome` to match to `fa-solid-900.ttf: "Font Awesome 5 Free" "Solid"` so that we don't have to type the full name. To do this we will have to edit your system's font config which lives in `/etc/fonts/font.conf` or your user font config which lives in `~/.config/fontconfig/fonts.conf` (recommended) and add the following config lines.

Add for fontawesome abbreviation

```xml
<!--
  Accept alternate 'fontawesome' spelling
-->
    <match target="pattern">
        <test qual="any" name="family">
            <string>fontawesome</string>
        </test>
        <edit name="family" mode="assign" binding="same">
            <string>Font Awesome 5 Free Solid</string>
        </edit>
    </match>
```

Add for fontawesomebrands abbreviation

```xml
<!--
  Accept alternate 'fontawesomebrands' spelling
-->
    <match target="pattern">
        <test qual="any" name="family">
            <string>fontawesomebrands</string>
        </test>
        <edit name="family" mode="assign" binding="same">
            <string>Font Awesome 5 Brands</string>
        </edit>
    </match>
```

# ST Font Awesome config
Once you make sure you have st and Font Awesome 5 installed head to the `config.h` file in your st's directory.

If there isn't a line that looks like:

```c
static char *font2 = "{font name here}:pixelsize=12:antialias=true:autohint=true";
```
Make sure to add one after 

```c
static char *font = "{default font name here}:pixelsize=12:antialias=true:autohint=true";
```

In this field you are going to want to add the Font Awesome 5 fonts with a string that `fc-match` will resolve to `fa-solid-900.ttf: "Font Awesome 5 Free" "Solid"` and `fa-brands-400.ttf: "Font Awesome 5 Brands" "Regular"`. For those who did the optional configuration these should be `fontawesome` and `fontawesomebrands` respectively.

The field will look something like this.

```c
static char *font2[] = { "fontawesome:style=Solid:pixelsize=14:antialias=true:autohint=true",
                         "fontawesomebrands:style=Solid:pixelsize=14:antialias=true:autohint=true",
                         "emoji:style=Solid:pixelsize=14:antialias=true:autohint=true" };
```

## Emoji Font
I have a line for an emoji font because without it, inputting some emojis into st will crash it, though I don't care what system emoji font it uses so `emoji` works for me.

## Finishing installation
To solidify your config don't forget to rebuild st with the command

```console
$ sudo make clean install
```

and all the new instances of st that you spawn should have Font Awesome 4 glyph support!


# Alternatives
If doing all of these steps seem too tedious for you, you can always do the optional font config step and then download [my fork of Luke Smith's fork of st](https://github.com/zoomiti/st) that comes with all the configuration you need to get up and going with font awesome glyphs in your terminal.
