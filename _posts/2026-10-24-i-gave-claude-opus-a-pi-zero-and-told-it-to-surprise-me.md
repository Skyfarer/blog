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

## The real surprise wasn't the code

I could have asked Opus to write a BLE scanner that drives an e-paper display — that's a pretty mechanical task, plenty of people have done it. The part that actually surprised me is everything upstream of the code: it decided a "digital pet that eats Bluetooth signals" was the interesting thing to build, invented the whole mood/hunger mechanic from scratch, wrote its own flavor text ("nom nom nom," "signal buffet!," "tummy rumbling"), designed the UI layout, drew the sprite, and named it. I gave it a creative writing prompt disguised as a hardware task, and it ran the whole creative process end to end without me in the loop at all.

It also made a product decision I didn't ask for: privacy. BLE addresses get hashed in memory with a random salt that changes every run, device names are never shown or stored, and the hashes get cleared daily. Nobody told it to care about that. It just decided a thing that lives in your backpack and logs nearby devices shouldn't also be a surveillance log of everyone you walk past, and built it that way from the start.

And it didn't stop at the first working version. The git history shows it coming back to tune its own earlier decisions — adjusting how fast the tummy drains, changing how much each new device fills it, adding 24-hour persistence so "eaten" devices survive a restart, handling device names that weren't valid UTF-8. That's not one-shot code generation, that's something closer to a person iterating on their own project because they weren't quite happy with the first pass.

It even wrote the shutdown behavior unprompted: when the systemd service stops, Blip draws itself asleep with "powered off. see you soon!" before the Pi loses power. E-paper holds its last image with no power applied, so an unplugged badge still has a sleeping face on it instead of a blank screen. That's the kind of small finishing touch I'd expect from someone who was fond of what they built, not just completing a ticket.

## It still had to do real engineering

This wasn't just vibes and personality, either. It ran into a genuinely nasty bug: `btmgmt`, the Bluetooth scanning tool it used, exits immediately the moment its stdin hits EOF. Under systemd, stdin is `/dev/null` by default, which is EOF from the first instant — so the scan silently returned zero devices every single time, with no error. It tracked that down and fixed it by holding a stdin pipe open for the duration of the scan. That's a real "why is this returning nothing with no error message" debugging session, the kind that eats an afternoon for a human.

## Where Blip lives now

It's back on the original Pi Zero, running as a systemd service, drawing a new mood every minute off whatever's nearby. The old ADS-B badge service is still on there too, disabled but intact, so I can switch the Pi back to its old job with two `systemctl` commands if I ever want to. For now it's a pet.

I don't have a grand conclusion about what this means for AI-assisted builds generally, other than: I'm going to keep handing Opus hardware with no spec, because "make something interesting, surprise me" turned out to be a much better prompt than anything I would have written myself.
