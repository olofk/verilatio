# verilatio
A protocol for communicating with HDL simulations over websockets

## Background
All chip designs inevitably have some kind of inputs and outputs to the world outside of the device under test. This I/O can be modeled in different ways for different purposes. For regression tests this often takes the form of predefined stimuli sent into the device and outputs being monitored to make sure the device did the expected thing. In other cases however, there is a need to communicate interactively with the simulated design. As an example this can be a simulation of a SoC running a user program, where the user wants to send commands over a simulated UART connection and see that the program behaves as expected by observing changes in the device's output. This adds more requirements to the simulated design, especially when there are multiple types of inputs and outputs. Many solutions exist already for various cases, such as outputting the UART output on the screen, creating a window to show graphics and communicating over sockets and FIFOs. One common problem with these solutions however is that they decrease the portability by depending on external libraries which can be hard to compile on some platforms. They are also not standardized which makes reuse harder

This is a proposal for a protocol to be used over websockets to handle input, output and run control of chip designs running in simulations. Websockets was chosen since it's a a light-weight extension over standard sockets and allows the client side to be implemented in a web browser or any program that implements websockets.

Some use cases for the protocol are

* VCD on/off (to just capture the interesting parts)
* Software loading
* 2-way UART
* Data loading (e.g. SPI Flash emulation)
* Video output
* Ethernet
* Run and pause
* Emulated sensors

Note: No attempt to emulate communication on bus-level like SPI. Focus on higher-level use cases like memories and sensors. Let simulator figure out the lower-level details.

Basics
------
Messages are JSON-encoded. All messages from simulator carries a timestamp of current simulator time. This time can not be decreasing. Simulator can send messages only containing the timestamp to inform the client of the current simulation time.

```json
/* Turn on LED 0 */
{
"time" : 12345,
"led" : {"0" : true}
}
```

Messages from client to simulator does not contain timestamps.

```json
/* Turn off switch 3. Turn on VCD capturing */
{
"switch" : {"3" : false},
"vcd" : true
}
```

Main websocket channel is text. Binary high-bandwidth data (e.g. ethernet and video) are sent over separate sockets as binary data. Client can ask simulator which additional sockets are used.

TODO : This covers several classes of devices. Would be good to mimic a strcuture here (like the Linux kernel) to avoid having to come up with a device classification from scratch.

TODO: Figure out corner cases and error handling

TODO: Is it necessary to have different sockets for high-bandwidth data. No clue how this works in practice

TODO: Should we be specific about I/O like LEDs, buttons and switches or just lump the together as gpio? I do think that's better, really.

TODO: What about slightly more complex I/O e.g. RGB leds? How do we handle 7-segment displays?

TODO: Several use cases slot in somewhat with VirtIO. Maybe develop VirtIO<->verilatio bridges for e.g. ethernet, block devices, console

TODO: How to handle non-buffered data streams like UART? One messages for each character? Do some clever queueing? Treat as binary data or text?

TODO: Transfering large chunks of data such as ihex file for programming or maybe a frame from a graphics output might be split across several messages (frames?) Need to investigate if this is a problem

TODO: Allow LED brightness to be a float to emulate PWM'd LEDs

# Protocol

led
---
Turn LEDs on and off

Argument : dict of name/value (string/bool) pairs

dir: to sim

switch
------
Turn switches on and off

Argument : dict of name/value (string/bool) pairs

dir: from sim

vcd
---
Turn waveform capturing on and off

Argument : bool

dir: to sim

line
----
A line of text for line-buffered communication

Argument : string

dir : both

run
---
Run/pause simulation

Argument : bool

dir: to sim

memwrite
--------
Write an Intel Hex file to memory.

Argument : string

dir: to sim
