# GroovyMAME real-time (emusync/PART) information

## System requirements

* At least a 4th generation Intel i5 4-core or similar.
* 4 GB RAM.
* System running from SSD.
* A supported graphics card.

## TLDR

* Use `-nosleep`.
* For Windows, right click the MAME binary, click Compatibility and select
  "Disable fullscreen optimizations".
* To enable automatic frame delay, set `-framedelay 0` and `-autoframedelay`.
  Press F11 to get real-time frame delay information when a game is running.
* Make sure no other application aside from MAME is running (this includes
  frontends) and use `-sound part -audio_latency 0` (0=sets defaults) with
  on-board audio connected to wired speakers.

Example:

    mame [game] -nosleep -autoframedelay -framedelay 0 -sound part -audio_latency 0

Read the following sections if the above example doesn't work for you, or see
the bottom of the document for some bullet points you can try.

## Emusync information

Emusync estimates where the CRT is currently drawing (it does beam-racing).
Since the timestep of each frame is usually in the order of 16-20 ms, it is
important that MAME is allowed to run uninterrupted to not miss the next
deadline (which is VBLANK).

Emusync also enables automatic frame delay, making uninterrupted execution
even more important. The `-nosleep` parameter helps out with this and is
recommended to set. Keep in mind though that using `-nosleep` can increase
power consumption, heat dissipation and thus fan noise.

## PART information

The "PART" sound backend can provide close to PCB-level latencies, but is not
as forgiving as the other APIs.

To get the lowest possible latencies, *exclusive* access to the audio output
device is required. That means that *no other* application can use it at the
same time, which could cause problems with frontends and other running
applications. So if you're having problems with no audio output, shut down all
other apps and try launching MAME directly.

Compared to the native APIs, it only supports a single stereo output device, and
no mic input.

### API/Device selection

When initializing MAME, the info log will output something like this:

    PART: API ALSA has 13 devices
    PART: ALSA: "HDA NVidia: 27GL850 (hw:0,3)"
    PART: ALSA: "HDA NVidia: HDMI 1 (hw:0,7)"
    PART: ALSA: "HDA NVidia: HDMI 2 (hw:0,8)"
    PART: ALSA: "HDA NVidia: HDMI 3 (hw:0,9)"
    PART: ALSA: "HDA ATI HDMI: 0 (hw:1,3)"
    PART: ALSA: "HD-Audio Generic: ALCS1200A Analog (hw:2,0)"
    PART: ALSA: "HD-Audio Generic: ALCS1200A Digital (hw:2,1)"
    PART: ALSA: "HD-Audio Generic: ALCS1200A Alt Analog (hw:2,2)"
    PART: ALSA: "hdmi"
    PART: ALSA: "jack"
    PART: ALSA: "pipewire"
    PART: ALSA: "pulse"
    PART: ALSA: "default" (default)
    PART: API OSS has 0 devices

If you want to specify a device explicitly, you can do it using the `-part_api`
parameter in combination with the `-part_device` parameter, such as:

    -part_api "ALSA" -part_device "HD-Audio Generic: ALCS1200A Analog (hw:2,0)"

Later down in the log, when opening the sound device, something like this will
be output:

    PART: Opening device "ALSA: HD-Audio Generic: ALCS1200A Analog (hw:2,0)"
    PART: Sample rate is 48000 Hz, device output latency is 18.67 ms
    PART: Allowed additional buffering latency is 2.00 ms/96 frames

Which reports a latency figure. The resulting latency with the above example
(default `-audio_latency`) is reported to be around ~18.67 ms + 2.0 ms = ~20.67ms.

### Latency information

The lower the latency, the more likely it is that you'll experience audio
crackles.

The main source of latency is the resulting latency of the frequency at which
the operating system forwards audio to the audio device (given you're using
analog, wired, on-board audio). For PART, this is controlled with the `-audio_latency`
parameter (specified in milliseconds). This is almost always the source of
crackling audio and might require some trial and error to figure out what the
system is capable of. The Linux/Windows sections detail how to figure out this
setting.

### Manual configuration on Windows

In Windows, the info log with available devices can look like this:

    PART: API Windows WASAPI has 4 devices
    PART: Windows WASAPI: "Speakers (High Definition Audio Device)" (default)
    PART: Windows WASAPI: "Digital Audio (S/PDIF) (High Definition Audio Device)"
    PART: Windows WASAPI: "Speakers (High Definition Audio Device) [Loopback]"
    PART: Windows WASAPI: "Digital Audio (S/PDIF) (High Definition Audio Device) [Loopback]"
    PART: API Windows WDM-KS has 2 devices
    PART: Windows WDM-KS: "Speakers (HD Audio Speaker)" (default)
    PART: Windows WDM-KS: "SPDIF Out (HD Audio SPDIF out)"

`-part_api "Windows WASAPI"` or `-part_api "Windows WDM-KS"` are supported.

Start out with `-audio_latency 4`, increase to `8` and then `16` if audio crackles.

Example WASAPI invocation:

    mame -verbose -nosleep -sound part -part_api "Windows WASAPI" -audio_latency 4

Example WDM-KS invocation:

    mame -verbose -nosleep -sound part -part_api "Windows WDM-KS" -audio_latency 4

For WASAPI the verbose log tells us that we end up with ~6.0ms, and WDM-KS
gives us ~7.0 ms:

    PART: Opening device "Windows WASAPI: Speakers (High Definition Audio Device)"
    PART: Sample rate is 48000 Hz, device output latency is 4.00 ms
    PART: Allowed additional buffering latency is 2.00 ms/96 frames
    ...
    PART: Opening device "Windows WDM-KS: Speakers (HD Audio Speaker)"
    PART: Sample rate is 48000 Hz, device output latency is 5.00 ms
    PART: Allowed additional buffering latency is 2.00 ms/96 frames

Use the `-part_device` parameter to force a specific device if desired, the reported
latency numbers are put in the verbose log (like above).

### Manual configuration on Linux

On Linux, the raw ALSA device or PipeWire should be used. Most of the testing
has been done with the raw ALSA device directly. Only specifying the `-part_api`
parameter uses the default device of the selected API (usually pipewire).

Most often, pipewire ends up being the default. To use the raw ALSA device of
the sound chip, specify it explicitly using both `-part_api` and `-part_device`
like the example below. Latency figures using the raw ALSA device seem to be
reported accurately by the verbose log (based on actual measurements).

To figure out what latency the system supports, start out with a non-demanding
game and set `-audio_latency 4` along with the other desired settings. Increase
`-audio_latency` to either `8` or `16` if you get crackling audio.

Example ALSA invocation with raw ALSA device (bypassing pipewire/pulseaudio):

    mame -verbose -nosleep -sound part -part_api "ALSA" -part_device "ALSA: HD-Audio Generic: ALCS1200A Analog (hw:2,0)" -audio_latency 4

Example PipeWire invocation:

    mame -verbose -nosleep -sound part -part_api "ALSA" -part_device pipewire -audio_latency 4

For both examples, the verbose log reports a latency of ~5.5ms:

    PART: Opening device "ALSA: HD-Audio Generic: ALCS1200A Analog (hw:2,0)"
    PART: Sample rate is 48000 Hz, device output latency is 4.00 ms
    PART: Allowed additional buffering latency is 2.00 ms/96 frames
	...
    PART: Opening device "ALSA: pipewire"
    PART: Sample rate is 48000 Hz, device output latency is 4.00 ms
    PART: Allowed additional buffering latency is 2.00 ms/96 frames

## Bullet points

### MAME settings

* Use `-nosleep` for best performance, this might increase power consumption/heat
  but helps out with both emusync and PART.
* Use `-framedelay 0` and `-autoframedelay` for automatic frame delay.
* Set `-samplerate` to a sample rate natively supported by the audio interface
  (or just leave it at 48000, the default).
* The `-audio_latency` parameter for PART is specified in milliseconds. Check the
  verbose log if you are unsatisfied with the defaults. On Windows the defaults
  work well with on-board audio.

### System Tweaks

* Use on-board audio connected to analogue wired speakers.
* Try to have as few applications open as possible.
* For Windows, make sure “Allow applications to take exclusive control” is
  enabled for the audio interface (enabled by default).
* For Windows, right click the MAME binary, click Compatibility and select
  "Disable fullscreen optimizations".
* Power plan: Use High Performance / Ultimate Performance (might increase power
  consumption/heat/fan noise).
* Enable "Game mode".
