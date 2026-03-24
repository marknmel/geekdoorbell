# geekdoorbell
A smrt doorbell addon for dumb doorbells.

Why: Because my doorbell sucks and I refused to spend money fixing it properly.

## Disclaimer

This worked for me. It might work for you. It might also let the magic smoke out.

Proceed accordingly.

---

# Introduction

I've got an Edwards Signaling C210-W doorbell. It's… not great. Terrible actually. I can barely hear it 10 feet away, let alone from another room or my office.

And before you say I should just replace it with something new — I'm not doing that because:
- I'm cheap
- I don't know how long I'm staying in this house
- I like the charming lighted push button at my door
- I can't easily run new wiring to the doorbell enclosure
- I'm not sure I actually want a video doorbell
- I don't feel like making tech bros richer

So I'm left with two options: live with a terrible dumb doorbell, or find a way to make it SMRT.

I stumbled across ESPHome recently, and it’s a pretty slick add-on for Home Assistant. It seemed like a perfect fit for this kind of nonsense.

---

## The original method of operation

The Edwards doorbell is a 10V AC system. There’s a small transformer wired to a junction box in my cold cellar, and from there the wires run up to the doorbell enclosure.

At the enclosure, another pair runs to the momentary lighted switch. The light stays on even when the switch is open, because the lamp filament actually completes the circuit — just not with enough current to fire the electromagnet.

When the button is pressed, the circuit can pass enough current to ring the chime bar (ding dong).

---

## The new method of operation

I measured the voltage at the doorbell enclosure and found quite a bit of drop — about 7 volts. The weird part is that it’s *always* 7 volts (thanks again, lighted switch).

So now the problem becomes: how do I turn that into a clean digital signal — pressed vs not pressed?

Warning: this is junk drawer engineering.

I used a small bridge rectifier and a 6V relay. Yes, 6V on a ~7V supply… it’s what I had, and the duty cycle is low. Remember point #1: I’m cheap.

Since the source is AC, the bridge rectifier keeps the relay from chattering. When the button is pressed, the relay energizes and gives me a nice clean set of dry contacts.

---

## The power problem (aka the fun part)

To use ESPHome, I needed a microcontroller. I had an Espressif ESP-01S in the parts bin — perfect.

What I didn’t have was a clean way to power it without opening walls. That just wasn’t happening for this project.

So I built a second DC supply off the same doorbell wiring:
- Another bridge rectifier
- A buck converter (cheap and cheerful from AliExpress)
- A 2200µF / 35V capacitor to smooth things out and prevent brownouts

I briefly considered using a linear regulator, but dropping that much voltage down to 3.3V would just turn it into a tiny space heater.

The capacitor ended up being key — without it, the ESP-01 would brown out when the chime fired.

I’ve created a schematic in case you want to try this yourself. It's in the project repo.

---

## The digital interface

The relay contacts are wired to GPIO2 using the internal pull-up.

When the button is pressed, the relay contacts close and GPIO2 gets pulled to ground. ESPHome then sends a `doorbell_ring` event up to Home Assistant.

Simple. Right!?

---

## The code

You'll find the ESPHome config in this repo.

I’d love to say this is all handcrafted brilliance, but let’s be honest — this is mostly prompt-driven vibe code.

That said, ESPHome makes this kind of thing ridiculously easy, and this is really just a small extension of that simplicity.

You will notice I'm a weirdo.  I have my door bell running several non-door bell services.  What's in a geekdoorbell anyway?

- ota: over the air upgrades (I'm pretty lazy!)
- webserver: handy to get the deets!
- wifi: signal details exposed to HA & Webserver
- DHCP: publishes hostname in DHCP server
- resetButton: In Webserver UI, there is a reboot button
- heartBeat: I want to know that the device is alive in Home Assistant
- doorBell: Oh yeah - a door bell sensor with software debounce

---

# The notifications

I use an Android device with the Home Assistant app installed — because Home Assistant is awesome.

Some people configure Home Assistant entirely in YAML. While I *can*, I do enough of that at work. I try not to turn my house into another job.

Where possible, I stick to the HA GUI.

I created an automation to send a notification to my phone. It looks something like this:

- When: Doorbell Ring Turned ON
- Then do: Notification send to myMobilePhone

Inside that action:

- Message: Ding-Dong
- Checkmark < Title
  - Front Doorbell  
- Checkmark < Data
  - channel: doorbell_alarm
  - importance: max
  - priority: high
  - ttl: 0
  - category: alarm
  - full_screen_intent: true
