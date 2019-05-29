---
layout: post
title:  Starbucks Reflection Xss Becomes Storage Xss Due To Cache
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - L3m0n</p>

漏洞点十分简单，输出是在`<script>`标签中，因为参数只对双引号做了转义出来，所以我们可以通过`</script>`进行逃逸绕过。

<img src="https://i.loli.net/2019/05/29/5cee8ca68ecd810327.png">

但是当时测试的时候最麻烦的点有两个：

1、Akamai Waf，测试发现Waf似乎还会进行简单的解码操作，当时都想`<h1>hacked</h1>`以这种HTML注入方式提交，但是想想还是算了，想要继续扩大危害。

2、访问时候出现莫名其妙的返回，也就是存在缓存，导致我一些payload打过去返回不对。不过侥幸是正因为了这个缓存，才能让反射型XSS变为了存储型XSS，并且还能在chrome上触发。

Waf绕过花了点时间，最终payload为：

{% highlight javascript %}
</script><iframe+srcdoc='%26%2360;img+src=1+onerro%26%23x000000072%26%2361;alert`1`%26%2362;'></iframe>
{% endhighlight %}

其实就是借用iframe的srcdoc中可以使用HTML编码来绕过，因为Waf还会解码，所以给HTML16进制编码的时候多加了几个，即`x000000072`

work on firefox：

<img src="https://i.loli.net/2019/05/29/5cee8ca67f55997917.png">

and work on chrome：

<img src="https://i.loli.net/2019/05/29/5cee8ca68687410637.png">