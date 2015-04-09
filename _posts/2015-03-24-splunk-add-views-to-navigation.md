---
layout: post
title:  "Splunk: Add views to the navigation menu"
date:   2015-03-24 15:00:00
categories: splunk
tags: splunk dashboard
---
{% include JB/setup %}

In the app, open the navigation menu:
* Settings
* User Interface
* Navigation menus
* Select the menu

Views
=====
* The `match` attribute is basically a string contains.  It is **NOT** a regex :(
* `source="unclassified"` will take all views that have not yet been displayed.

{% highlight xml %}
<collection label="Transactions">
  <view source="unclassified" match="transaction" />
</collection>

<collection label="Views">
  <view source="unclassified" />
</collection>
{% endhighlight %}
