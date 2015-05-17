---
layout: post
category: splunk
title: Splunk my Life
tags: [splunk, garmin, fitness]
---

Sync garmin activities to Dropbox with Tapiriik

Start by creating a `fitness` app.

* Create index in splunk `fitness`

    cd C:\users\Me\DropBox\Apps\tapiriik
    splunk add monitor . -index fitness -sourcetype tcx


Now lets explore what we've got

    index=fitness

Ooh that was a mistake.  It's created an event for every single track point.  Slow down and delete `index=fitness | delete` everything.  Upload the file and create the sourcetype.  We'll move this to the app later.

Look at the [props.conf](http://docs.splunk.com/Documentation/Splunk/6.2.3/admin/Propsconf) documentation for multiline events.  There are a few things we need to set for this sourcetype

* `TIME_PREFIX  = \<Id\>` This seems to be the case on my Garmin activities.  Not sure about other systems.
* `BREAK_ONLY_BEFORE = \<\?xml version`  I've set this to the XML declaration header.  Each file is a single event so don't break
* `MAX_EVENTS = 999999999` Splunk breaks at 256 lines.  I set this to 5 orders of magnitude higher than my biggest file

Save the sourcetype as `tcx` in the app `fitness`.  I'm unsure about the TIME_PREFIX field, but we're good to go for now.

Now to re-index what's in the directory.  I didn't feel like mucking about with resetting fishbucket etc so a quick powershell line

    gci | % { splunk add oneshot $_ -index fitness -sourcetype tcx }

Let's see what we've got

    `index=fitness sourcetype=tcx | timechart count by Sport`

Wahey! We're getting some data.

Looking at the actual TCX files, I'm not sure how I'm going to deal with multi-lap events.  That's a problem for later in the post.  I want to get some data out of the lap, total time, distance, calories, etc.

I'm going to do a lot of these extractions so it's worthwhile making a macro `tcx_extract`.  I'll figure out how (or if) you can extract them automatically:

    spath path=TrainingCenterDatabase.Activities.Activity.Lap.TotalTimeSeconds output=LapTime
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.DistanceMeters output=Distance
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.MaximumSpeed output=MaxSpeed
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.Calories output=Calories
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.AverageHeartRateBpm.Value output=AvgHr
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.MaximumHeartRateBpm.Value output=MaxHr
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.Intensity output=Intensity
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.Cadence output=Cadence
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.TriggerMethod output=TriggerMethod
    | spath path=TrainingCenterDatabase.Activities.Activity.Lap.Extensions.LX.MaxBikeCadence output=MaxBikeCadence

Note that I start the macro without a `|`.  This is because I want to make searches look like `sourcetype=tcx Sport=Biking | \`tcx_extract\``

Now for something I haven't been able to discover previously.  How has my cadence changed over time?

    index=fitness sourcetype=tcx Sport=Biking | `tcx_extract`| search Cadence=* | timechart span=mon avg(Cadence)

This is a good start.  Tune in for round 2 later when I'll deal with multi-lap activities and multi-activity events (Triathlon)
