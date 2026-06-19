<h1> MoMIDI: Morse over MIDI </h1>

Specification for sending Morse Code over MIDI events.

This document builds on existing MIDI protocols used by several hardware and software manufacturers for triggering keying events between hardware keys, and software keyers, transmitters, practice oscillators, etc.  The goal is to use MIDI to trigger key-down and key-up events between morse keys, and software running on a computer (eg: an SDR transmitter, remote station, practice keyer, etc.)

This document represents [MoMIDI version](#12-momidi-versions) **v0.1 PROPOSED**

A Quick Reference for the MIDI Protocol: <https://www.songstuff.com/recording/article/midi-message-format/>

<div class="unstyledtemplate template" style="display: block;">
	<div id="groupsio_unstyled_embed_signup">
		<form action="https://groups.io/g/momidi/signup?u=5426540281110385330" method="post" id="groupsio-embedded-subscribe-form" name="groupsio-embedded-subscribe-form" target="_blank">
			<div id="groupsio_unstyled_embed_signup_scroll">
				<label for="email" id="unstyletemplateformtitle">Subscribe to the MoMIDI groups.io email list:</label>
				<br>
				<input type="email" value="" name="email" class="email" id="email" placeholder="Email Address" required="">
				<!-- real people should not fill this in and expect good things - do not remove this or risk form bot signups-->
				<div style="position: absolute; left: -5000px;" aria-hidden="true">
					<input type="text" name="b_5426540281110385330" tabindex="-1" value="">
				</div>
				<div id="templatearchives"><p><a id="archivelink" href="https://groups.io/g/momidi/topics">View Archives</a></p></div>
				<input type="submit" value="Subscribe" name="subscribe" id="groupsio-embedded-subscribe" class="button">
			</div>
		</form>
	</div>
</div>

<h2>Table of Contents</h2>

- [1. MoMIDI Specification](#1-momidi-specification)
	- [1.1. Terms](#11-terms)
	- [1.2. MoMIDI Versions](#12-momidi-versions)
		- [1.2.1. List of Versions](#121-list-of-versions)
- [2. MoMIDI Protocol](#2-momidi-protocol)
	- [2.1. MoMIDI Without Timing](#21-momidi-without-timing)
	- [2.2. Timing Information](#22-timing-information)
		- [2.2.1. How Time Is Measured Between Events](#221-how-time-is-measured-between-events)
	- [2.3. Representing Time in MIDI](#23-representing-time-in-midi)
		- [2.3.1. No Time Available, or Timer Reset](#231-no-time-available-or-timer-reset)
		- [2.3.2. Time Encoding: Base126](#232-time-encoding-base126)
		- [2.3.3. The largest number that can be represented](#233-the-largest-number-that-can-be-represented)
	- [2.4. Concerns](#24-concerns)
- [3. System Exclusive (SysEx) Manufacturer IDs](#3-system-exclusive-sysex-manufacturer-ids)
	- [3.1. MoMIDI Manufacturer ID table](#31-momidi-manufacturer-id-table)
- [4. Contributers:](#4-contributers)

<h2>Document History </h2>

| Date | Version | Author | Changes |
| ---- | ------- | ------ | ------- |
| 2025-11-24 | n/a | @SmittyHalibut | First draft. |
| 2025-12-03 | n/a | @SmittyHalibut | Updated timing from 1ms to 2ms resolution. Someone should check my math. |
| 2026-01-19 | n/a | @SmittyHalibut | Going back to 1ms resolution. Added pseudocode and tables to Rules. |
| 2026-02-20 | v0.0 | @SmittyHalibut | Changes proposed by @schrockwell: Base126.<br/>Added version numbering. |
| 2026-02-25 | v0.0 | @SmittyHalibut | Fixing "off-by-one" confusion on CC=0 Velocity. Not bumping version. |
| 2026-03-20 | v0.0 | @SmittyHalibut | Fixed order of operations in `velocity` calculation. |
| 2026-06-18 | v0.1 | @SmittyHalibut | Issue #1:<br/>Timing MSB to Polyphonic Aftertouch, sent after Note On/Off.<br/>MoMIDI Version to Song Select.<br/>Start tracking Manufacturer IDs for SysEx messages.  |

# 1. MoMIDI Specification

## 1.1. Terms

* **Key**: In the context of this document, a "key" can be any hardware device meant for sending morse code.
* **Straight Key**: A key who's timing of dits and dahs is done before the conversion to MIDI.  Either a literal straight key, a bug, or even a hardware iambic keyer, etc.  The important part here is that the straight key has a single electrical contact that controls the transmitter directly.
  * Note: This could also include a hardware iambic keyer.  The output of the hardware keyer is a single contact closure, and all the timing is generated on the user side of the MIDI interface.
* **Iambic Key**, or **Paddles**: A key who's timing of dits and dahs is done in software, on the receiving side of the MIDI events.
* **MIDI event**: MIDI, or the Musical Instrument Digital Interface, is a standard for sending very precise timing events over a serial link.  It's designed around musical instruments, but we are coopting it for another time and latency sensitive use: sending morse code.
  * MIDI Note On event: Includes two 7-bit values: Note, and Velocity
  * MIDI Note Off event: Includes two 7-bit values: Note, and Velocity
  * MIDI Control Change event: Includes two 7-bit values: Channel, and Value
* **Timing Round**: The state machine that tracks timing information.  It starts with a time value of zero, and ends after there's been more than [`MAX_COUNT`](#233-the-largest-number-that-can-be-represented) milliseconds since the last tracked event, after which the next event sent will be sent with a value of zero and start a new Timing Round.
* **Sender**: the device sending the MoMIDI packets.
* **Receiver**: the software on the computer receiving and interpreting the MoMIDI.

## 1.2. MoMIDI Versions

The MoMIDI version indicates what features the receiver can expect from the sender.  It is sent as the Value in a Song Select (MIDI Status 0xF3) event, so it's limited to 7 bits.

Those 7 bits are split into Major (3 MSB) and Minor (4 LSB) version numbers:

* Bits 6..4: Major version.  Incremented on breaking changes.
* Bits 3..0: Minor version.  Incremented on major, but backward compatible, added features.

The version can be written for human consumption using decimal as: `v[major 0..7].[minor 0..15]`  For example: `v2.10`.

When writing the version in a technical context, it may also be written as a hexadecimal number: `0x[major 0..7][minor 0..F]`  For example: `0x2A` is equivalent to `v2.10`.

### 1.2.1. List of Versions

| Version | Release Date | Description/More Information |
| ------- | ------------ | ---------------------------- |
| no version | 2025-11-24 | Initial development. Changed a whole lot. |
| 0x00 | 2026-02-20 | First indicated version, still under development. |
| 0x01 **PROPOSED** | 2026-06-18 | Timing MSB to Polyphonic Aftertouch, sent after Note On/Off.<br/>MoMIDI Version to Song Select.<br/>Start tracking Manufacturer IDs for SysEx messages. |
| 0x10 | TBD | Initial release. |

# 2. MoMIDI Protocol

Convention: bare numbers are decimal. Hexidecimal are prefixed with `0x`.  eg: `20` is decimal, `0x14` is its hexidecimal equivalent.

MIDI Note On/Off events send a 7-bit Note value, and a 7-bit value called Velocity. It's been pretty well established to use these MIDI Notes to trigger CW events:

* Note On events (MIDI Status 0x8n): key down
* Note Off events (MIDI Status 0x9n): key up
* Note 20: left paddle, or straight key
* Note 21: right paddle
* Note 30: straight key
* Note 31: PTT
* (There are a lot of other MIDI events used for various other radio control features, but we are concentrating on CW here.)

## 2.1. MoMIDI Without Timing

Sending timing information is optional.  If no timing information is being tracked, send Notes with a Velocity of either 127, or 0.  Usually, 127 is sent with Note On events, and 0 with Note Off events.  Both of these values mean "no timing information available" and the software receiving those events measures the time of events by when the MIDI event was received.  Do not send or expect any additional MIDI events.

## 2.2. Timing Information

To send timing information, measure in firmware the time between key down/up events to millisecond precision.  Then send that information to the software in the Velocity field of the Note On/Off event.

7 bits in the Note On/Off Velocity field is only good to measure up to 126ms (remember, 127 means "no timing information") which isn't enough in many cases.  So we also send a Polyphonic Aftertouch event, immediately after the Note On/Off, to the same note, with 7 additional (most significant) bits sent in the Pressure value.  14 bits is enough to count over 16 seconds.  Beyond that, precise timing isn't critical.

Calculating the values sent in the Note On/Off Velocity and Aftertouch Pressure are defined below.

Sending a Polyphonic Aftertouch event is not optional (as the Control Code was in previous MoMIDI versions).  When the time is \<= 126ms, send the Polyphonic Aftertouch event with a Pressure value of 0.

If possible, send the Note On/Off and the Aftertouch events in the same packet, to minimize latency on the receiver side.

> **Side note**: Above 30wpm (40ms per morse symbol), the MIDI latency between key-down and sound-on in a software based keyer (measured using current hardware and software as: 10ms at best, typically 15ms to 25ms) becomes significant.  We assume fast CW operators (above 30wpm) will use hardware based keyers with hardware side-tone.

### 2.2.1. How Time Is Measured Between Events

Timing is tracked between all CW MIDI events, even if they were a different event.  For example, when pressing Left, then pressing Right, then releasing Left, and releasing Right, The left-down is sent with 0 signifying the start of timing, the right-down is sent with the time since the left-down, the left-up is sent with the time since the right-down, and the right-up is sent with the time since the left-up.

| Wall clock time | Event | MIDI Sent | Time sent with event |
| --------------- | ----- | --------- | -------------------- |
| 01:23:45.678 | Left Down | Note On 20 | 0, start of timing. |
| 01:23:45.778 | Right Down | Note On 21 | 100ms |
| 01:23:45.808 | Left Up | Note Off 20 | 30ms |
| 01:23:45.958 | Right Up | Note Off 21 | 150ms |

## 2.3. Representing Time in MIDI

Here are the rules used to encode time since last event 1 .. 16128ms, into MIDI events and values.

### 2.3.1. No Time Available, or Timer Reset

When it's been more than [`MAX_COUNT`](#233-the-largest-number-that-can-be-represented) milliseconds since the last event, the timer is reset to zero and not started again until another event is sent.  

Three MIDI events are sent, one additional before the normal two:

1. MoMIDI Version Indication: Song Select (MIDI Status 0xF3), with the value set to the MoMIDI version.
2. Event Note On (MIDI Status 0x80) or Off (MIDI Status 0x90) to the event's Note, with Velocity set to 0.
3. Polyphonic Aftertouch (MIDI Status 0xA0) to the event's Note, with Pressure set to 0.

### 2.3.2. Time Encoding: Base126

The number of milliseconds since the last event (here called "Time") is sent in two parts, 7 bits each: the least significant bits in the Velocity of a Note On/Off event, followed by the most significant bits in the Pressure of the Polyphonic Aftertouch event.

Remember, a Note On/Off Velocity of 0 and 127 mean "No timing information available" so we can't use those values.  That leaves 126 Velocity values, and all 128 Aftertouch Pressures.

Since we can't use Time=0, and to minimize off-by-one confusion while debugging, we consider `Time - 1` when doing the following math.  This makes the Note On/Off Velocity equal to the actual time, when the Aftertouch Pressure is calculated to be 0.

The following pseudo-code works for values `1 <= Time <= MAX_VALUE`.  Notably, they do NOT work for `Time == 0`.  So make sure you handle a zero time explicitly.

* `Note On/Off Velocity = (Time-1)%126 + 1`
* `Polyphonic Aftertouch Pressure = floor((Time-1)/126)`

On the receiving side, you calculate the time as:

* `Time = Velocity + Pressure*126`

[A spreadsheet to validate the math can be found here](https://docs.google.com/spreadsheets/d/e/2PACX-1vQ38EwbZWGbNSEr9GhxVg4X3fGhydFB1SyfVwB9YaKn2DKvBieZ9MsJkJnr08AiFBOpsHDTO9GHzv3w/pubhtml?gid=1862224261&single=true).

Two MIDI events are sent:

1. Event Note On (MIDI Status 0x80) of Orr (MIDI Status 0x90) to the event's Note, with the Velocity set as above.
2. Polyphonic Aftertouch (MIDI Status 0xA0) to the event's Note, with Pressure set as above.

### 2.3.3. The largest number that can be represented

The maximum value that can be represented, `MAX_COUNT = 128*126 = 16128`.  This represents the number of idle milliseconds when the timing gets reset.

## 2.4. Concerns

If you have any concerns with this system, put them here.

* 2025-12-03, @SmittyHalibut: Should this document be kept to morse events only: left/right paddle, and straight key?  This is where millisecond level timing is most important. But MIDI is used for SO MUCH MORE, it would be good to get that documented too.  'course, if we do that, then "Morse over MIDI" is an increasingly inappropriate name.  I'm open to thoughts on this.

# 3. System Exclusive (SysEx) Manufacturer IDs

MIDI has the concept of a System Exclusive, or SysEx, message.  It represents data that is unique to a particular manufacturer's equipment, and is not meant to be "standardized" across multiple manufacturers.  It is often used for things such as firmware updates, audio patch downloads, etc.  Can be large data, is unique to a specific device or manufacturer.

As such, SysEx messages include a globally recognized "Manufacturer ID" that identifies what type of hardware a given device is.  MIDI maintains their own list of manufacturers, which are out-of-scope for MoMIDI.  To let MoMIDI manufacturers use SysEx messages to communicate with their hardware, MoMIDI tracks its own list of manufacturers.

MoMIDI manufacturers are responsible for specifying their own protocol for data inside the SysEx stream, since it is by its nature unique to your their product.  But it is recommended that the SysEx header begin with a Product ID so that a single Manufacturer ID can be used with multiple products.  The scope of that Product ID is limited to a given Manufacturer ID.

## 3.1. MoMIDI Manufacturer ID table

At some future date, this make be split out to its own document.  For now, we will embed it in the MoMIDI Spec.

| Mfg ID | Manufacturer / Purpose |
| ------ | ---------------------- |
| 0x00 | No manufacturer assigned.  Use this before your Manufacturer ID is assigned. |
| 0x01 | [Lynovations](https://lynovation.com/) |
| 0x02 | [Halibut Electronics](https://electronics.halibut.com/) |

Contact the [MoMIDI groups.io list](https://groups.io/g/momidi) to request to be assigned a Manufacturer ID.

# 4. Contributers:

* Mark Smith, Halibut Electronics
  * Email: [mark-momidi@halibut.com](mailto:mark-momidi@halibut.com)
  * GitHub: [@SmittyHalibut](https://github.com/SmittyHalibut/)
  * QRZ: [N6MTS](https://www.qrz.com/db/N6MTS)
* Andrew Rodland, NetKeyer
  * Email: [andrew@cleverdomain.org](mailto:andrew@cleverdomain.org)
  * GitHub: [@arodland](https://github.com/arodland/)
  * QRZ: [KC2G](https://www.qrz.com/db/KC2G)
* Rockwell Schrock, Remote Ham Radio
  * Email: [rockwell@schrock.me](mailto:rockwell@schrock.me)
  * GitHub: [@schrockwell](https://github.com/schrockwell)
  * QRZ: [WW1X](https://www.qrz.com/db/WW1X)
* Lynn Hansen, Lynovations
