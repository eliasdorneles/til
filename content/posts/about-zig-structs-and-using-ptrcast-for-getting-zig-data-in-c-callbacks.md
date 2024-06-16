---
title: "About Zig structs and using @ptrCast for getting Zig data in C callbacks"
date: "2024-06-16T15:01:52+02:00"
tags:
  - zig
  - fluidsynth
  - C-library
  - midi
---

In [the previous
post](https://eliasdorneles.com/til/posts/the-basics-of-the-fluidsynth-sequencer-api/),
I went about my experiments with [Zig](https://ziglang.org/) and the [FluidSynth
Sequencer API](https://www.fluidsynth.org/api/Sequencer.html).

I wasn't fully happy with my Zig code, because I was resorting to global
variables and didn't quite know how to organize it.

So I went on and learned how to make abstractions in Zig, and it is actually
quite simple: just use a
[struct](https://ziglang.org/documentation/0.13.0/#struct)! Much like a C
typedef struct, that allows for state and behavior encapsulation -- almost like
a class, except no inheritance. But inheritance kinda stinks anyway, so...
we're good! :D

I started to refactor my code into a struct I called `Metronome` much like in
the Python version from the previous post, adding functions that receive the
reference to the struct as first argument, "cool, this is a lot like Python
actually".

Then I hit a problem: how will I get a pointer to `self` inside my callback
function that needs to be called from C, which I cannot change its signature?

At first, I left one global variable that pointed to the struct, which I used
in the callback. Then, after asking [for help on the
internet](https://ziggit.dev/t/how-to-wrap-a-zig-function-to-be-called-from-c/4747),
someone gave me the solution: use the `data` parameter of the callback function
to pass along the reference, do some pointer casting _et voilà !_

That allowed me to put the callback inside the `Metronome` and get rid of the
final global variable!

The solution looks obvious now, but I was too deep in my mess to see it, plus I
didn't know about pointer casting.

So yeah, that's how I learned about using structs in Zig, and
[`@ptrCast`](https://ziglang.org/documentation/0.13.0/#ptrCast) to convert
pointers from different types, and also to combine it with
[`@alignCast`](https://ziglang.org/documentation/0.13.0/#alignCast) to change
the pointer alignment.

I am now quite pleased with the code, here it is:

```Zig
// this is a Zig port of the C example at:
// https://www.fluidsynth.org/api/fluidsynth_metronome_8c-example.html
const std = @import("std");
const fluid = @cImport(@cInclude("fluidsynth.h"));

const Metronome = struct {
    // metronome parameters
    tempo: u32 = 120,
    pattern_size: u32 = 4,

    // sequencer instance
    seq: ?*fluid.fluid_sequencer_t = null,

    // will mark the begin time of a measure, updated every time a new measure is scheduled
    time_marker: c_uint = 0,

    // sequencer destination ports
    synth_port: fluid.fluid_seq_id_t = 0,
    client_callback_port: fluid.fluid_seq_id_t = 0,

    pub fn init(
        self: *Metronome,
        seq: ?*fluid.fluid_sequencer_t,
        synth: ?*fluid.fluid_synth_t,
    ) void {
        self.seq = seq;

        // we'll send note events to the synth port
        self.synth_port = fluid.fluid_sequencer_register_fluidsynth(seq, synth);

        self.client_callback_port = fluid.fluid_sequencer_register_client(
            seq,
            "zig-fluid-metronome",
            sequencer_callback,
            @ptrCast(self),
        );

        // get the current sequencer time
        self.time_marker = fluid.fluid_sequencer_get_tick(seq);
    }

    // this callback will be called every time a sequencer event is received
    fn sequencer_callback(
        time: c_uint,
        event: ?*fluid.fluid_event_t,
        seq: ?*fluid.fluid_sequencer_t,
        data: ?*anyopaque,
    ) callconv(.C) void {
        _ = time;
        _ = event;
        _ = seq;
        const self: *Metronome = @ptrCast(@alignCast(data.?));
        self.schedule_timer_event();
        self.schedule_metronome_pattern();
    }

    fn schedule_timer_event(self: *Metronome) void {
        const event = fluid.new_fluid_event();
        defer fluid.delete_fluid_event(event);

        fluid.fluid_event_set_source(event, -1);
        fluid.fluid_event_set_dest(event, self.client_callback_port);
        fluid.fluid_event_timer(event, null);
        _ = fluid.fluid_sequencer_send_at(self.seq, event, self.time_marker, 1);
    }

    fn schedule_note_on(
        self: *Metronome,
        midi_chan: i16,
        time: c_uint,
        note: i16,
        velocity: i16,
    ) void {
        const event = fluid.new_fluid_event();
        defer fluid.delete_fluid_event(event);

        fluid.fluid_event_set_source(event, -1);
        fluid.fluid_event_set_dest(event, self.synth_port);
        fluid.fluid_event_noteon(event, midi_chan, note, velocity);
        _ = fluid.fluid_sequencer_send_at(self.seq, event, time, 1);
    }

    fn schedule_metronome_pattern(self: *Metronome) void {
        const beat_duration_ms: u32 = 60000 / self.tempo;

        const midi_chan = 9; // channel 10 is the drum channel

        const strong_note = 76; // woodblock high
        const weak_note = 77; // woodblock low

        var i: u32 = 0;
        while (i < self.pattern_size) : (i += 1) {
            const note_to_play: i16 = if (i == 0) strong_note else weak_note;
            const velocity: i16 = if (i == 0) 127 else 90;
            const play_at: c_uint = self.time_marker + (i * beat_duration_ms);
            self.schedule_note_on(midi_chan, play_at, note_to_play, velocity);
        }
        self.time_marker += beat_duration_ms * self.pattern_size;
    }

    pub fn start(self: *Metronome) void {
        self.schedule_metronome_pattern();
        self.schedule_timer_event();
        self.schedule_metronome_pattern();
    }
};

pub fn main() void {
    const settings = fluid.new_fluid_settings();
    defer fluid.delete_fluid_settings(settings);
    const synth = fluid.new_fluid_synth(settings);
    defer fluid.delete_fluid_synth(synth);

    // here we load the soundfont file
    _ = fluid.fluid_synth_sfload(synth, "soundfonts/GeneralUser_GS_v1.471.sf2", 1);

    // set up the audio driver to play sounds from the synth
    const audio_driver = fluid.new_fluid_audio_driver(settings, synth);
    defer fluid.delete_fluid_audio_driver(audio_driver);

    // initialize the sequencer
    const seq = fluid.new_fluid_sequencer2(0);
    defer fluid.delete_fluid_sequencer(seq);

    var metro = Metronome{ .tempo = 120, .pattern_size = 4 };
    metro.init(seq, synth);
    metro.start();

    // run the sequencer for 1 minute
    std.time.sleep(std.time.ns_per_min * 1);
    std.debug.print("Practice time is over, stopping now\n", .{});
}
```
