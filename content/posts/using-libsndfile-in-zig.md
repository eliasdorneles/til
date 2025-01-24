---
title: "Using libsndfile in Zig"
date: "2025-01-24T01:18:50+01:00"
tags:
- Zig
- audio
- libsndfile
---
Continuing on my journey learning Zig and audio programming, and also making a
good follow-up to [the time I wrote Zig to manually generate a WAVE
file](https://eliasdorneles.com/til/posts/generate-wave-file-in-zig-choosing-the-bit-depth/),
today I decided to generate a WAVE file using [libsndfile](https://libsndfile.github.io/libsndfile/).

If you don't know what libsndfile is, it's a library for reading and writing
audio files. It supports many formats and is quite popular. Learning its basics
is a good preparation step for me to write more ambitious programs.

Let's just go straight to the code -- I've added comments to the relevant bits:

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("sndfile.h");
});
const print = std.debug.print;
const math = std.math;

pub fn main() !u8 {
    // Global params
    const volume = 0.3;
    const samplerate = 44100;
    const nchannels = 2;
    const duration = 3; // seconds
    const outfile = "out_sndfile_test.wav";

    // Here we build a sine table that we'll use to generate the audio samples later
    const TABLE_SIZE = 200;
    var sine_table: [TABLE_SIZE]f32 = undefined;
    const table_size_float: f32 = @floatFromInt(TABLE_SIZE);
    for (&sine_table, 0..) |*item, i| {
        item.* = math.sin((@as(f32, @floatFromInt(i)) / table_size_float) * 2.0 * math.pi);
    }

    // Here we build the SF_INFO struct, which configures the output WAV file
    var sf_info: c.SF_INFO = undefined;
    sf_info.samplerate = samplerate;
    sf_info.channels = nchannels;
    sf_info.format = c.SF_FORMAT_WAV | c.SF_FORMAT_PCM_32;

    const sf: ?*c.SNDFILE = c.sf_open(outfile, c.SFM_WRITE, &sf_info);
    if (sf == null) {
        var errstr: [256]u8 = undefined;
        _ = c.sf_error_str(sf, &errstr, errstr.len - 1);
        print("Cannot open sndfile {s} for writing: {s}\n", .{ "test.wav", errstr });
        return 1;
    }

    const nframes = samplerate * duration;

    const buffer_size = 1024;
    var buffer: [buffer_size]f32 = undefined;
    var left_phase: usize = 0;
    var right_phase: usize = 0;

    // Now, we'll proceed by generating the audio samples and writing them to the WAV file
    const needed_iterations = nframes / (buffer_size / nchannels);
    for (0..needed_iterations) |_| {
        // First, we fill the buffer with interleaved left and right channel samples...
        var buf_index: usize = 0;
        while (buf_index < buffer_size - 1) : (buf_index += 2) {
            left_phase += 1;
            if (left_phase >= TABLE_SIZE) {
                left_phase -= TABLE_SIZE;
            }
            // we use a different phase for the right channel, to have a different note:
            right_phase += 2;
            if (right_phase >= TABLE_SIZE) {
                right_phase -= TABLE_SIZE;
            }
            buffer[buf_index] = volume * sine_table[left_phase];
            buffer[buf_index + 1] = volume * sine_table[right_phase];
        }

        // Then, we write the buffer to the output file -- note that we divide
        // buffer_size by nchannels to get the correct number of frames
        const frames_to_write = buffer_size / nchannels;
        _ = c.sf_writef_float(sf, &buffer, frames_to_write);
    }
    _ = c.sf_close(sf);
    print("Wrote {d} frames to {s}\n", .{ nframes, outfile });

    return 0;
}
```
