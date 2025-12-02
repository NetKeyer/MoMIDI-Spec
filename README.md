# MoMIDI: Morse over MIDI

Specification for sending Morse Code over MIDI events.

This document builds on existing MIDI protocols used by several hardware and software manufacturers for triggering keying events between hardware keys, and software keyers, transmitters, practice oscillators, etc.  The goal is to use MIDI to trigger key-down and key-up events between morse keys, and software running on a computer (eg: an SDR transmitter, remote station, practice keyer, etc.)

<div class="classictemplate template" style="display: block;">
	<style type="text/css">
		#groupsio_embed_signup input {border:1px solid #999; -webkit-appearance:none;}
		#groupsio_embed_signup label {display:block; font-size:16px; padding-bottom:10px; font-weight:bold;}
		#groupsio_embed_signup .email {display:block; padding:8px 0; margin:0 4% 10px 0; text-indent:5px; width:58%; min-width:130px;}
		#groupsio_embed_signup {
		background:#fff; clear:left; font:14px Helvetica,Arial,sans-serif;
		}
		#groupsio_embed_signup .button {

		width:25%; margin:0 0 10px 0; min-width:90px;
		background-image: linear-gradient(to bottom,#337ab7 0,#265a88 100%);
		background-repeat: repeat-x;
		border-color: #245580;
		text-shadow: 0 -1px 0 rgba(0,0,0,.2);
		box-shadow: inset 0 1px 0 rgba(255,255,255,.15),0 1px 1px rgba(0,0,0,.075);
		padding: 5px 10px;
		font-size: 12px;
		line-height: 1.5;
		border-radius: 3px;
		color: #fff;
		background-color: #337ab7;
		display: inline-block;
		margin-bottom: 0;
		font-weight: 400;
		text-align: center;
		white-space: nowrap;
		vertical-align: middle;
		}
	</style>
	<div id="groupsio_embed_signup">
		<form action="https://groups.io/g/momidi/signup?u=5426540281110385330" method="post" id="groupsio-embedded-subscribe-form" name="groupsio-embedded-subscribe-form" target="_blank">
			<div id="groupsio_embed_signup_scroll">
				<label for="email" id="templateformtitle">Subscribe to the MoMIDI groups.io email list:</label>
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

## Document History

* 2025-11-24, @SmittyHalibut: First draft.

## Terms

* Key: In the context of this document, a "key" can be any hardware device meant for sending morse code.
* Straight Key: A key who's timing of dits and dahs is done before the conversion to MIDI.  Either a literal straight key, a bug, or even a hardware iambic keyer, etc.  The important part here is that the straight key has a single electrical contact that controls the transmitter directly.
    * Note: This could also include a hardware iambic keyer.  The output of the hardware keyer is a single contact closure, and all the timing is generated on the user side of the MIDI interface.
* Iambic Key, or Paddles: A key who's timing of dits and dahs is done in software, on the receiving side of the MIDI events.
* MIDI event: MIDI, or the Musical Instrument Digital Interface, is a standard for sending very precise timing events over a serial link.  It's designed around musical instruments, but we are coopting it for another time and latency sensitive use: sending morse code.

## Simple State

It's been pretty well established to use the following MIDI Notes to trigger CW events:

* Note 20: left paddle, or straight key
* Note 21: right paddle
* (There are a lot of other MIDI events used for various other radio control features, but we are concentrating on CW here.)

A Note On event is sent to signal the key is pressed, and a Note Off event is sent to signal the key is released.

## Timing information

Sending timing information is optional.  If no timing information is being tracked, send a velocity of either 127, or 0.  Usually, 127 is sent with Note On events, and 0 with Note Off events.  Both of these values are ignored and the software measures the time of events by when the MIDI event was received.

MIDI Note On/Off events also send a 7 bit value called Velocity.  In firmware, we can measure the time between key up and key down events to millisecond precision.  We can send that information to the software in the Velocity field.

7 bits in the Velocity field is only good to measure up to 126ms (remember, 127 means "no timing information") which isn't enough in some cases.  So we can also send another MIDI event called a Control Change event, with an additional 7 bits.  14 bits is enough to count 16384ms, over 16 seconds.  Beyond that, precise timing isn't critical.

Timing is measured between CW MIDI events, even if they were a different event.  For example, when pressing Left, then pressing Right, then releasing Left, and releasing Right, the first time sent is 0 (see below), the second is the time between Left Down and Right Down, the third is the time between Right Down and Left Up, and the fourth is the time between Left Up and Right Up.

### Timer Reset

When it's been more than 2^14-1 (16383) milliseconds since the last event, the timer is reset to zero and not started again until another event is sent.  That first event is sent with a time of 0.  (See the example above.)

### Long Events

When it's been more than 2^7-1 (127) milliseconds since the last event, a Control Change event is sent to the same channel number as the Note number, before the Note On/Off event.  The CC event contains the 7 high order bits.  The receiver sees a CC event before the Note On/Off event, it shifts the value of the CC up by 7 bits, then adds the Velocity from the Note event, and uses the result as the number of milliseconds since the previous event.

### Short Events

When it's been less than 2^7-1 (127) milliseconds since the last event, no Control Change event is sent.  The receiver sees a Note event without a CC event, it sets the 7 high order bits to 0.

### Special Cases

When it's been EXACTLY 2^7-1 (127) milliseconds since the last event, we can't just send 127 in the Note because 127 means "we aren't tracking timing."  So instead, we send a Control Change event with a value of 0, and send a Note event with a Velocity of 126.  When the receiver receives a CC of 0 and a Note of 126, they treat that as 127.  When the receiver receives a Note of 126 without a preceding CC, then its an actual 126.

When it's been a whole number multiple of 2^7 (128, 256, 384, etc) milliseconds since the last event, the Velocity in the Note event will be 0, but it will be preceded by a Control Change event with a non-zero value.  When the receiver receives a CC with non-zero, followed by a Note with a Velocity of zero, the receiver MUST NOT ignore the timing in the note.

## Examples

Below is a timeline showing what MIDI events are sent depending on how long it's been since the last MIDI event.

Simple short key presses:
* Time 0:  Left paddle pressed.  NoteOn Note=20, Velocity=0
* Time 100: Left paddle released.  NoteOff Note=20, Velocity=100
* Time 200: Right paddle pressed.  NoteOn Note=21, Velocity=100
* Time 250: Right paddle released.  NoteOff Note=21, Velocity=50

Interleaved key presses.  Note how timing is from the most recent event, even if its a different event.
* Time 300: Left paddle pressed.  NoteOn Note=20, Velocity=50
* Time 400: Right paddle pressed.  NoteOn Note=21, Velocity=50  (measured from the Left pressed.)
* Time 450: Right paddle released.  NoteOff Note=21, Velocity=50
* Time 500: Left paddle released.  NoteOff Note=20, Velocity=50

Long time measurement, greater than 126ms:
* Time 700: Left paddle pressed.  CC Channel=20, Value=1.  NoteOn Note=20, Velocity=72
* Time 900: Left paddle released.  CC Channel=20, Value=1.  NoteOff Note=20, Velocity=72

Longer than 2^7-1 milliseconds, timer resets to 0:
* Time 30000: Left paddle pressed.  NoteOn Note=20, Velocity=0

Exactly 127ms:
* Time 30127: Left paddle released.  CC Channel=20, Value=0.  NoteOff Note=20, Velocity=126

Whole number multiple of 128:
* Time 31000: Right paddle pressed.  CC Channel=21, Value=6.  NoteOn Note=21, Velocity=105
* Time 31256: Right paddle released.  CC Channel=21, Value=2.  NoteOff Note=21, Velocity=0

## Concerns

If you have any concerns with this system, put them here.

* 2025-11-24: @SmittyHalibut  I worry about N*128 values.  They result in a Velocity of 0, with a preceding CC event.  But software that only looks events, and not the timing, may see a NoteOn Velocity=0 and treat that as a Note Off.  Should we say in that case that we send a velocity of 1ms, which breaks the exact precision of our measurements, but only by 1ms which is pretty small, even at 40wpm (30ms dit length).
