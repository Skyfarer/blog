---
layout: post
title: Ruby vs NodeJS performance on large text files
categories: tech
---

I'm working on a project that uses weather forecast data from the Euro model, also
known as the ECMWF. The standard format in the Meteorology world is called
GRIB. It's a binary format but laid out in a series of records where each
record contains geolocation, a few other general parameters, then one
significant parameter like temperature. For this project, I need temperature,
dewpoint and wind. I'm having to make multiple passes over a GRIB file, one for
each parameter I need. Add to this, the GRIB contains over 800,000 records
within the geographic areas I want. 

I started with Ruby, taking each record and storing it in a Redis cache, but
running this synchronously results in rate of about 800 records per second. 
This means about 16 minutes for each parameter (4) so an hour for one forecast
interval. Why 4? Well, wind is in the GRIB as two values, velocity in the
X direction and in the Y direction.
Since I want the forecast every 6 hours for 5 days, this rapidly explodes to 20
hours of processing data. Well, these could run in parallel, but I think
there's a better tool.

Having used NodeJS on other projects, I realized an asynchronous language like
NodeJS is the better tool for this job. So I gave Perplexity a crack at
translating my Ruby script to NodeJS. What it gave me was not too bad, and after
some corrections, my test runs showed a dramatic improvement. I can now process
800,000 records and store them in Redis in less than 10 seconds.  

