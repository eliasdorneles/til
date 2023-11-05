---
title: "How to call fluidsynth C API from Zig"
date: "2023-11-05T17:02:37+01:00"
tags:
  - zig
  - fluidsynth
  - C-library
  - midi
  - sound-synthesis
---

I find quite cool how quickly you can start using a C library from
[Zig](https://ziglang.org/)! No need to write bindings or to fiddle with a FFI,
[it provides direct integration with C
libraries](https://ziglang.org/learn/overview/#integration-with-c-libraries-without-ffibindings).

Today I decided to play with the [fluidsynth C
API](https://www.fluidsynth.org/api/index.html), can call it from Zig.

First, I tried out the [C
example](https://www.fluidsynth.org/api/example_8c-example.html) from
Fluidsynth documentation, compiling it as described in there, which worked like
a charm.

> **Note:** I am using Ubuntu 22.04, and had previously installed the package
> `libfluidsynth-dev`. If you want to reproduce the results, you'd need to find
> the equivalent for your system. You'll also need to provide a sound font,
> possibly updating the path to it in the code. I've used the [GeneralUser GS
> soundfont](https://schristiancollins.com/generaluser.php).

Then, I went on to write a Zig version, here it is:

```zig
const std = @import("std");
const fluid = @cImport(@cInclude("fluidsynth.h"));

pub fn main() void {
    const settings = fluid.new_fluid_settings();
    const synth = fluid.new_fluid_synth(settings);

    // here we load the soundfont file
    _ = fluid.fluid_synth_sfload(synth, "soundfonts/GeneralUser_GS_v1.471.sf2", 1);

    // this triggers the synthesizer to play whatever events
    // we send later
    _ = fluid.new_fluid_audio_driver(settings, synth);

    var key: i16 = 60;
    const note_length = std.time.ns_per_s / 4; // 1/4th of a second

    // now, let's send MIDI note ON and OFF events...
    _ = fluid.fluid_synth_noteon(synth, 0, key, 100);
    std.time.sleep(note_length);
    _ = fluid.fluid_synth_noteoff(synth, 0, key);

    key += 2;
    _ = fluid.fluid_synth_noteon(synth, 0, key, 100);
    std.time.sleep(note_length);
    _ = fluid.fluid_synth_noteoff(synth, 0, key);

    key += 2;
    _ = fluid.fluid_synth_noteon(synth, 0, key, 100);
    std.time.sleep(note_length);
    _ = fluid.fluid_synth_noteoff(synth, 0, key);

    // wait for another note length, so that last note's full envelope sounds
    std.time.sleep(note_length);
}
```

When I first attempted to run it, I've got a segmentation fault:

```shell
$ zig run zig_fluid_example.zig -lfluidsynth
Segmentation fault at address 0x0
???:?:?: 0x0 in ??? (???)
Aborted (core dumped)
```

After some puzzling, a quick web search revealed out the problem: I needed to
link `libc`, as Zig does not link to it automatically. Adding `-lc` to the
command fixed it:

```shell
$ zig run zig_fluid_example.zig -lfluidsynth -lc
```

I am quite content with the result and look forward to playing more with this combo!
