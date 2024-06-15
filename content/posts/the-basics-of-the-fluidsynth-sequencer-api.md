---
title: "The basics of the FluidSynth Sequencer API"
date: "2024-06-15T22:36:58+02:00"
tags: []
---
Today I learned the basics of the [FluidSynth Sequencer API](https://www.fluidsynth.org/api/Sequencer.html).
It is not super complicated, but it took me some time to understand the
concepts, as not everything is explained in great detail. As often is the
case when learning a new API, the best way to learn it is to read through the
code of the provided examples, and check the API reference docs as needed.

Here is a sequence diagram that I came up with to summarize the flow for a typical program:

[![](https://mermaid.ink/img/pako:eNp9ks1qwzAQhF9l0aUJpC9giqF_h0Ipoeml4MvG2tgisqRKq5Q05N27btz8lKQ-WauZb8ayNqr2mlShEn1kcjU9GGwidpUDeQJGNrUJ6BhuQwBM8O5zhGn0B1G_cV2WMBsIsYAnZ9igNV90mP4Rrx23p8J-cpH4So1JTBG4HaSw8MPqQsIZM8I9WjvHegkjXMrKGnI83hmt96GH0kqUws6OjYXEPgTSF9mzuiWdLcGLZ4LHlfASsIepxTU8I_9Ta299M51E_nh7651MomkaiqSPEXvr8QnOyOnj7Jt5LEefLTkwfCVNBD0-Z5c-xW_M_lQO5pNKO1UaOBKoJkq2OzRabs6mH1dKfkVHlSrkVdMCs-VKVW4rUszspW2tCo6ZJioHLR81XDRVLNAm2n4DrGbarw?type=png)](https://mermaid.live/edit#pako:eNp9ks1qwzAQhF9l0aUJpC9giqF_h0Ipoeml4MvG2tgisqRKq5Q05N27btz8lKQ-WauZb8ayNqr2mlShEn1kcjU9GGwidpUDeQJGNrUJ6BhuQwBM8O5zhGn0B1G_cV2WMBsIsYAnZ9igNV90mP4Rrx23p8J-cpH4So1JTBG4HaSw8MPqQsIZM8I9WjvHegkjXMrKGnI83hmt96GH0kqUws6OjYXEPgTSF9mzuiWdLcGLZ4LHlfASsIepxTU8I_9Ta299M51E_nh7651MomkaiqSPEXvr8QnOyOnj7Jt5LEefLTkwfCVNBD0-Z5c-xW_M_lQO5pNKO1UaOBKoJkq2OzRabs6mH1dKfkVHlSrkVdMCs-VKVW4rUszspW2tCo6ZJioHLR81XDRVLNAm2n4DrGbarw)

Now of course, just reading code can be a bit boring and tiresome, I really like to learn by
doing/tinkering, so I decided to translate the [metronome
example](https://www.fluidsynth.org/api/fluidsynth_metronome_8c-example.html)
to [Zig](https://ziglang.org) -- as I'm interested in learning Zig as well.

### FluidSynth Metronome - Zig Version

Here is what I came up with:

```zig
const std = @import("std");
const fluid = @cImport(@cInclude("fluidsynth.h"));

const tempo = 120;
const beat_duration_ms: u32 = 60000 / tempo;
const pattern_size = 4;

// global variable to keep track of the current sequencer time
var TIME_MARKER: c_uint = 0;
var SYNTH_DESTINATION: fluid.fluid_seq_id_t = 0;
var CLIENT_DESTINATION: fluid.fluid_seq_id_t = 0;

fn schedule_timer_event(seq: ?*fluid.fluid_sequencer_t, time_marker: c_uint) void {
    const event = fluid.new_fluid_event();
    defer fluid.delete_fluid_event(event);

    fluid.fluid_event_set_source(event, -1);
    fluid.fluid_event_set_dest(event, CLIENT_DESTINATION);
    fluid.fluid_event_timer(event, null);
    _ = fluid.fluid_sequencer_send_at(seq, event, time_marker, 1);
}

fn schedule_note_on(seq: ?*fluid.fluid_sequencer_t, midi_chan: i16, time: c_uint, note: i16, velocity: i16) void {
    const event = fluid.new_fluid_event();
    defer fluid.delete_fluid_event(event);

    fluid.fluid_event_set_source(event, -1);
    fluid.fluid_event_set_dest(event, SYNTH_DESTINATION);
    fluid.fluid_event_noteon(event, midi_chan, note, velocity);
    _ = fluid.fluid_sequencer_send_at(seq, event, time, 1);
}

fn schedule_metronome_pattern(seq: ?*fluid.fluid_sequencer_t) void {
    const midi_chan = 9;

    const strong_note = 76; // woodblock high
    const weak_note = 77; // woodblock low

    var note_time: c_uint = TIME_MARKER;
    var i: i16 = 0;
    while (i < pattern_size) : (i += 1) {
        var note_to_play: i16 = weak_note;
        var velocity: i16 = 90;
        if (i == 0) {
            note_to_play = strong_note;
            velocity = 127;
        }
        schedule_note_on(seq, midi_chan, note_time, note_to_play, velocity);
        note_time += beat_duration_ms;
    }
    TIME_MARKER += beat_duration_ms * pattern_size;
}

// this callback will be called every time a sequencer event is received
fn sequencer_callback(time: c_uint, event: ?*fluid.fluid_event_t, seq: ?*fluid.fluid_sequencer_t, data: ?*anyopaque) callconv(.C) void {
    _ = time;
    _ = data;
    _ = event;

    schedule_timer_event(seq, TIME_MARKER);
    schedule_metronome_pattern(seq);
}

pub fn main() void {
    const settings = fluid.new_fluid_settings();
    defer fluid.delete_fluid_settings(settings);
    const synth = fluid.new_fluid_synth(settings);
    defer fluid.delete_fluid_synth(synth);

    // here we load the soundfont file
    // load_soundfont(synth, "soundfonts/GeneralUser_GS_v1.471.sf2");
    _ = fluid.fluid_synth_sfload(synth, "soundfonts/GeneralUser_GS_v1.471.sf2", 1);

    // set up the audio driver to play sounds from the synth
    const audio_driver = fluid.new_fluid_audio_driver(settings, synth);
    defer fluid.delete_fluid_audio_driver(audio_driver);

    // initialize the sequencer
    const seq = fluid.new_fluid_sequencer2(0);
    defer fluid.delete_fluid_sequencer(seq);

    // this will be destination port for the synth where we'll send note events
    SYNTH_DESTINATION = fluid.fluid_sequencer_register_fluidsynth(seq, synth);

    CLIENT_DESTINATION = fluid.fluid_sequencer_register_client(seq, "zig-fluid-metronome", sequencer_callback, null);

    // get the current sequencer time
    TIME_MARKER = fluid.fluid_sequencer_get_tick(seq);

    schedule_metronome_pattern(seq);
    schedule_timer_event(seq, TIME_MARKER);
    schedule_metronome_pattern(seq); // schedule the next one, to be in advance

    // run the sequencer for 1 minute
    std.time.sleep(std.time.ns_per_min * 1);
    std.debug.print("Practice time is over, stopping now\n", .{});
}
```

About 60% the size of the C program in terms of lines of code, that's not bad!

I've hardcoded the soundfont to the GM GeneralUser GS that I already had a copy laying around.

I was quite happy with it, I struggled with compiler errors for some time but
once I got it to compile, it worked on first try!

In terms of code quality it's not that great though, I am not comfortable with
creating abstractions in Zig -- gotta learn that in order to build bigger
programs. The fact that I had to rely on global variables here bothers me, I
want to learn a better way of doing this.

### FluidSynth Metronome - Python Version

By the end, I was curious how quickly I could come up with a Python
implementation, using the
[pyfluidsynth](https://github.com/nwhitehead/pyfluidsynth) Python bindings for
the FluidSynth C API, which I already knew from using it when writing
[upiano](https://github.com/eliasdorneles/upiano).

It was relatively straightforward, except a few gotchas:

* for some reason, with `pyfluidsynth` I _have_ to call `program_select` to
  select the "Standard Drum" program on the appropriate MIDI channel, otherwise
  it won't play any sound if you send events to channel 10 (the drum/percussion
  channel), something that I didn't have to do when calling directly the C API
  from Zig.
* I didn't know which bank/preset number the "Standard Drum" was in, so I had
  to grab a tool to inspect the GM GeneralUser soundfont. I used the
  [tynisoundfont](https://github.com/nwhitehead/tinysoundfont-pybind) tool,
  which you can use like `tinysoundfont --info $SOUNDFONT_FILE`, and it will
  describe the contents -- FYI, it was the first preset of bank `120`.
* the Sequencer class provided by pyFluidSynth that wraps the Sequencer API is
  rather neat, though I had some runtime errors when inadverentely sending
  `float` instead of `int` for the time argument.

Here is the code:

```Python
import time
import fluidsynth


class Metronome:
    def __init__(self, tempo=120, pattern_size=4):
        self.tempo = tempo
        self.beat_duration_ms = int(60000 / tempo)
        self.pattern_size = pattern_size

        self._synth = fluidsynth.Synth()
        self._seq = fluidsynth.Sequencer(use_system_timer=False)
        self.current_time = self._seq.get_tick()

        # load my old friend General User GS SoundFont ...
        soundfont_id = self._synth.sfload("soundfonts/GeneralUser_GS_v1.471.sf2")
        # ... and select "Standard Drum" program for percussion channel
        self._synth.program_select(9, soundfont_id, 120, 0)

        # register the synth which will play the note events...
        self._synth_id = self._seq.register_fluidsynth(self._synth)
        # ... and the client callback which will be called each time the timer ticks
        self._callback_id = self._seq.register_client(
            "py-fluid-metronome", self.seq_callback
        )

    def start(self):
        self._synth.start()

        # schedule the pattern twice so as to always be 1 measure ahead
        self.schedule_metronome_pattern()
        self.schedule_timer_event()
        self.schedule_metronome_pattern()
        print("Metronome started, hit Ctrl+C to stop")

        try:
            # sleep for 24 hours (or until interrupted)
            time.sleep(60 * 60 * 24)
        except KeyboardInterrupt:
            pass
        finally:
            self._synth.delete()

    def schedule_timer_event(self):
        self._seq.timer(self.current_time, dest=self._callback_id)

    def seq_callback(self, _time, _tick, _event, _data):
        # This is called by the sequencer each time a timer event is triggered

        # First thing we do is to schedule the next timer event, in order
        # to keep the sequencer loop going ...
        self.schedule_timer_event()

        # ... and then we schedule the notes to be played
        self.schedule_metronome_pattern()

    def schedule_metronome_pattern(self):
        strong_note = 76  # wood block high
        weak_note = 77  # wood block low

        midi_channel = 9  # percussion channel

        for i in range(self.pattern_size):
            note, vel = weak_note, 80
            if i == 0:  # first beat
                note, vel = strong_note, 120

            self._seq.note_on(
                time=self.current_time + self.beat_duration_ms * i,
                channel=midi_channel,
                key=note,
                velocity=vel,
                dest=self._synth_id,
            )

        self.current_time += self.beat_duration_ms * self.pattern_size


if __name__ == "__main__":
    Metronome().start()
```


I'm quite happy with this code, because I could refactor the initial mess I had
came up with into a nice `Metronome` class encapsulating the behavior. I was
much more productive in Python obviously, besides having already got a good
graps of the API, Python is the language I've been using on a daily basis over
the past 10 years, so there was a lot less new things to learn.
