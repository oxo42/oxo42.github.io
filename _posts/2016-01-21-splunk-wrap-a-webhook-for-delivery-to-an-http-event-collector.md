---
layout: post
title: "Splunk: Wrap a webhook for delivery to an HTTP Event collector"
description: ""
category: splunk
tags: [splunk, fogbugz, http event collector]
---
{% include JB/setup %}

It all started with [FogBugz WebHooks](http://help.fogcreek.com/10800/webhooks).  I wanted to bring them into [Splunk](http://splunk.com) and using the [Splunk HTTP Event Collector](http://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector) seems like a good fit.

There were a few problems with sending the raw data to a Splunk HTTP event collector:

1. There was no way of configuring the Authorization header in the FogBugz webhook
2. The Splunk event collector expects the event data to be in an `event` field

### FogBugz WebHook

{% highlight json %}
{"eventtype":"CaseEdited", "casenumber":"123", "caseeventid":"2345", ...}
{% endhighlight %}

### Required Splunk HTTP Event Collector Event

{% highlight json %}
{"event": {"eventtype":"CaseEdited", "casenumber":"123", ...} }
{% endhighlight %}

I already have Nginx running on a reverse proxy (with SSL) sitting in front of my search head.  What I want to do is take the FogBugz webhook data, wrap it into a Splunk event, add the Auth header and post to the Splunk Forwarder running on the same box. Like this:

![Webhook Architecture](/assets/FogBugzWebhookArchitecture.png)

## 1. Splunk Forwarder

First, setup the HTTP input.  I have created an app that I push out to the Nginx server with my deployment server

{% highlight ini %}
[http]
disabled = 0

[http://fogbugz-webhook]
disabled = 0
index = main
indexes = main
sourcetype = fogbugz
token = SOME-GUID
{% endhighlight %}

When you restart the forwarder, this will start the HTTP Event Collector listening on port 8088.

## 2. Nginx
Next up you need to modify the Nginx config file to add the location `/fogbugz-GUID`.  I put a GUID in the location just to make it unguessable.

{% highlight nginx %}
location /fogbugz-GUID {
    proxy_pass            https://localhost:8088/services/collector;
    proxy_read_timeout    90;
    proxy_connect_timeout 90;
    proxy_redirect        off;
    proxy_set_header      Host $host;
    proxy_set_header      X-Real-IP $remote_addr;
    proxy_set_header      X-Forwarded-For $proxy_add_x_forwarded_for;

    # wrap the fogbugz webhook body for splunk
    proxy_set_body        "{\"event\":$request_body}";
    # Add the Splunk token into the Authorization header
    proxy_set_header      Authorization "Splunk SOME-GUID";
  }
{% endhighlight %}

* `service nginx reload` will enabled the location
* [Nginx-Proxy.conf](https://gist.github.com/oxo42/ae3c8a0d4c05749aed50) is a gist of my anonymised nginx config file that is running in production

## 3. FogBugz

Create the webhook in FogBugz to point to your shiny new location.

![FogBugz Webhook Setup](/assets/FogBugzWebhookSetup.png)

### Notes

* There was another issue of opening up the firewall and maintaining SSL certificates which was solved by my solution
* I can't get the event time to line up with `eventtime` in the webhook data.  If anyone can help, please holler
