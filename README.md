# HackerDAQ: A Data Acquisition System for Hackers

HackerDAQ is a work-in-progress project to create an open-source data acquisition system. The project is currently in its early stages, but the goal is to create something that can be used in place of industrial DAQ systems to record high-quality data.

## Background and Motivation

The tech industry is in kind of a weird place when it comes to data acquisition (DAQ) systems. Suppose you have some sort of mechanical system you want to work with -- maybe a science experiment, maybe an industrial process, maybe an automation system, who knows. You have some sensors that you need to read and record the data from, and maybe you want to control some outputs as well, like turning the power on to some valves/solenoids/motors in response to a sensor reading. To do this, you need a DAQ system. However, an odd divide exists in the DAQ industry right now, between the incredibly professional and the almost insultingly hobbyist.

At one end of the spectrum, there are industrial-grade offerings from companies like National Instruments, Beckhoff, and Dewesoft. These systems are capable, flexible, and reliable -- but also extremely expensive. Even an entry-level NI system will run you well into the mid 4 figures, and other manufacturers are similar or worse. What if you can't afford that? Well, in many cases, your only option is getting an Arduino or Raspberry Pi and some basic dev boards for the sensors you need, and then hacking together enough software to log the data over serial. I've seen this many times, and while it does sometimes work, by doing it, you give up a lot of niceties like defined accuracy, noise resistance, a constant sample rate, etc.

Meanwhile, technology has moved on, and we're now in an era where highly accurate, multi-input Analog-Digital Converter (ADC) chips can be purchased very cheaply. This project has selected the [AD7770](https://www.analog.com/en/products/ad7770.html) from Analog Devices, a 21-bit, 32kHz, 8-input ADC which can be purchased for about $15 each at quantity. Using this chip enables DAQ devices to be made quite simply and cheaply, as one basically just needs to attach different front-ends to this chip to convert the signals based on the sensor being sampled. With the existence of this chip and others like it, there is really no justification for DAQ devices to cost thousands of dollars unless you need better accuracy or sampling rate than these types of ADCs can provide.

## Basic Overview

The HackerDAQ would consist of a main board with a microcontroller on it and basic circuitry such as power supplies, an Ethernet port, and a MicroSD slot. The main board would have a number of connectors for cards to plug in, each of which would carry power, GPIOs, and SPI bus signals.

Cards can be designed for many different applications, including digital I/O and many different types of analog sensors. The user could select the cards they need for the application they have in mind, keeping the base cost of the system down and allowing the user to get exactly the I/O they need. The SPI bus was selected for communications with these cards as it is fast, there are a wide range of chips available that speak it, and it supports bussing together multiple chips on the card side over one set of bus lines. In a lot of ways, these cards would be similar to existing sensor interconnect systems like SparkFun Qwiic / Adafruit Stemma / Digilent Pmod, where you can buy something off the shelf and connect it to your existing HW. This would just be focused more on DAQ stuff and be more industrial-grade.

In terms of software, we would provide firmware that would run on the board and interface with all the cards, send data to connected user interface(s), and receive and execute commands. On the PC side, there would be a basic program that can display and record data. The goal would be that for users who don't want to program, they can see the live data in the GUI and record it to a file and can manually control outputs. For users who are a little more comfortable with programming, they could write scripts (likely python) that would run on the host machine and be able to control the DAQ. And for users who have embedded software experience, they could write their own code that extends or replaces the firmware on the actual DAQ!

## Key Decisions

| Area | Decision | Why |
|------------|-------------|--------------|
|Physical architecture | Main board + swappable cards | <ul><li>Avoids need to create multi-function inputs which would increase complexity</li><li>Makes system cheaper if you only need a few types of inputs</li><li>Avoids need to mess around with external amplifiers or other circuitry for sensors that need them -- they can be built directly into the card</li><li>Allows for much wider range of off-the-shelf supported functionality</li></ul>|
|CPU|NXP MCX Alpha Microcontroller running Mbed OS|<ul><li>Using a microcontroller, rather than a linux CPU + supporting circuitry, keeps the cost down</li><li>Using an RTOS rather than a full OS makes microsecond-level timestamping and SPI transaction scheduling possible</li><li>This MCU has two CPU cores at 150 MHz, plus a powerful DMA controller, a fast internal bus, and lots of SPI connectivity</li></ul>|
|SPI bus topology|2 card connectors per SPI bus, one leader card and one follower. All cards on a given bus must share the same polling rate.|<ul><li>MCU does not have enough pins for each of the 8 slots to have its own dedicated SPI bus, so there needs to be some way for cards to share a bus.</li><li>In most real systems, you generally have a lot of sensors at the same data rate, so I do not expect this to be too crippling of a limitation</li></ul>|
|Connection to host|Ethernet, potential upgrade later to add USB|<ul><li>Ethernet provides high quality electrical isolation and is highly noise-immune, so it's great for industrial applications</li><li>Ethernet peripherals in microcontrollers are well-optimized for high data rate applications</li><li>Allows the DAQ to connect to a larger network and interact with multiple machines, plus receive time via NTP</li></ul>|
|Local recording|MicroSD (SDIO or SPI - TBD)|<ul><li>Important to have a method of locally recording data for standalone applications</li><li>Mbed has a preexisting driver for MicroSD cards in SPI mode</li><li>SDIO mode is faster but driver support is less present (though not totally hopeless either)</li></ul>|
|Card identification|4kiB SPI EEPROM on the card (thinking M95320-RDW6TP)|<ul><li>Allows SW to largely autoconfigure itself based on the cards that are physically installed</li><li>Provides a convenient place to store calibration data!</li></ul>|
|Card to sensor connectors|Molex Mini-Fit|<ul><li>I have used these connectors in the past and they are really balanced all-around:<ul><li>Retention clip ensures they can't vibrate loose</li><li>Rated for ~8A continuous per pin</li><li>Good size: not so small that they are hard to work with, but not so big that they make your entire system bigger</li></ul></li><li>Big downside is that the crimp tool costs $400. Could mitigate by supplying one wire harness with each card that you can splice your sensor wires to</li></ul>|
|Card to main board connector|15-pin, 2 row D-Sub (DA-15)|<ul><li>Very cheap</li><li>Screws provide relatively beefy mechanical retention</li><li>Not a size used in consumer hardware so unlikely to be plugged in to, say, a monitor accidentally</li><li>Still uses standard D-sub hardware so it should be possible to get premade cables, crimpers, etc without too much trouble</li></ul>|

## The Main Unit
The main unit would contain a relatively fast microcontroller with Ethernet, multiple SPI busses, a MicroSD slot, and a number of GPIO pins. 

### Data Recording
Both the Ethernet and the MicroSD would be available for recording data. The Ethernet would be for livestreaming when you want to view and process on a PC, while the MicroSD card would be for simple applications where you just want to record locally, and could also be configured as a backup so that data isn’t lost if the ethernet connection drops out.

### Power
The main unit would regulate power for the HackerDAQ system. It would supply a 12V rail with 1A+ of capacity, plus 5V and 3.3V logic (unfiltered!) rails, to the main board and all cards.

### Card Ports
The main unit would have a number of SPI busses (likely 3-4). Each SPI bus would have multiple (2-3) card ports on it. Card ports on the same bus can have different cards connected, but *must* operate at the same speed as SPI transactions for all three cards will happen at once.

### Expansion Port
Some of the microcontroller I/O pins that are not connected to one of the SPI busses will be routed to the expansion port. This will include some digital pins (with I2C/PWM/UART) and some analog pins. Advanced users who are comfortable programming embedded C++ can add write their own software that uses these pins. This allows the HackerDAQ to be used as a jumping-off point for one's own projects!

Also, SWD debug pins will be broken out to a header, for ease of debugging with something like an MCU-Link.

### Open Questions
- Is it worth adding a battery-backed RTC so that the time can be maintained if there is a power outage?
- Does PoE support make sense?

### Card Interface
The main unit would provide the following signals to each card:
- SPI signals (MOSI, MISO, SCLK)
- 3x CS lines
    - This allows having 2 addressable chips on the card side, plus the EEPROM
    - 8.192MHz clock signal for the ADC
    - +12V, +5V, and +3.3V digital rails
    - A data ready line from the card
    - A sync line to the card

### Analog Card Basics
Each analog card would contain the same basic DNA:
- Analog front-end, with ESD protection and passive filtering
- Instrumentation amplifier, if needed
- Voltage reference, LTC6658 on cards that need to supply precision excitation voltage, other cheaper option (or built-in ref) if only voltage input is needed
- Multi-input ADC with SPI interface
- SPI EEPROM that stores the card's identity information and calibration