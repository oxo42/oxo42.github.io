---
layout: post
category: splunk
date: 2015-05-18
title: Splunk my Garmin - Get the data in - Part 1
tags: [splunk, garmin, fitness, dropbox, tapiriik]
---

{% include JB/setup %}

## Splunk my Garmin Series
* [Part 1: Get the data in](splunk-my-garmin-part1.md)
* [Part 2: Make sense of TrackPoint]
* [Part 3: Build the app]
* [Part 4: Investigate the API]
* [Part 5: ???]

Ever since [Dave](http://blogs.splunk.com/author/dgreenwood/) wrote about [Downhill Splunking](http://blogs.splunk.com/2015/03/22/downhill-splunking-part-1/) I've wanted to suck my [Garmin Connect](https://connect.garmin.com/modern/profile/John_Oxley_) activities into Splunk and explore.  This is the first of a multi-part post on my experience doing so.


## Where my files at?

I've been using [Tapiriik](https://tapiriik.com/) for a long time to sync my Garmin activities to Endomondo.  At the time I decided what's the harm in putting them in [Dropbox](https://db.tt/a9NHe6F).  Turns out that was a great idea, because I now have a year's worth of data to play with.


## Splunk it up!

Start by creating a `fitness` app.  All sourcetypes, macros, extractions, dashboards etc are going to go in here.  Once I've got something useful, I'll push it up to GitHub and publish on Splunkbase.

* Create index in splunk `fitness`

Put a monitor on the directory so files automatically get indexed.

    cd C:\users\Me\DropBox\Apps\tapiriik
    splunk add monitor . -index fitness -sourcetype tcx


## What is TCX

TCX is Training Center Database XML file.  it looks like:

{% highlight xml %}
<?xml version='1.0' encoding='UTF-8'?>
<TrainingCenterDatabase xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:ns2="http://www.garmin.com/xmlschemas/UserProfile/v2" xmlns:tpx="http://www.garmin.com/xmlschemas/ActivityExtension/v2" xmlns:ns5="http://www.garmin.com/xmlschemas/ActivityGoals/v1" xmlns:ns4="http://www.garmin.com/xmlschemas/ProfileExtension/v1" xmlns="http://www.garmin.com/xmlschemas/TrainingCenterDatabase/v2">
  <Activities>
    <Activity Sport="Biking">
      <Id>2015-05-17T07:41:32.000Z</Id>
      <Lap StartTime="2015-05-17T07:41:32.000Z">
        <TotalTimeSeconds>14599.468</TotalTimeSeconds>
        <DistanceMeters>105244.0</DistanceMeters>
        <MaximumSpeed>14.508333333333333</MaximumSpeed>
        <Calories>2496</Calories>
        <AverageHeartRateBpm><Value>134</Value></AverageHeartRateBpm>
        <MaximumHeartRateBpm><Value>186</Value></MaximumHeartRateBpm>
        <Intensity>Active</Intensity>
        <Cadence>73</Cadence>
        <TriggerMethod>Manual</TriggerMethod>
        <Track>
            <!-- Kajillions of TrackPoints -->
        </Track>
        <Extensions>
          <LX xmlns="http://www.garmin.com/xmlschemas/ActivityExtension/v2">
            <MaxBikeCadence>178</MaxBikeCadence>
          </LX>
        </Extensions>
      </Lap>
    </Activity>
  </Activities>
</TrainingCenterDatabase>
{% endhighlight %}

There can be lots of laps per activity and lots of activities per file.  The sourcetype that I define below will kind of cater to multiple activities.


## Explore the data

Now lets explore what we've got

    index=fitness

Ooh that was a mistake.  Splunk created an event for every single track point because it found multiple timestamps.  Slow down and `index=fitness | delete` everything.  Upload the file and create the sourcetype.  We'll move this to the app later.


## Fix up the sourcetype

Look at the [props.conf](http://docs.splunk.com/Documentation/Splunk/6.2.3/admin/Propsconf) documentation for multiline events.  There are a few things we need to set for this sourcetype

* `TIME_PREFIX  = \<Id\>` This seems to be the case on my Garmin activities.  Not sure about other systems.
* `BREAK_ONLY_BEFORE = \<\?xml version`  I've set this to the XML declaration header.  Each file is a single event so don't break.  In the future, this will change to break for `<Activity>`.
* `MAX_EVENTS = 999999999` Splunk breaks at 256 lines.  I set this to 5 orders of magnitude higher than my biggest file
* `KV_MODE = xml` to get field extractions

Save the sourcetype as `tcx` in the app `fitness`.  I'm unsure about the TIME_PREFIX field, but we're good to go for now. The props.conf stanza is

{% highlight ini %}
[tcx]
BREAK_ONLY_BEFORE = \<\?xml version
MAX_EVENTS = 999999999
NO_BINARY_CHECK = true
TIME_PREFIX = \<Id\>
category = Structured
description = Garmin Training Center Database XML File
disabled = false
pulldown_type = true
KV_MODE = xml
{% endhighlight %}


## Data upload - round 2

Now to re-index what's in the directory.  I didn't feel like mucking about with resetting fishbucket etc so a quick powershell line

{% highlight PowerShell %}
gci | % { splunk add oneshot $_ -index fitness -sourcetype tcx }
{% endhighlight %}

Let's see what we've got

    sourcetype=tcx  | timechart count by "TrainingCenterDatabase.Activities.Activity{@Sport}"

Wahey! We're getting some data.

Looking at the actual TCX files, I'm not sure how I'm going to deal with multi-lap events.  That's a problem for later in the post.  I want to get some data out of the lap, total time, distance, calories, etc.

## Alias some fields

These are the fields that I have aliased in [props.conf](https://gist.github.com/oxo42/34d2d755027b109b986a).

    FIELDALIAS-Sport = "TrainingCenterDatabase.Activities.Activity{@Sport}" AS Sport
    FIELDALIAS-Cadence = TrainingCenterDatabase.Activities.Activity.Lap.Cadence AS Cadence

Now for something I haven't been able to discover previously.  How has my cadence changed over time?

    index=fitness sourcetype=tcx Sport=Biking Cadence=* | timechart span=mon avg(Cadence)

This is a good start.  I'll look at pulling some information about the track points apart.  I want to try to get parity with what garmin connect shows as statistics.
