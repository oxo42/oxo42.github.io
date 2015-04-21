---
layout: post
category: splunk
title: "Timechart calculated value with group by clause"
tags: [splunk, timechart]
---

    sourcetype=transactions
    | bucket _time span=5m
    | stats count as total, count(eval(response=00)) as approved by _time region
    | eval perc=approved/total*100
    | timechart avg(perc) avg(total) by region
