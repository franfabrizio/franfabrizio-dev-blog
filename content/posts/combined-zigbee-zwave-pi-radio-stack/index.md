---
title: "Rebuilding My Smarthome with a Combined Zigbee & Zwave Pi-Based Radio Stack"
date: 2025-08-17T23:38:44-05:00
draft: true
toc: true
---

## Introduction

Settling on a home automation platform can be stress-inducing. Between Zigbee, Z-Wave, Wifi, Thread+Matter... what standard to use is not always obvious, not to mention the many different vendor platforms built on top of these, like SmartThings, Wyze, Philips Hue, and so on. Because I didn't build out my Smarthome particularly intentionally, I've now got a number of different Zigbee and Zwave devices using a Smartthings hub and a growing wifi-based Wyze ecosystem. Many of my devices are quite old by now, and there are tons of never-done project ideas I just didn't get around to. So in 2025, I want to do a ground-up redo of my smart home. But what should I standardize on?

I conceptualize these choices as:

* Zigbee: cost-effective, huge device selection, limited signal range, less robust implementations
* Z-Wave: costlier devices (1.5-2x), good device selection, significantly better signal range, a robust standard
* Wifi: ubiquitous and cheap, pollutes my homelab network with lots of IoT devices, energy-inefficient, seems like overkill
* Thread+Matter: the future, but still emerging, limited device selection, the transition is going to take a long time (maybe a decade)
* Vendor Platforms: No. I want to get away from needing vendor apps on my phone and shipping data into various vendor clouds. I want an open platform running my house.

With this framework in mind, I've embarked on a complete rework of my ad-hoc, organically-grown smarthome setup.

As I considered my alternatives, I quickly realized two things:

1. I want to get off of my wifi for smart home devices wherever possible to declutter my network, and
2. Thread+Matter isn't ready yet.

While I'd keep an eye on a future path to Thread+Matter, for now that left me with a choice of Zigbee or Z-Wave. And I couldn't decide, so I've decided to support both. My thought is that for mission-critical things, like anything related to the security of my house, I should use Z-Wave, whereas for more trivial things like "welcome home" lighting scenes and such, I should use the more cost-effective Zigbee. Supporting two protocols would be a chore, but then I'd have more flexibility to invest the right amount of money into the various aspects of my smart home, and in cases where one protocol had better device choices than the other, I wouldn't have to compromise.

So, the task now is to support two protocols with the least hassle possible. From a technical sense, that meant presenting both Z-Wave and Zigbee messages to HA in the most consistent way I could.

## Architecture

I would of course need Zigbee and Z-Wave radios to serve as the hubs of their respective device mesh networks. The radios would need to be hosted by some sort of computer, and then from there I would need to relay messages to my Home Assistant instance.

To simplify my Home Assistant instance, I want to converge the various protocols in use in my smart home setup into MQTT[^mqtt] at the earliest possible moment, so that my HA instance would not need to speak native Z-Wave or Zigbee, and could just receive everything through a single MQTT integration point.

MQTT will bring other benefits to my HA setup as well. It might help provide a legacy integration point for some of my vendor-cloud-locked wifi devices (if they support MQTT or can be flashed with custom firmware), so I could have a more leisurely transition off of my wifi devices and not need to replace everything all at once. More importantly, once MQTT becomes the standard way my devices talk to HA, it becomes easier to integrate other consumers of my smarthome data such as Influx/Grafana, which makes it easier to keep HA focused on real time control/state and keep Influx DB and Grafana Dashboards as a robust time series database for long term history. Plus whatever else I can dream up - MQTT gives me a pub/sub broker that anything can subscribe to.

## Hardware

For hardware, what I decided to do is to use a Raspberry Pi to host both Z-Wave and Zigbee radio dongles and then to pass messages along to HA in whatever way made the most sense. A Pi would be great for being both low power and low profile. My HA instance runs on a Pi 4, but it lives in my networking closet in a far corner of my basement. This wouldn't be great for signal strength to the rest of the house to host the radio dongles on this Pi, so I instead set out to build the lightest-weight setup that I could and then install it in the attic of my one story (plus basement) house, which is my easiest point of access to place radios centrally without having unsightly tech in, say, the middle of my living room.

To simplify the install I wanted to leverage the existing PoE capabilities of my homelab, which I have used extensively in my attic to deliver power to my cameras and APs. So, I obtained a Pi 3B+ and the PoE+ HAT for it. Then I ordered a Sonoff Zigbee 3.0 USB Dongle Plus (from Amazon) and a Zooz 800 Series Z-Wave Long Range USB Stick ZST39 (from The Smartest House), and a couple of 6' USB extension cables (to separate the radios and avoid interference). I found a nifty design for a Pi 3B+ + PoE HAT case and mount that I 3D printed, as well.

With hardware in hand, I proceeded to software setup.

## Software

Again with lightweight solutions in mind, I imaged the Pi with Raspberry OS Lite. I then began to set up a docker-based software stack for the Zigbee and Z-Wave radios. We'll need something to receive Zigbee messages and translate them to MQTT messages, something to do the same on the Z-Wave side, and finally, an MQTT broker to receive the messages and pass them along to Home Assistant. Since both radio stacks will need the MQTT broker, let's start there.

### MQTT Broker - Mosquitto

For our MQTT broker, we'll install Mosquitto. There's not much to this, just use the Mosquitto docker container...

```yaml
# docker-compose.yml

services:
  mosquitto:
    image: eclipse-mosquitto:2.0.18
    container_name: mosquitto
    restart: unless-stopped
    environment:
      - TZ=America/Chicago # change as needed
    volumes:    # map these to however you provide persistent storage in your docker environment
      - ./config/mosquitto:/mosquitto/config:ro
      - ./data/mosquitto:/mosquitto/data
    ports:
      - "1883:1883"
```

... and set up a minimal configuration...

```t
# mosquitto.conf
# Persistence
persistence true
persistence_location /mosquitto/data/

# Listeners
listener 1883
allow_anonymous true  # you can lock this down if you want

# Optional WebSocket UI/debug (comment out if not needed)
# listener 9001
# protocol websockets

# Logging
log_timestamp true
```

With a broker in place, now we can move on to the protocol-specific handlers.

### Zigbee - Zigbee2MQTT

Here we'll use Zigbee2MQTT to process Zigbee radio traffic and push messages to MQTT. This setup is also pretty straightforward, though a bit more involved than the Mosquitto config we just completed.

The trickiest part was that I first had to flash the Sonoff dongle firmware to a current version. Thankfully, they provide [a web flasher](https://dongle.sonoff.tech/sonoff-dongle-flasher/) for that.

With updated firmware, I now plugged the dongle into the Pi and observed the kernel log to make sure the device got recognized. As this was the only USB device on this Pi at this point, it was mapped to /dev/ttyUSB0, but to provide a more durable mapping, I looked for its id-based mapping instead:

```sh
$ ls /dev/serial/by-id/
usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_6e4d3e172d53ef118b902ae9294bec31-if00-port0
$
```

When I configure the container, I will use that to reference the Zigbee coordinator dongle, because that's more durable and not going to change as devices get plugged and unplugged.

To set up the container:

```yaml
# docker-compose.yml

 zigbee2mqtt:
    image: koenkk/zigbee2mqtt:1.39.0   # pin a version
    container_name: zigbee2mqtt
    restart: unless-stopped
    environment:
      - TZ=America/Chicago
    depends_on:
      - mosquitto
    volumes:
      - ./data/zigbee2mqtt:/app/data
      - ./config/zigbee2mqtt/configuration.yaml:/app/data/configuration.yaml:ro
    devices:
      # map to /dev/ttyUSB0 inside container
      - "/dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_6e4d3e172d53ef118b902ae9294bec31-if00-port0:/dev/ttyUSB0"
    ports:
      - "8080:8080"   # Zigbee2MQTT Web UI
```

Then, a Zigbee2MQTT configuration file:

```yaml
homeassistant:
  enabled: true
frontend:
  enabled: true
  port: 8080
mqtt:
  server: mqtt://mosquitto:1883
  base_topic: zigbee2mqtt
serial:
  port: /dev/ttyUSB0
  # small trick here - see below
  adapter: ember
  baudrate: 115200
  rtscts: false
advanced:
  # see below for explanation of these settings
  channel: 25
  log_level: info
  pan_id: 0xAC37
  network_key: GENERATE # let Z2M create it for you
  # alternately, specify one with 16 random bytes like this
  # network_key: [0x5F,0x19,0xA2,0x7C,0x41,0xE3,0x90,0x0B,0xD4,0x2A,0x66,0x12,0x3C,0x88,0x7E,0x51]  

version: 4
```
A couple of things need explanation here.

First, the Sonoff dongle has had a couple of different hardware revisions, which used different Zigbee chipsets. That will affect what setting you use here for `adapter:`. Some versions use the TI CC2652/CC1352 chipset (along with many other Zigbee dongle vendors). For those, you use `adapter: zstack`.  Other Sonoff dongle versions use the Silicon Labs EFR32 chipset. Those require `adapter: ember` (you may see older documentation that refers to `adapter: ezsp`, but this is deprecated in Zigbee2MQTT). For adapters using chipsets other than these, check online documentation.

Second, some of the advanced settings need some context.

* channel - In the US, channel 25 is often recommended because it has the least overlap with the standard 2.4GHz wifi channels (1/6/11).
* pan_id - This is like a unique SSID for your Zigbee network. It's a 4-digit hexadecimal number, you can pick anything (except Ox0000 and OxFFFF, those are special), just don't change it after you set it.
* network_key - This is the encryption key for your network. You can just let Z2M `GENERATE` it for you, but if for whatever reason (e.g. migrating an existing one) you need to specify it, you can.

**The important thing is to not change channel, pan_id, or network_key once you start building out your network. If you do, be prepared to re-join all of your devices.**

TBD how do I start and verify?

### Z-Wave - ZWaveJS-UI

[^mqtt]: Message Queuing Telemetry Transport, a message protocol designed to reliably deliver small messages over flaky networks, originally for use with satellites and now widely adopted for IoT.







