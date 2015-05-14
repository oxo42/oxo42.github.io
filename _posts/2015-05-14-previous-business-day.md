---
layout: post
category: splunk
title: Get results for the previous business day
tags: [splunk, search]
---

Save the following as a macro `previous_business_day` and export it globally

    [| stats count
    | eval day_of_week=tonumber(strftime(now(), "%w"))
    | eval earliest=case(day_of_week==0, "-2d@d", day_of_week==1, "-3d@d", 1==1, "-1d@d")
    | eval latest=case(day_of_week==0, "-1d@d", day_of_week==1, "-2d@d", 1==1, "@d")
    | return earliest, latest]

Then run your search

    sourcetype=foo `previous_business_day`
