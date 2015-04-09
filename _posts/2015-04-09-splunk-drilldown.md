---
layout: post
title:  "Splunk: Dashboard Drilldowns"
date:   2015-04-09 09:00:00
categories: splunk
tags: splunk dashboard drilldown
---
{% include JB/setup %}

This is a note to future-me on how to do row based drill downs on tables in a Splunk dashboard.

Set a token
===========

Set the `drilldown` to `row` on the source table and set a `token` based on one of the clicked values, e.g. in a `<table>`, you could do:

{% highlight xml %}
<option name="drilldown">row</option>
<drilldown>
    <set token="travel_type">$row.type$</set>
</drilldown>
{% endhighlight %}

The token can be used in further panels.

Setting the `depends` attribute of a panel downstream will hide it until the token has been populated

{% highlight xml %}
<chart depends="$travel_type$">
{% endhighlight %}


Open a link
===========

Another type of drilldown is for a link (hint: link to a dashboard in your own instance with prepopulated tokens):
{% highlight xml %}
<option name="drilldown">row</option>
<drilldown>
    <link target="_blank">
        http://splunk.example.com/en-GB/app/transaction_app/merchant_query?form.merchant_name=$row.merchant$
    </link>
</drilldown>
{% endhighlight %}
