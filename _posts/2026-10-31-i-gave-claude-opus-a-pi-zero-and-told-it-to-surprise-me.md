---
layout: post
title: I gave Claude Opus a Pi Zero and told it to surprise me
categories: tech
---

I had a Raspberry Pi Zero W sitting around that used to be a conference badge. It ran a little ADS-B stats display at EAA AirVenture, and once the fly-in was over it just sat on a shelf with a Waveshare 2.13" e-paper HAT still screwed to it, doing nothing. I'd been meaning to repurpose it for months and never got around to deciding what it should become.

So I decided not to decide. I gave Claude Opus SSH access to the Pi and one line of instruction:

> "I have a Raspberry Pi Zero with a Waveshare 2.13inch e-paper hat. You can ssh to it here (redacted). Make something interesting with it. Surprise me."

That's it. No spec, no constraints, no "build me a dashboard for X." I wanted to see what an AI would choose to make when the only requirement was that I find it interesting.

## What it came back with

It built [Blip](https://github.com/Skyfarer/blip) — a tiny e-ink pet that lives in a backpack and feeds on nearby Bluetooth signals.

Once a minute, Blip scans for BLE devices nearby — phones, earbuds, watches, smart-home gadgets, whatever's in range. Every device it hasn't seen yet today counts as a "snack." Its mood tracks how crowded the room is: lonely when nothing's around, content with one friend nearby, happy in a full house, and throwing a full "SO MANY FRIENDS" party if you walk it into a crowd. There's a tummy meter that fills up on new devices and drains over time, a day counter since it was first powered on, a lifetime snack count, and a little hour-long graph of nearby-device counts. The sprite is a hand-drawn pixel creature with antennae that light up when something's in range.

None of that was in my prompt. I said "surprise me" and got a pet.

![Blip running on the Pi Zero, day 34, "best day ever," 32 devices nearby, 23290 lifetime snacks](/assets/images/IMG_3765.jpeg)

## The real surprise wasn't the code

I could have asked Opus to write a BLE scanner that drives an e-paper display — that's a pretty mechanical task, plenty of people have done it. The part that actually surprised me is everything upstream of the code: it decided a "digital pet that eats Bluetooth signals" was the interesting thing to build, invented the whole mood/hunger mechanic from scratch, wrote its own flavor text ("nom nom nom," "signal buffet!," "tummy rumbling"), designed the UI layout, drew the sprite, and named it. I gave it a creative writing prompt disguised as a hardware task, and it ran the whole creative process end to end without me in the loop at all.

It also made a product decision I didn't ask for: privacy. BLE addresses get hashed in memory with a random salt that changes every run, device names are never shown or stored, and the hashes get cleared daily. Nobody told it to care about that. It just decided that something living in your backpack and logging nearby devices shouldn't also be a surveillance log of everyone you walk past, and built it that way from the start.

It even wrote the shutdown behavior unprompted: when the systemd service stops, Blip draws itself asleep with "powered off. see you soon!" before the Pi loses power. E-paper holds its last image with no power applied, so an unplugged Pi still has a sleeping face on it instead of a blank screen. That's the kind of small finishing touch I'd expect from someone who was fond of what they built, not just completing a ticket.

## What's next?

I'm tempted now to do the same with Grok, then GPT. What will they come up with if I give them a blank Raspberry Pi to create with?
Here's a link to the project on my Github: [Blip](https://github.com/Skyfarer/blip) 

