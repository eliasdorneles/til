---
title: "How to use PipeWire for music production with Reaper on Linux"
date: "2023-11-15T21:47:12+01:00"
tags:
  - Linux
  - music
  - MIDI
  - audio
  - Reaper
  - PipeWire
---

That's it, I've made the switch to [PipeWire](https://pipewire.org/)!

And things are looking great!

I was previously using [JACK](https://jackaudio.org) and
[PulseAudio](https://en.wikipedia.org/wiki/PulseAudio) in parallel, each with
its own soundcard, and the setup was a bit cumbersome as for some reason I had
not been able to get everything to load at startup, so I had to start JACK
manually everytime...

Now with PipeWire, everything is integrated, no need to choose anymore!

I am on Ubuntu 22.04, and had to do a few things in order to get it to work
correctly for my setup, which is essentially about being able to record
audio and MIDI with [Reaper](https://www.reaper.fm) and my sound cards.

Things that I had to do:

- use the [upstream Debian packages of
  PipeWire](https://pipewire-debian.github.io/pipewire-debian/), as the
  packages provided by Ubuntu were about an year behind and missing some
  important improvements
- in Reaper, I had to "reconnect" the MIDI devices, and it was a bit confusing
  initially as they showed up only at the end of a looong list of other
  "phantom" MIDI devices. [It has also happened to someone else other than
  me](https://www.reddit.com/r/linuxaudio/comments/vr6z8w/cannot_connect_midi_keyboard_to_reaper_after/).
- to get the low latency in Reaper, I had to set the environment variable
  `PIPEWIRE_LATENCY`, which is used by PipeWire libraries that replace Jack.
  I did this by modifying my Reaper .desktop file, it now looks like this:

```toml
[Desktop Entry]
Name=Reaper
Exec=env PIPEWIRE_LATENCY=256/48000 /home/elias/apps/reaper_linux_x86_64/REAPER/reaper
Icon=/home/elias/apps/reaper_linux_x86_64/REAPER/Resources/main.png
Type=Application
Categories=AudioVideo;
```
