---
title: "How to create a WAVE file"
date: "2024-07-29T00:30:55+02:00"
tags:
  - Zig
  - Python
  - wave
  - audio programming
---

Today I set out to learn how to create a WAVE file.

I had recently gone through an article about [reading and writing wave files in
Python](https://realpython.com/python-wav-files/#write-your-first-wav-file-in-python),
which whet my appetite to do somewhat the same in Zig.

First I had some fun playing around with Python in a Jupyter Notebook, which is
fun because you can use `IPython.display.Audio("your_wave_file.wav",
autoplay=True)` at the end of the cell where you generate your wave file, and
it will play it immediately upon execution. With the immediate feedback, it's
just very satisfying to play around with the sine waves.

Here is the Python code, to run in a Jypyter notebook:

```python
import array
import copy
import itertools
import math
import os
import wave
from IPython import display


FRAMES_PER_SECOND = 44100

INT_8FAC = 2 ** 8 - 1
INT_16FAC = 2 ** 16 - 1
INT_32FAC = 2 ** 32 - 1

bit_depth_factors = {
    1: INT_8FAC,
    2: INT_16FAC,
    4: INT_32FAC
}

def clip(value, sample_width=2):
    upper_limit = bit_depth_factors[sample_width] / 2
    lower_limit = -(bit_depth_factors[sample_width] + 1) / 2
    return int(max(lower_limit, min(value, upper_limit)))

def sound_wave(frequency, num_seconds, sample_width=2):
    for frame in range(round(num_seconds * FRAMES_PER_SECOND)):
        time = frame / FRAMES_PER_SECOND
        amplitude = math.sin(2 * math.pi * (frequency - frame * 0.001) * time)
        yield clip(round((amplitude + 1) / 2 * bit_depth_factors[sample_width]), sample_width=sample_width)

WAVE_FILE = "audio_out.wav"

with wave.open(WAVE_FILE, mode="wb") as wf:
    left = list(sound_wave(220, 5))
    right = list(sound_wave(225, 5))
    stereo_frames = array.array("h")
    stereo_frames.extend([x for it in zip(left, right) for x in it])
    wf.setnchannels(2)
    wf.setsampwidth(2)
    wf.setframerate(44100)
    wf.writeframes(stereo_frames.tobytes())

display.Audio(WAVE_FILE, autoplay=True)
```

I know, it's ugly, I didn't care to clean it up as it was just fun & play at
this point. I used this mostly to explore "how to generate audio data", that I
wasn't super familiar with.

Once I felt like I had a decent graps of how to generate the audio, I thought
"okay, time to learn how to do it in Zig and create the wave file from scratch!".

I read some articles online about creating wave files with Zig:

- https://blog.jfo.click/makin-wavs-with-zig/
- https://zig.news/xq/re-makin-wavs-with-zig-1jjd (review/critique of the previous one)

These were interesting and good to have as reference, but I wanted to do
something on my own, to really solidify the learning.

I landed into [this great article explaining the WAVE
format](http://soundfile.sapp.org/doc/WaveFormat/), and spent some time poking
around the wave files generated with Python with the script above in an
hexadecimal editor, to check my understanding.

### Aside: a little rant on hexadecimal editors

**Aside:** I tried three hexadecimal editors for Linux, and none were really
good for what I wanted: to get the little endian encoded value from a
sequence of bytes I selected.

`hexedit` is just hard to use, I couldn't figure out how to use quickly, so I
gave up.

[Bless](https://github.com/afrantzis/bless) has
the feature I want but its UI is very buggy: the sequence of bytes you select
doesn't match the sequence it displays, very frustrating!

[GHex](https://wiki.gnome.org/Apps/Ghex) works correctly (no bugs), but it
doesn't have the feature: it allows you to select a sequence of bytes, but it
only display the value for one byte at the time (the last selected).

Later I found [this online hex editor](https://hexed.it/) that has the feature
and it works, but I found the UI for selecting counterintuitive and also gave up on it.

I ended up using GHex and doing the calculations in a Python shell.

### Back to WAVE files and Zig!

Upon a first read about the WAVE format, I thought that the aspect of mixing
[little endian and big endian](https://en.wikipedia.org/wiki/Endianness) in the
same file would be annoying, but it turns out it's actually pretty intuitive.
Basically: all the "text" markers in the file are written in big endian (which
is the order you'd expect to read it, as a human), and all the rest in little
endian (i.e. starting from the least significant byte, which is more efficient
for the computer).

Luckily, Zig has a function `writeInt` that takes an `endianess` argument as a
parameter, so no needless bit fiddling required in your code, you can just do:

```zig
    try stdout.writeInt(u32, sample_size, Endian.little)
```

... and it will do all the required bit fiddling for you. (or should I say byte
fiddling instead? =P)

I wanted to be able to choose the filenames when running the program, but I
couldn't find an easy way to handle command-line arguments that in Zig that
didn't involve adding a library. To keep it simple, I decided to just write
the WAVE file in the standard output directly. Let the user do `> myfile.wav`
-- thank you, Unix philosophy!

So, without further ado, here is the code:

```zig
const std = @import("std");
const Endian = std.builtin.Endian;

/// Convert a floating point number to an integer, scaling it to the range of
/// the integer type and clipping it if necessary.
fn scaleFloatToInt(comptime samplewidth: u8, signal_value: f32) u16 {
    const int_factor = 2 << (samplewidth * (8 - 1));
    const scaled: f32 = (signal_value + 1) * int_factor;
    const clipped = if (scaled > int_factor) int_factor else scaled;
    return @intFromFloat(clipped);
}

/// Write a WAV file with a simple stereo signal to the stdout.
/// The left channel has a frequency that increases over time, while the right
/// channel has a frequency that decreases over time.
pub fn main() !void {
    const stdout_file = std.io.getStdOut().writer();
    var bw = std.io.bufferedWriter(stdout_file);
    const stdout = bw.writer();

    const header_size = 36;

    const samplerate = 44100;
    const duration = 10;
    const sample_size: u32 = duration * samplerate;
    const nchannels = 2;

    const samplewidth = 2;
    const bit_depth = samplewidth * 8;

    // This article explains the WAV format in detail:
    // http://soundfile.sapp.org/doc/WaveFormat/

    // We start with the RIFF header
    try stdout.writeAll("RIFF");

    // The size of the file, minus the first 8 bytes
    try stdout.writeInt(
        u32,
        header_size + sample_size * nchannels * samplewidth,
        Endian.little,
    );
    try stdout.writeAll("WAVE");

    // format chunk
    try stdout.writeAll("fmt ");
    try stdout.writeInt(u32, 16, Endian.little);

    try stdout.writeInt(u16, 1, Endian.little); // audio format (1 = PCM)
    try stdout.writeInt(u16, nchannels, Endian.little);

    try stdout.writeInt(u32, samplerate, Endian.little);

    // byte rate
    try stdout.writeInt(u32, samplewidth * samplerate * nchannels, Endian.little);

    // block align, i.e. number of bytes per sample
    try stdout.writeInt(u16, nchannels * samplewidth, Endian.little);

    // bit depth
    try stdout.writeInt(u16, bit_depth, Endian.little);

    // now, setup the data chunk...
    try stdout.writeAll("data");
    try stdout.writeInt(u32, sample_size, Endian.little);

    // and let's write some audio data!
    for (0..sample_size) |frame_idx| {
        const time = @as(f32, @floatFromInt(frame_idx)) / @as(f32, samplerate);

        const freq_fun_factor = @as(f32, @floatFromInt(frame_idx)) * 0.001;

        // in the left channel, let's make the frequency increasing...
        const signal1 = @sin(2 * std.math.pi * (220 + freq_fun_factor) * time);
        try stdout.writeInt(
            u16,
            scaleFloatToInt(samplewidth, signal1),

            Endian.little,
        );

        // and in the right channel, let's make it decreasing
        const signal2 = @sin(2 * std.math.pi * (221 - freq_fun_factor) * time);
        try stdout.writeInt(
            u16,
            scaleFloatToInt(samplewidth, signal2),
            Endian.little,
        );
    }

    try bw.flush();
}
```

In the glorious style of "everything in the main function"! :smile:

Hey, at least it has comments! =P

Anyway, as you can see, there is nothing dynamic going on here, all parameters
are defined at compile time, so the program always spit the same wave file
unless you change the code and recompile.

This makes the code very fast, but of course not super useful as-is. I'll leave
it for some next time to make this dynamic, maybe after learning how to handle
cmdline argument parsing. ;-)
