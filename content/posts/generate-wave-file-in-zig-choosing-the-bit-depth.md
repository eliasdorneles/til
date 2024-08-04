---
title: "Generate WAVE file in Zig, choosing the bit depth"
date: "2024-08-04T20:01:59+02:00"
tags:
  - Zig
  - audio processing
  - wave
---

In the last post, I told you how I learned the basics of generating a WAVE
file, both in Python and Zig.

It turns out, my Zig code had a few bugs:

- it was using the incorrect type for the 16-bits bit depth, which made the
  audio a bit distorted
- the audio length was incorrect because the size of the data chunk was
  incorrectly calculated

Another thing was bugging me, I wanted to make it generic regarding the [bit
depth](https://en.wikipedia.org/wiki/Audio_bit_depth), so that I could decide
if I wanted to use 8-bits or 16-bits each time.

In the process, I learned how to use types as values at compilation time. I
thought there were would be some magic needed, but it turns out there's no
magic required: it's the same syntax and semantics, the only difference is that
they need to be known at compile time. Namely, you can have variables with type
`type`, and assign values to them like `bool`, `u8` or `i16` to them, and use
these values in expressions as well, e.g. `if (someType == u8) ...`. As long as
the value of `someType` is known at compile time, it's all good!

This allows for interesting ways to create abstractions.

I peeked at the code of a project found online,
[zig-wav](https://github.com/veloscillator/zig-wav), and it has some really
cool tricks using this to convert samples from any known type to a target type.
[See the code in sample.zig
module](https://github.com/veloscillator/zig-wav/blob/main/src/sample.zig), I
thought the implementation of that `convert()` function real smart. I changed
the way I convert the samples from float to int to a similar way that's done
there.

Finally, I also added a tiny usability improvement: I added a `volume` scaler,
so that the result it's not too loud. =D

Here's the final code:

```zig
const std = @import("std");
const Endian = std.builtin.Endian;

// PCM Format:
// 8-bit samples are stored as unsigned bytes, ranging from 0 to 255.
// 16-bit samples are stored as 2's-complement signed integers, ranging from -32768 to 32767.

fn convertSampleToInt(comptime T: type, value: f32) T {
    const min = std.math.minInt(T);
    const max = std.math.maxInt(T);
    var toScale = value;

    // here we handle the special case of 8-bit samples
    if (T == u8) toScale = (toScale + 1) / 2;

    return @intFromFloat(std.math.clamp(@round(toScale * max), min, max));
}

// Write a WAV file with a simple stereo signal to the stdout.
// The left channel has a frequency that increases over time, while the right
// channel has a frequency that decreases over time.
pub fn main() !void {
    const stdout_file = std.io.getStdOut().writer();
    var bw = std.io.bufferedWriter(stdout_file);
    const stdout = bw.writer();

    const duration = 5; // seconds
    const bit_depth = 16; // try changing this to 8
    const samplerate = 44100;
    const nchannels = 2;

    // we will use this type to write the samples to the file
    const sampleType = switch (bit_depth) {
        8 => u8,
        16 => i16,
        else => @panic("unsupported bit depth, only 8 and 16 are supported."),
    };

    const num_of_samples: u32 = duration * samplerate;
    // This article explains the WAV format in detail:
    // http://soundfile.sapp.org/doc/WaveFormat/

    // We start with the RIFF header
    try stdout.writeAll("RIFF");

    // The size of the file, minus the first 8 bytes
    const header_size = 36;
    try stdout.writeInt(
        u32,
        header_size + num_of_samples * nchannels * (bit_depth / 8),
        Endian.little,
    );
    try stdout.writeAll("WAVE");

    // format chunk
    try stdout.writeAll("fmt ");
    try stdout.writeInt(u32, 16, Endian.little);

    try stdout.writeInt(u16, 1, Endian.little); // audio format (1 = PCM)
    try stdout.writeInt(u16, nchannels, Endian.little);

    try stdout.writeInt(u32, samplerate, Endian.little);

    // block align, i.e. number of bytes per sample (all channels included)
    const blockAlign = nchannels * (bit_depth / 8);

    // byte rate
    try stdout.writeInt(u32, blockAlign * samplerate, Endian.little);

    try stdout.writeInt(u16, blockAlign, Endian.little);

    // bit depth
    try stdout.writeInt(u16, bit_depth, Endian.little);

    // now, setup the data chunk...
    try stdout.writeAll("data");
    try stdout.writeInt(
        u32,
        num_of_samples * nchannels * (bit_depth / 8),
        Endian.little,
    );

    // and let's write some audio data!
    const volume = 0.5;
    for (0..num_of_samples) |frame_idx| {
        const time = @as(f32, @floatFromInt(frame_idx)) / @as(f32, samplerate);

        const freq_fun_factor = @as(f32, @floatFromInt(frame_idx)) * 0.002;

        // in the left channel, let's make the frequency increasing...
        const signal1 = volume * @sin(2 * std.math.pi * (220 + freq_fun_factor) * time);
        try stdout.writeInt(
            sampleType,
            convertSampleToInt(sampleType, signal1),

            Endian.little,
        );

        if (nchannels == 1) {
            continue;
        }

        // and in the right channel, let's make it decreasing
        const signal2 = volume * @sin(2 * std.math.pi * (220 - freq_fun_factor) * time);
        try stdout.writeInt(
            sampleType,
            convertSampleToInt(sampleType, signal2),
            Endian.little,
        );
    }

    try bw.flush();
}
```
