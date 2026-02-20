# 1. MoMIDI: Morse over MIDI

Specification for sending Morse Code over MIDI events.

This document builds on existing MIDI protocols used by several hardware and software manufacturers for triggering keying events between hardware keys, and software keyers, transmitters, practice oscillators, etc.  The goal is to use MIDI to trigger key-down and key-up events between morse keys, and software running on a computer (eg: an SDR transmitter, remote station, practice keyer, etc.)

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

- [1. MoMIDI: Morse over MIDI](#1-momidi-morse-over-midi)
	- [1.1. Document History](#11-document-history)
	- [1.2. TODOs](#12-todos)
	- [1.3. Terms](#13-terms)
	- [1.4. Summary of Simple (no timing) Standard](#14-summary-of-simple-no-timing-standard)
	- [1.5. Timing Information](#15-timing-information)
	- [1.6. Timing Rules, and Special Cases](#16-timing-rules-and-special-cases)
		- [1.6.1. No Time Available, or Timer Reset](#161-no-time-available-or-timer-reset)
		- [1.6.2. Short Events: \<127ms](#162-short-events-127ms)
		- [1.6.3. Long Events: \>127ms since last event](#163-long-events-127ms-since-last-event)
		- [1.6.4. Special: 127ms](#164-special-127ms)
		- [1.6.5. Special: N\*128ms](#165-special-n128ms)
	- [1.7. Rules Tables](#17-rules-tables)
		- [1.7.1. Table for Receiver](#171-table-for-receiver)
		- [1.7.2. Table for Sender](#172-table-for-sender)
	- [1.8. Concerns](#18-concerns)
- [2. Contributers:](#2-contributers)

## 1.1. Document History

* 2025-11-24, @SmittyHalibut: First draft.
* 2025-12-03, @SmittyHalibut: Updated timing from 1ms to 2ms resolution. Someone should check my math.
* 2026-01-19, @SmittyHalibut: Going back to 1ms resolution. Added pseudocode and tables to Rules.

## 1.2. TODOs

* Draw a timing diagram for section 1.5

## 1.3. Terms

* Key: In the context of this document, a "key" can be any hardware device meant for sending morse code.
* Straight Key: A key who's timing of dits and dahs is done before the conversion to MIDI.  Either a literal straight key, a bug, or even a hardware iambic keyer, etc.  The important part here is that the straight key has a single electrical contact that controls the transmitter directly.
    * Note: This could also include a hardware iambic keyer.  The output of the hardware keyer is a single contact closure, and all the timing is generated on the user side of the MIDI interface.
* Iambic Key, or Paddles: A key who's timing of dits and dahs is done in software, on the receiving side of the MIDI events.
* MIDI event: MIDI, or the Musical Instrument Digital Interface, is a standard for sending very precise timing events over a serial link.  It's designed around musical instruments, but we are coopting it for another time and latency sensitive use: sending morse code.
  * MIDI Note On event: Includes two 7-bit values: Note, and Velocity
  * MIDI Note Off event: Includes two 7-bit values: Note, and Velocity
  * MIDI Control Change event: Includes two 7-bit values: Channel, and Value

## 1.4. Summary of Simple (no timing) Standard

It's been pretty well established to use the following MIDI Notes to trigger CW events:

* Note On events: Key down
* Note Off events: Key up
* Note 20: left paddle, or straight key
* Note 21: right paddle
* (There are a lot of other MIDI events used for various other radio control features, but we are concentrating on CW here.)

When not sending timing information, send Note On events with a Velocity of 127, and Note Off events with a Velocity of 0.

## 1.5. Timing Information

Sending timing information is optional.  If no timing information is being tracked, send a Velocity of either 127, or 0.  Usually, 127 is sent with Note On events, and 0 with Note Off events.  Both of these values mean "no timing information available" and the software receiving those events measures the time of events by when the MIDI event was received.

MIDI Note On/Off events send a 7-bit Note value, and a 7-bit value called Velocity.  In firmware, we can measure the time between key down and key up events to millisecond precision.  We can send that information to the software in the Velocity field.

7 bits in the Velocity field is only good to measure up to 126ms (remember, 127 means "no timing information") which isn't enough in many cases.  So we can also send a MIDI Control Change event, with an additional 7 bits when needed.  14 bits is enough to count over 16 seconds.  Beyond that, precise timing isn't critical.

> **Side note**: Above 30wpm (40ms per morse symbol), the MIDI latency between key-down and sound-on in a software based keyer (measured using current hardware and software as: 10ms at best, typically 15ms to 25ms) becomes significant.  We assume fast CW operators (above 30wpm) will use hardware based keyers with hardware side-tone.

Timing is tracked between CW MIDI events, even if they were a different event.  For example, when pressing Left, then pressing Right, then releasing Left, and releasing Right, the first time sent is 0 (see below), the second is the time between Left Down and Right Down, the third is the time between Right Down and Left Up, and the fourth is the time between Left Up and Right Up.

**TODO** Diagram for the above.

## 1.6. Timing Rules, and Special Cases

Here are the rules used to encode time since last event 0 .. 16255ms, into MIDI events and values.  The rules are presented in written word, psudo code, and in tables of values below.  It all represents the same logic, in different ways.  If you find any inconsistency, please [file a GitHub Issue](https://github.com/NetKeyer/MoMIDI-Spec/issues/new/choose) so it can be corrected.

MoMIDI use Control Change events with a Value of 0 to handle certain special cases that don't work with normal Long Event and Short Event logic.  They're listed as Rules with "Special" in the name.

### 1.6.1. No Time Available, or Timer Reset

When it's been more than (126<<7 - 1=) 16127 milliseconds since the last event, the timer is reset to zero and not started again until another event is sent.  The next event is sent no Control Change event, and a Note Velocity of 0.

### 1.6.2. Short Events: \<127ms

When it's been **less than** 127 milliseconds since the last event, no Control Change event is sent.  A Note On/Off is sent with the Velocity set to the time since last event.  When the receiver sees a Note On/Off event without a Control Change event, it sets the 7 high order bits of the time since the last event to 0.

Sender logic:
```c
if (Time < 127ms) {
	Control Change = null (don't send)
	Velocity = Time
}
```

Receiver logic:
```c
if (No Control Change event) {
	Time = Velocity
}
```

### 1.6.3. Long Events: \>127ms since last event

When it's been **more than** (2^7-1=) 127 milliseconds since the last event, a Control Change event is sent before the Note On/Off event, to the same Channel number as the Note.  The Control Change event Value contains the 7 high order bits of the time since last event.  When the receiver sees a Control Change event before the Note On/Off event, it shifts the Control Change Value up by 7 bits, then adds the Velocity from the Note event, and uses the result as the number of milliseconds since the previous event.

Sender logic:
```c
if (Time > 127 and Time%128 != 0) {
	Control Change Value = int(Time/128)
	Velocity = Time%128
}
```

Receiver logic:
```c
if (Control Change Value > 0) {
	Time = Control Change Value*128 + Velocity
}
```

### 1.6.4. Special: 127ms

When it's been **EXACTLY** (2^7-1=) 127 milliseconds since the last event, we can't just send a Note with a Velocity of 127, because Velocity=127 without a Control Change means "we aren't tracking timing."  So instead, we send a Control Change event with a Value of 0, then send the Note event with a Velocity of 127.

This can use the same logic as Long Events (Time = ControlChange*128 + Velocity), but it is different than Special: N\*128ms below.

Sender logic:
```c
if (Time == 127) {
	Control Change Value = 0
	Velocity = 127
}
```

Receiver logic:
```c
if (Control Change Value == 0 and Velocity == 127) {
	Time = 127
}
```

### 1.6.5. Special: N\*128ms

When the time since last event has been a whole number multiple of 2^7 (128ms, 256ms, 512ms, etc), the Long Event logic would set Velocity to 0.  Even though it is preceded by a Control Change event, software that is not MoMIDI aware might treat a NoteOn with Velocity of 0 as a NoteOff event.

Instead, we send the Control Change = 0 event, then set the Note Velocity to the high seven bits.  For example, if it's been 512ms since the last event and the user presses the right paddle, the receiver would get a Control Change event to channel 21 with a value of 0, followed by a Note On to note 21 with a value of (512/128=) 4.

Sender logic:
```c
if (Time%128 == 0) {
	Control Change Value = 0
	Velocity = int(Time/128)
}
```

Receiver logic:
```c
if (Control Change Value == 0 and Velocity < 127) {
	Time = Velocity*128
}
```

## 1.7. Rules Tables

### 1.7.1. Table for Receiver

This table maps Control Change and Note events to their Time values.

| **Rule** | **Control Change Value** | **Note On/Off Velocity** | **Time since last event, or other meaning** |
| -------- | ------------------------ | ------------------------ | ------------------------------------------- |
| No Timing | None | 0 | No timing information sent, or timer reset to 0ms. |
| Short Event | None | 1 .. 126 | 1 .. 126ms |
| No Timing | None | 127 | No timing information sent, or timer reset to 0ms. |
| Special: 127ms | 0 | 127 | 127ms |
| Special: N\*128ms | 0 | 1 | 128ms |
| Special: N\*128ms | 0 | 2 | 256ms |
| Special: N\*128ms | ... | ... | ... |
| Special: N\*128ms | 0 | Vel=1 .. 126 | Vel*128ms |
| Invalid ~~Special: N\*128ms~~ | 0 | 127 | Conflicts with 127ms above |
| Long Event | 1 | 1 .. 127 | 129 to 255ms |
| Long Event | 2 | 1 .. 127 | 257 to 383ms |
| Long Event | ... | ... | ... |
| Long Event | CC=1 .. 126 | Vel=1 .. 127 | CC*128 + Vel ms |
| Invalid ~~Long Event~~ | 127 | 1 .. 127 | Unused >= 16127 |
| Invalid | CC=0 .. 127 | 0 | Senders: Do not send. <br/> Receivers: Generate warning, and treat as a Long Event with Velocity=0: CC*128 + 0ms |

### 1.7.2. Table for Sender

This represents the same information as above, but in terms of time, rather than MIDI messages.

| **Rule** | **Time since last event, or other meaning** | **Control Change Value** | **Note On/Off Velocity** |
| -------- | ------------------------ | ------------------------ | ------------------------------------------- |
| No timing available, or reset to 0ms | 0ms | None | 0, or 127 |
| Short Event | 1 .. 126ms | None | 1 .. 126 |
| Special: 127ms | 127ms | 0 | 127 |
| Special: N\*128ms | 128ms | 0 | 1 |
| Long Event | 129 .. 255ms | 1 | 1 .. 127 |
| Special: N\*128ms | 256ms | 0 | 2 |
| Long Event | 257 .. 383ms | 2 | 1 .. 127 |
| ... | ... | ... | ... |
| Special: N\*128ms | 16128ms | 0 | 126 |
| Long Event | 16129 .. 16255ms | 126 | 1 .. 127 |
| Reset timer | >= 16256ms | None | 0 |
| Invalid | Conflicts with 127ms | 0 | 127 |
| Invalid | Unused "Long Event" | 127 | 1 .. 127 |
| Invalid | Avoid Velocity=0 | 0 .. 127 | 0 |

## 1.8. Concerns

If you have any concerns with this system, put them here.

* 2025-12-03, @SmittyHalibut: Should this document be kept to morse events only: left/right paddle, and straight key?  This is where millisecond level timing is most important. But MIDI is used for SO MUCH MORE, it would be good to get that documented too.  'course, if we do that, then "Morse over MIDI" is an increasingly inappropriate name.  I'm open to thoughts on this.

# 2. Contributers:

* Mark Smith, Halibut Electronics
	* Email: [mark-momidi@halibut.com](mailto:mark-momidi@halibut.com)
	* GitHub: [@SmittyHalibut](https://github.com/SmittyHalibut/)
	* QRZ: [N6MTS](https://www.qrz.com/db/N6MTS)
* Andrew Rodland, NetKeyer
* Rockwell Schrock, Remote Ham Radio
* Lynn Hansen, Lynovations
