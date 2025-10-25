---
title:  "Code Completion With CAPS-LOCK (not Tab!)"
layout: post
---

The overloading of the Tab key to mean "indent" and "complete code" has always
been pretty horrible.  Now it's intolerable: suggestions from AI come out of
the blue at any moment, and can hijack what would have been an indent
nanoseconds before.  The Tab key becomes completely unreliable in function.

This infuriated me enough to do something about it.

I decided to turn Caps Lock into an actually useful key.

It isn't too hard to do... *if you know what to do*.  Here's a handy guide for
how to do it in a way that is consistent across Windows, macOS, and Linux!

*(If using VSCode, see extra notes at the bottom regarding the keymapping.)*

## WINDOWS: [PowerToys Keyboard Manager][1]

[1]: https://github.com/microsoft/PowerToys?tab=readme-ov-file

VSCode is actually able to map the caps lock key to completion without any
special tools... BUT... *it won't stop the key from toggling the capitalization
state*.  If your Caps Lock has a light it will go on and off, and each time
you do a completion it will toggle the casing.

So use [Microsoft PowerToys][1] *(it's open source!)*, specifically **Keyboard Manager**.

<img src="https://ae1020.github.io/assets/images/powertoys-caps-lock-remap-to-f19.png" alt="Powertoys Caps Lock Remap to F19" style="width: 75%; max-width: 663px; height: auto;" />

I've remapped the Caps Lock key to **F19**.  My keyboard only has 12 function
keys, so I'm not bumping anything I'd use out of the way.

*(At first I didn't realize these extra function keys were available... and
tried the "virtual keys" you can assign, with names "VK 1" up to "VK 252".  But choosing something in this range is a bit fraught... e.g. VK 1 is actually the
mouse button.  And they show up as "unknown" in the VSCode keyboard mapping.)*

The strange choice of F19 (when it goes up to F24) is because the Mac solution
can only go up to F20, and the Linux solution has a default collision that you
have to do special overrides for F20.  If you want to be able to share your
keymaps across your computers, F19 seems a good choice.

## MACOS: [Karabiner Elements][2]

[2]: https://karabiner-elements.pqrs.org/

Apple lets you swap Caps Lock for another modifier key (like Shift or Ctrl),
or disable it entirely.  But you can't make it do anything specific as a key
out of the box.

A program called [Karabiner Elements](https://karabiner-elements.pqrs.org/)
does a bunch of useful things ("and it's only a hundred megabytes", sigh).

You'll have to approve a list of scary-sounding permissions that you are
installing an app and driver that sees every key that you press.  But,
Karabiner Elements is also open source:

<https://github.com/pqrs-org/Karabiner-Elements>

And they seem generally trustworthy enough in the scheme of things, to use
their binaries, but... I guess it depends on how badly you want to remap the
Caps Lock key.

It's in "Simple Modifications":

<img src="https://ae1020.github.io/assets/images/karabiner-caps-lock-remap-to-f19.png" alt="Karabiner Caps Lock Remap to F19" style="width: 75%; max-width: 690px; height: auto;" />

Note that if you previously had the OS doing mappings of your modifier keys,
this overrules that.  So you'll have to reconfigure your swappings of
Command/Ctrl/Fn/etc.

## LINUX: [`keyd`][3] (key daemon)

[3]: https://github.com/rvaiya/keyd

You would think a configuration thing like this would be easy on Linux, but it
took me and the AI quite a while of trying things with no effect before a
solution was found.

Some files sure *look* like editing them should be able to produce the effect.
`/usr/share/X11/xkb/symbols/pc` and `/usr/share/X11/xkb/keycodes/evdev` seem
like source code for specifying the entirety of your keyboard's behavior.  But
when it comes to the caps lock key, no dice--about the best I could accomplish
was get it to not do anything (though the light on the key toggled on and off).

The keyd project seems to be the right solution to this problem, and works on
x11 or Wayland, and it's got some cool features too.  But it's not in the
Debian repositories so you have to build it from source:

    git clone https://github.com/rvaiya/keyd
    cd keyd
    make
    sudo make install
    sudo systemctl enable keyd
    sudo systemctl start keyd

Once you've got it, you edit `/etc/keyd/default.conf`:

    [ids]
    *

    [main]
    capslock = f19

Then you restart the daemon:

    $ sudo systemctl restart keyd

If you're lucky, that should work!  But if your `xmodmap` contains instructions
that route the keys to functions, you won't get them falling through to where
VSCode and such can see them.  That's what's wrong with F20, it defaults to
mapping the microphone input.  You can see it with a grep for F20's keycode,
which is 198:

    $ xmodmap -pke | grep 198
    keycode 198 = XF86AudioMicMute NoSymbol XF86AudioMicMute

Your Linux installation might have made other arbitrary choices, perhaps even
with F19.

To get into the weeds with xmodmap, you can edit your `~/.Xmodmap` file:

    keycode 198 = F20

Load it to see the changes:

    $ xmodmap ~/.Xmodmap

For me, that worked to let me map it for F20.  But in this handy guide, I'm
just suggesting using F19 so you don't have to go further with adding that
xmodmap persistently to your startup files.

## Then Remap Completion to F19 In Your Editor

I'll mention that in VSCode, it turns out that there's not a single place to
change "Accept Suggestion" from Tab to F19.  Instead, you have to use the
search box (helps if you change the sort order to "Sort by Precedence").

You'll see something like this:

<img src="https://ae1020.github.io/assets/images/vscode-search-to-remap-tab.png" alt="VSCode Search To Remap Tab" style="width: 100%; max-width: 522px; height: auto;" />

Wherever it looks like it's completion-related, change it.

It didn't take me long to acclimate to using Caps Lock for completion, and I
hope this can help others be less frustrated as well!
