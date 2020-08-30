# Asterisk AudioSocket Server

A Python socket server for use with the [Asterisk](https://github.com/asterisk/asterisk) [AudioSocket](https://github.com/CyCoreSystems/audiosocket) Dialplan application and channel driver.

While it's intended to be used like a Python module/library (see example), you can of course use the information
provided inside of the main file to make something more custom fit for your needs.

The creator of the Asterisk-side programs also made a library himself in Golang, which you
can find in the link to the original AudioSocket project above.


**PLEASE NOTE:** While AudioSocket is an absolutely excellent tool, and still one of the only (sorta) built-in ways to extract
raw audio stream of calls from Asterisk. A few oddities exist with how it works that I've encountered, and will explain in more detail below.


## Instructions

### Intro

*What does this allow me to do?*

AudioSocket, whether used as a channel driver or Dialplan application, behaves the same and has the primary purpose of
letting us access an Asterisk channel's incoming and outgoing audio stream and use it in externally for whatever. Though unless attached via ChanSpy, it cannot be used to passively 'intercept' audio on a channel (it blocks execution in the Dialplan whenever its called).

You can also use it to trigger a hangup on the channel from within your program (by calling the audiosocket object's `hangup()` method), that's about it signaling wise, but is really all you need.


### Server usage

After placing the `audiosocket.py` file within your projects directory, you can start using it like this:

```python
from audiosocket import *

audiosocket = new_audiosocket()
audiosocket.start()
```

This would create a new audiosocket object in a separate thread, bind it to all network interfaces 
on the computer using an open port the OS chose (accessible via `audiosocket.port`) an start listening for incoming connections indefinitely.

If you wanted to bind it to a specific address and/or port only and set a timeout on the listening time, you could do this:

```python
audiosocket = new_audiosocket(addr='10.0.0.5', port=3278, timeout=30)
audisocket.start()
```

Internally, FIFO queue objects are used to send/receive audio, but to make usage as simple as possible, queues are created by default and hidden behind the `audiosocket.read()` and `audiosocket.write(<data>)` methods.
If you wanted to pass in your own though, you can do so like this:

```python
custom_receive_q = queue.Queue()
custom_send_q = queue.Queue()

audiosocket = new_audiosocket(rx_audio_q=custom_receive_q, tx_audio_q=custom_send_q)
audiosocket.start()
```
You could then use the `get()` method of the receiveing queue and the `put(<data>)` method of the sending queue to receive and transmit audio without using the `read()` and `write(<data>)` methods of the audiosocket object.

Sending/receiving audio using the provided `read()` and `write()` methods is intended to be done in a `while` loop for as long as `audiosocket.connected` is True. That loop should also send/receive audio to/from another source, for example
you could use [sounddevice](https://github.com/spatialaudio/python-sounddevice) to play audio from AudioSocket to your speakers and send audio from your microphone to AudioSocket, sorta creating a simple softphone.

In the [example](https://github.com/NormHarrison/audiosocket_server/blob/master/example_application.py) usage here, audio is simply read from audiosocket, and then sent back to it, creating an echo on the connected channel.


### Handling audio

By default AudioSocket sends the server audio in the format of 16-bit, 8KHz, mono LE PCM, *at least* when used
as a standalone Asterisk application.

Unfortunately, this is now when weird parts of AudioSocket begin to show up.

When used as a standalone Asterisk application (This has occurred on many different computers and Asterisk versions, spanning three different CPU architectures), for some reason (I'm assuming the reason is within [app_audiosocket.c](https://github.com/asterisk/asterisk/blob/master/apps/app_audiosocket.c) maybe?)
as soon as the application is called and starts sending/receiving audio, **one CPU core on the Asterisk server will remain at 100% usage** until the channel is hungup. This can cause some problems...

I don't know C and I haven't looked through the of the application itself, so I'm not quite sure what could be causing this. Thankfully though there is a way around it.

When AudioSocket is used like a channel driver, for example `Dial(AudioSocket/<uuid>/127.0.0.1:3278)`, CPU usage remains perfectly normal, but... depending on what the AudioSocket is going to bridged with (for example, a softphone connected via SIP), the audio sent to your server will no longer be in
16-bit, 8KHz, mono LE PCM format.

*Instead...* It will be encoded and sent as whatever audio codec was agreed upon between the two channels. So in my experience, when a SIP softphone that uses the u-law (G.711) codec makes a call to a place in the Dialplan
that eventually calls AudioSocket, the audio you will be sent will also be in encoded as u-law, which can be both a positive and negative. Due to Asterisk's ability to handle a
wide range codecs and transcode between them though, I assume there is probably a way around this by manually setting the codec to use within the Dialplan, right before AudioSocket() is called, but I haven't experimented with that yet.

Now with sending audio back to AudioSocket. Even though AudioSocket will send you audio in a different codec when brided with certain channels, **it still wants to receive
16-bit, 8KHz, mono LE PCM** when you send audio back to it.

For me, this was a very difficult thing to deal with initially, until I found that an execellent built-in Python module exists, called [audioop](https://docs.python.org/3/library/audioop.html), for handing raw PCM in many different ways
(resampling it, converting between mono and stereo, converting to/from u-LAW). I strongly recommend using this to prepare your audio source for AudioSocket whenever it's not already in telephone-quality audio, which is probably almost always.

### Integrated [audioop](https://docs.python.org/3/library/audioop.html) features

Certain methods of audioop are now provided within the audiosocket object itself, so if you wanted, you could resample/remix input or output audio like this:


Creation of a new audiosocket object is still done as normal
```python
audiosocket = new_audiosocket(addr='10.0.0.5', port=3278, timeout=30)
```

But now, before calling start(), you can choose to have the server prepare the audio being sent, received, or both by calling these methods and providing correct arguments.

Keep in mind that inrate and channels must match the sample rate and number of channels of the audio data your writing.
By default, CD quality audio is assumed (44000Hz, 16-bit stereo linear PCM), but you can change this to accept whatever values audioop's ratecv() method supports.
```python
audiosocket.prepare_input(inrate=48000, channels=2)
```

For recieved audio, outrate and channels specifiy how you want the server to prepare audio before
returning it to you via the read() method. The argument ulaw2lin is also available, this will convert audio data received in ULAW encoding from Asterisk
to 8Khz, 16-bit mono linear PCM (which you can then upsample too if needed). This is very useful for when AudioSocket is bridged with SIP or IAX channels (which still commonly use ULAW encoding)
```python
audiosocket.prepare_output(outrate=44000, channels=2, ulaw2lin=True)
```

Finally, you would then use the start() method as normal.
```python
audiosocket.start()
```
Please keep in mind that (for now) these integrated audioop methods only work when using the built-in `read()` and `write()` methods (not when sending/receiving audio via queue objects you created).

### Final notes

Throughout the course of trying to use this initially myself, some certain aspects about auduo terminology became clearer to me, but most are still unclear overall, so what I say below should be taken lightly.

AudioSocket sends it's audio data in chunks of 320 bytes, which represents 20ms of 16-bit 8KHZ mono PCM (this is what you will receive when doing this: `audio_data = audiosocket.read()`), and that's also
what it needs to receive back from you when sending audio. Anymore or less will result in distorted audio. Thankfully 20ms seems to be a common length of audio to provide within the world of APIs and probably programming in general (I'm sure there's an explanation of this somewhere). So all you
should have to do to prepare your audio source before sending to AudioSocket, is to downsample it to the required 8KHz mono.
