---
layout: post
title: Tracking high altitude balloons with WSPR
date: 2021-10-13 12:09 -0500
---

I've always been fascinated with amateur high altitude balloons.  I've always
wanted to launch a balloon with a radio payload and see where it goes and what
data it collects. But, never have I ever, launched a balloon or helped out with
a launch.

Recently I checked on the current state of the art, and I was blown away!
Amateur radio operators are flying solar powered WSPR transmitters on balloons
that stay aloft for weeks at a time. They circumnavigate the globe multiple
times. Several are out there now floating around the earth at about 12000m (40000ft)!

I was already operating a WSPR station from time to time, I'm fascinated that
I can put 1 watt into a piece of wire in my attic, and see stations all over the
world hear my signal.  More on WSPR here at [WSPRNet](https://www.wsprnet.org/drupal/).
So naturally, the challenge of picking up one of these
balloons was too much to pass up.

WSPR balloons are transmitting a standard WSPR packet, which contains a 4 digit
maidenhead grid location. This defines about a 11000 sq km (7000 sq mi) box
where the balloon is located. But some balloons are transmitting a second WSPR
packet that's non-standard. It encodes an additonal 2 digits for the grid
square, plus temperature and altitude in a single WSPR packet. 

There's a tracking site called [habhub](https://tracker.habhub.org/) that's a neat site where you can see these
WSPR balloons, plus APRS and LORA balloons as well. But I was a bit
disappointed in the frequency of updates on the WSPR balloons. I wanted to try
querying the WSPR spots data and decode the packets myself. I found that Dave,
SM3ULC, has written a python script doing this and put it up on Github: [sm3ulc/hab-wspr](https://github.com/sm3ulc/hab-wspr).
I cloned it and tinkered around with it, and decided to write a Ruby version
based on his ideas. Thanks to him, the math formulas were already worked out
for extracting the telemetry from the WSPR packets.

My version is up on Github and there's more details there if you find this
interesting:
[jeremyfsu/hab-wspr-telemetry-decoder](https://github.com/jeremyfsu/hab-wspr-telemetry-decoder) 

Right now, my Ruby script runs every few minutes, queries for WSPR spots that
are balloons, decodes them, then formats APRS packets from the data.  I don't
send the APRS packets into the APRS network, there are people who do this
already for
the balloons. I don't see the need to feed the already engorged APRS network.  Instead, I'm sending the APRS packets to a local
[APRSC](http://he.fi/aprsc/)
server, then mapping them with [Xastir](http://xastir.org/index.php/Main_Page).
Xastir is taking screenshots every few minutes and dropping them here:
[home.jeremywalworth.com/aprs](http://home.jeremywalworth.com/aprs) 
"That maps looks like it's from 2003!" you say. Yes, I run Xastir with a basic setup. 
But yes, I would like to make an embedded Google Map on a page and plot the positions. 
[Habhub](https://tracker.habhub.org/) and [aprs.fi](https://aprs.fi) already do
an outstanding job at this. However, I think it would be cool to see my data
plotted on my own embedded Google Map without sending to the APRS network or
Habhub.  Stay tuned, maybe I'll get around to
doing this. ;-)

It's my hope that a future balloon operator might find my Ruby script useful
for transmitting their balloon packets into the APRS network.
