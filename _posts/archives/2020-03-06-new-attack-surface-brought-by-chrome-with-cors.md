---
layout: post
title:  New attack surface brought by Chrome with CORS
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - todaro</p>

## 导读

如果CORS配置Access-Control-Allow-Origin: * (或者Access-Control-Allow-Origin: hacker.com)，即使配置Access-Control-Allow-Credentials: false，攻击者还是可以通过chrome浏览器的缓存功能 (force-cache)读取到用户的敏感信息 (chrome浏览器不会修复这个问题)。


## CORS及SOP简介

对CORS的介绍要从浏览器的同源策略开始说起，SOP全称为Same Origin Policy即同源策略，该策略是浏览器的一个安全基石，同源策略规定: 不同域的客户端脚本在没有明确授权的情况下，不能读写对方的资源。简单来说同源策略就是浏览器会阻止一个源与另一个源的资源交互。可以试想一下，如果没有同源策略，当你访问一个正常网站的时候又无意间打开了另一个恶意网站，恶意网站会从你刚刚访问的正常网站上窃取你全部的信息。

SOP是一个很好的策略，在SOP被提出的时期，大家都默默地遵守着这个规定，但随着WEB应用的发展，有些网站由于自身业务的需求，需要实现一些跨域的功能能够让不同域的页面之间能够相互访问各自页面的内容。为了实现这个跨域需求，聪明的程序员想到了一种编码技术JSONP，该技术利用从客户端传入的json格式的返回值，在服务器端调用该接口处事先以定义函数的方式定义好json格式里参数值，并加载script标签调用该函数实现跨域。由此可见JSONP虽然好但他并非是在协议层面解决跨域问题，所以出现了很多安全问题。为了能更安全的进行跨域资源访问，CORS诞生了。

CORS是H5提供的一种机制，WEB应用程序可以通过在HTTP报文中增加特定字段来告诉浏览器，哪些不同来源的服务器是有权访问本站资源。

## CORS已经存在的问题

CORS在设置Access-Control-Allow-Origin: hacker.com且Access-Control-Allow-Credentials: true的情况下，hacker.com可以读取用户在网站的敏感信息。

## CORS配合Chrome浏览器产生的新问题

如果CORS配置Access-Control-Allow-Origin: * (或者Access-Control-Allow-Origin: hacker.com)，攻击者可以通过chrome浏览器的缓存功能读取到用户的敏感信息。

Chrome的force-cache: 浏览器在HTTP缓存中查找匹配的请求。

{% highlight html %}
如果有新的或旧的匹配，它将从缓存中返回。
如果不匹配，浏览器将发出正常的请求，并用下载的资源更新缓存。
{% endhighlight %}

与之相似的还有only-if-cached: 浏览器在HTTP缓存中查找匹配的请求。

{% highlight html %}
如果有新的或旧的匹配，将从缓存返回。
如果不匹配，浏览器将返回一个错误。
{% endhighlight %}

force-cache和only-if-cached的差别在于，后者要求请求者和资源者需同源，否则会弹出如下错误:

![1.png](https://i.loli.net/2020/03/06/4mKsLNHnIYAz6OE.png)

所以在Chrome的force-cache下，攻击者可以通过读取用户在浏览器的缓存来获取敏感信息。

Chrome开发团队认为缓存就是这样的工作机制，他们不会对这个问题进行修复，而是要求开发者使用"Vary"头部或者用户使用"--enable-features=SplitCacheByNetworkIsolationKey"启动浏览器:

![2.png](https://i.loli.net/2020/03/06/zbLrBYnq2DJlVk5.png)

火狐浏览器在上述场景中则不会返回敏感信息。

## CORS配合Chrome的几个测试案例

1. 在最新版Chrome浏览器 (80.0.3987.132)中测试以下几种CORS配置:

{% highlight html %}
 (1)	Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: false
 (2)	Access-Control-Allow-Origin: hacker.com
Access-Control-Allow-Credentials: true
 (3)	Access-Control-Allow-Origin: hacker.com
Access-Control-Allow-Credentials: false
{% endhighlight %}

2. 在最新版Firefox浏览器 (73.0.1)中测试如下CORS配置:

{% highlight html %}
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: false
{% endhighlight %}

网站 http://192.168.31.154:8010/ 存在三种环境分别对应上述的CORS配置:

{% highlight html %}
http://192.168.31.154:8010/index.php?type=1 对应 (1)
http://192.168.31.154:8010/index.php?type=2 对应 (2)
http://192.168.31.154:8010/index.php?type=3 对应 (3)
{% endhighlight %}

![3.png](https://i.loli.net/2020/03/06/pKD31LuNCgx6EHa.png)

如图，输入admin/admin后会登录到用户账户，然后根据type=1跳转到/secret.php，type=2跳转到/secret2.php，type=3跳转到/secret3.php，都会输出登录用户的$_COOKIE['auth']到当前页面:

![4.png](https://i.loli.net/2020/03/06/WDoX2LZcrE7U86p.png)

![5.png](https://i.loli.net/2020/03/06/Non1BGvFRVp9lIb.png)

![6.png](https://i.loli.net/2020/03/06/viq9mEYFaWtweG6.png)

另外一个网站 http://192.168.31.154:8001/ 用来模拟攻击者网站/poc.html负责读取用户的secret值，如果能通过跨域读取到该值，则输出到RESULT框中:

![7.png](https://i.loli.net/2020/03/06/PxQYmWSJEzOpCcH.png)

测试正式开始！

[1] 先访问 http://192.168.31.154:8010/index.php?type=1 输入admin/admin跳转到 http://192.168.31.154:8010/secret.php 能够正确输出secret值:

![8.png](https://i.loli.net/2020/03/06/sUwx5PH4qzemSKG.png)

在1 (1) 模式下通过CORS配置问题+Chrome浏览器缓存读取secret值 ("Disable cache"默认关闭)，点击"Steal secret":

![9.png](https://i.loli.net/2020/03/06/S6oQ3WEzYmcxHt9.png)

可以看到请求包不带cookie访问:

![10.png](https://i.loli.net/2020/03/06/egwXZnU38bJjipT.png)

返回包如下:

![11.png](https://i.loli.net/2020/03/06/esqoH7lgckDR2Tp.png)

成功跨域读取secret值并显示在RESULT中:

![12.png](https://i.loli.net/2020/03/06/yZ7MSV5qDoEKRXB.png)

在Chrome中按Ctrl+Shift+i，在"Network"中勾选"Disable cache"禁止缓存功能:

![13.png](https://i.loli.net/2020/03/06/517tDGpSqNcAHIW.png)

刷新 http://192.168.31.154:8001/poc.html 重新点击"Steal secret"，无法读取到secret值。

如果再次关闭"Disable cache"，需要在登录状态下重新访问一次 http://192.168.31.154:8010/secret.php 让浏览器缓存该结果，之后才能重新读取secret的值。

[2] 访问 http://192.168.31.154:8010/index.php?type=2 输入admin/admin跳转到 http://192.168.31.154:8010/secret2.php 能够正确输出secret值:

![14.png](https://i.loli.net/2020/03/06/l5uafv8eop1iUst.png)

在1 (2)模式下通过常规CORS配置漏洞读取secret值，点击"Steal secret":

![15.png](https://i.loli.net/2020/03/06/i9bHR16C5sGqJP4.png)

可以看到请求包带cookie访问:

![16.png](https://i.loli.net/2020/03/06/zRFt62qeBXpxgaQ.png)

返回包如下:

![17.png](https://i.loli.net/2020/03/06/753pQYfzGdOFqBH.png)

成功跨域读取secret值并显示在RESULT中:

![18.png](https://i.loli.net/2020/03/06/XG2sxQSkumMLhda.png)

这就是一个经典的CORS配置错误导致的跨域敏感信息读取。

[3] 访问 http://192.168.31.154:8010/index.php?type=3 输入admin/admin跳转到 http://192.168.31.154:8010/secret3.php 能够正确输出secret:

![19.png](https://i.loli.net/2020/03/06/78yXRzlnOEqHr5h.png)

在1 (3)模式下尝试通过CORS读取secret值，点击"Steal secret":

![20.png](https://i.loli.net/2020/03/06/osNL8i4RxzdfmZK.png)

可以看到console的报错信息:

![21.png](https://i.loli.net/2020/03/06/zS4TZmBoUNl6Ajf.png)

因为Access-Control-Allow-Credentials: false:

![22.png](https://i.loli.net/2020/03/06/4OD6wa5qGi3SXKe.png)

所以尝试读取secret值失败。

[4] 在Firefox浏览器中先访问 http://192.168.31.154:8010/index.php?type=1 输入admin/admin跳转到 http://192.168.31.154:8010/secret.php 能够正确输出secret:

![23.png](https://i.loli.net/2020/03/06/dBQlAS7sRxretf9.png)

在2模式下尝试通过CORS配置问题+Firefox浏览器缓存读取secret值，点击"Steal secret":

![24.png](https://i.loli.net/2020/03/06/sDPuiHAVQRJyXIq.png)

http://192.168.31.154:8001/poc.html 的内容需要稍作修改:

{% highlight html %}
将第38行const secret = htmlDoc.getElementsByName("secret")[0].text修改为
const secret = htmlDoc.getElementsByName("secret").text
{% endhighlight %}

可以看到请求包不带cookie访问:

![25.png](https://i.loli.net/2020/03/06/lFK4ZQhUwsyOqBY.png)

返回包如下:

![26.png](https://i.loli.net/2020/03/06/zqO2jLseSFGBgnk.png)

无法读取到secret的值:

![27.png](https://i.loli.net/2020/03/06/DSGX3fuyLnNZFR8.png)

如果在Firefox中通过1 (2) 的模式 (即经典CORS漏洞配置问题) 则能读取到secret的值:

![28.png](https://i.loli.net/2020/03/06/aTLXf4BGekPQty5.png)

## 真实案例

Keybase提供了一个API，该API允许用户对其他用户进行查找，但是没有做token检验。当已登录用户访问该接口时会返回用户的敏感信息。

由于Keybase的CORS规则配置成如下:

{% highlight html %}
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET
Access-Control-Allow-Headers: Content-Type, Authorization, Content-Length, X-Requested-With
Access-Control-Allow-Credentials: false
{% endhighlight %}

所以攻击者可以配合Chrome浏览器缓存问题跨域读取该值，最终payload如下:

{% highlight html %}
<html>
<script>
var url = "https://keybase.io/_/api/1.0/user/lookup.json?username={YOUR_USERNAME}";
fetch(url, {
    method: 'GET',
    cache: 'force-cache'
    });
</script>
</html>
{% endhighlight %}

用户一旦访问该页面，攻击者即可获取到其敏感信息。

## 一个延伸

以往在测试一些形似jsonp接口却不带callback时可能就只有放弃了，现在可以看看是不是有CORS配置Access-Control-Allow-Origin: *的存在，还有一些app里的H5场景。

## 总结

CORS如果配置Access-Control-Allow-Origin: * (或者Access-Control-Allow-Origin: hacker.com)，攻击者可以通过chrome浏览器的缓存功能 (force-cache) 读取到用户的敏感信息。

开发者需要设置Access-Control-Allow-Origin: company.com，或者使用"Vary"头部。

普通用户如果没有特殊需求，可以给Chrome浏览器启动添加"--enable-features=SplitCacheByNetworkIsolationKey"参数，或者勾选"Disable cache"。

有两个线上测试环境:

* <a href="https://victime.docker.bi.tk/" target="__blank">https://victime.docker.bi.tk/</a>
* <a href="http://87.98.164.31/malicious.html" target="__blank">http://87.98.164.31/malicious.html</a>

测试代码打包在<a href="https://statics.symbo1.com/file/symbo1/misc/files.rar" target="__blank">files.rar</a>中。

参考信息

{% highlight html %}
https://enumerated.wordpress.com/2019/12/24/sop-bypass-via-browser-cache/
https://bugs.chromium.org/p/chromium/issues/detail?id=988319
https://github.com/MayurUdiniya/Chrome-CORS
https://www.freebuf.com/company-information/216754.html
https://www.w3cschool.cn/fetch_api/fetch_api-hokx2khz.html
{% endhighlight %}