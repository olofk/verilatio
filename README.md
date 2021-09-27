This project is deprecated and replaced by https://github.com/olofk/vidbo

This repository will be removed once the relevant information has been migrated

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

If several commands of the same type is sent in the same message, the result is undefined since we can't guarantee ordering in the json struct. Example

```json
/* VCD on and off at the same time. Undefined outcome */
{
"time" : 12345,
"vcd" : false,
"vcd" : true
}
```

TODO : This covers several classes of devices. Would be good to mimic a strcuture here (like the Linux kernel) to avoid having to come up with a device classification from scratch.

TODO: Figure out corner cases and error handling

TODO: Is it necessary to have different sockets for high-bandwidth data. No clue how this works in practice

TODO: PWM for LEDs. Do we send each I/O change (lots of messages) or calculate a PWM value as a float in the sim (how do we decide when to send a new value? First case does not need special protocol handling so lets start with that

TODO: How do we handle 7-segment displays? Just a series of gpio to begin with to make it easy

TODO: Several use cases slot in somewhat with VirtIO. Maybe develop VirtIO<->verilatio bridges for e.g. ethernet, block devices, console

TODO: How to handle non-buffered data streams like UART? One messages for each character? Do some clever queueing? Treat as binary data or text?

TODO: Transfering large chunks of data such as ihex file for programming or maybe a frame from a graphics output might be split across several messages (frames?) Need to investigate if this is a problem

TODO: Should simulator send message with all GPIO state upon connecting? That would both help discover available I/O and sync up both sides

# Protocol

gpio
----
Turn GPIOs on and off. GPIO can be switches, LEDs (RGB LEDs would have three pins associated with them), buttons, 7-segment displays (each segment gets its own gpio pin)

Argument : dict of name/value (string/bool) pairs

dir: both

Example:
```json
/* To simulator */
{
   "gpio":{
      "SW0":true,
      "BTN4":false
   }
}

/* From simulator */
{
   "gpio":{
      "LED2":true,
   }
}
```

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

Argument : dict of name/value (string/string) pairs where name refers to the identifier of the memory to be programmed (in case there are multiple) and the value is the ihex data

dir: to sim

```json
/* Write a ihex file (containing a RISC-V LED blinker) to RAM */
{"memwrite" :
  {"ram" : ":1000000037050040130505003703010023005500A4\n:1000100093C21200B373000093831300E31E73FEB8\n:040020006FF0DFFEA0\n:00000001FF"}
}
```
