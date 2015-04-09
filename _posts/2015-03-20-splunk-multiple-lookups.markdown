---
layout: post
title:  "Splunk: Use a lookup to generate search terms in a subsearch"
date:   2015-03-20 14:57:59
categories: splunk
tags: splunk lookup
---
{% include JB/setup %}

I am collecting SNMP data using my own [SNMP Modular Input Poller][snmpmod].  I've then got a number of graphs and such coming off it.  The requirement is to build a table on a monthly basis of 95th percentile statistics for a selection of hosts and interface indexes.  The challenge is we want to display a textual name not the host and interface.  The output must look like:

| host  | ifIndex | name          |
|-------|---------|---------------|
| host1 | 1       | Peer 1 Link A |
| host1 | 2       | Peer 2 Link A |

Which I have saved to a [lookup] called `perc95_links`.

I want my search to be

    sourcetype=snmpip (host=host1 ifIndex=1) OR (host=host1 ifIndex=2) | `snmpif_traffic`
    | stats perc95(mbpsIn) as "mbpsIn" perc95(mbpsOut) as "mbpsOut" by host ifIndex
    | lookup perc95_links host ifIndex OUTPUT name

_Note the lookup on two fields :)_

Our friend to the rescue is [format].  By using the lookup as a generator

    | inputlookup perc95_links | fields host ifIndex | format

we get the output

    ( (host="host1" AND ifIndex="1" ) OR ( host="host1" AND ifIndex="2" ) )

Then we use the lookup search as a subsearch and get:

    sourcetype=snmpip [| inputlookup perc95_links | fields host ifIndex | format]] | `snmpif_traffic`
    | stats perc95(mbpsIn) as "mbpsIn" perc95(mbpsOut) as "mbpsOut" by host ifIndex
    | lookup perc95_links host ifIndex OUTPUT name

[snmpmod]:  https://github.com/oxo42/snmpmod
[lookup]:  http://docs.splunk.com/Documentation/Splunk/latest/SearchReference/lookup
[format]:  http://docs.splunk.com/Documentation/Splunk/latest/SearchReference/format
