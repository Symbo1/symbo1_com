---
layout: post
title: Osx Apache Code To Disclose
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - CongRong</p>

## 漏洞分析

&nbsp;&nbsp;&nbsp;最开始是在本地发现的，在请求文件名中插入BOM头字符会直接返回后端的代码：

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/03235511c2a5182171f420e55a9eef04fdd2ad02.png">

当时此漏洞的描述是：

&nbsp;&nbsp;&nbsp;The Apple mod_hfs_apple module for the Apache HTTP Server in Apple Mac OS X before 10.8.3 does not properly handle ignorable Unicode characters, which allows remote attackers to bypass intended directory authentication requirements via a crafted pathname in a URI.

&nbsp;&nbsp;&nbsp;意思就是运行Apache的Osx服务器上，数据取自HFS+文件系统，HFS+用unicode来命名文件或文件夹。根据HFS+特性，当视图向HFS+请求文件时，接收到的这个文件名中含有可被HFS+忽略掉的unicode序列的话，照常返回这个正常文件名的文件。而作为apache for mac在处理这种请求时，导致请求的文件没有在服务器上执行，也就是将所请求的文件当做普通文件来响应请求。

比如这些空的unicode字符序列都会被HFS+忽略：

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/03235042f97be2f10da2075940c99257a272074d.png">

再来看看Apple的<a target="_blank" href="https://support.apple.com/zh-cn/HT202694">公告</a>:

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/4fe.png">

&nbsp;&nbsp;&nbsp;Apple对这个漏洞的定义只是绕过了HTTP基本认证，过于模棱两可。所以我把这个漏洞的标题定义为”Code To Disclose Vulnerability”。

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/0421155498fc6f652b4f0e6d44d51a3dabfe40ad.png">

&nbsp;&nbsp;&nbsp;这个漏洞的描述是说`mod_hfs_apple`模块没有正确处理掉这些unicode序列，这个模块的最初意义是用来处理URI请求信息的，比如文件名大小写等等事情，并且是用户自愿启不启用的。所以补丁就打在了`mod_hfs_apple`模块：

{% highlight c %}

static int contains_ignorable_sequence(char* s) {

size_t len = strlen(s);

if (len <= 2) return 0;

for (size_t i = 0; i <= len - 2; i++) {

	// 2-char sequences

	if (s[i] == '\xFC' && '\x80' <= s[i+1] && s[i+1] <= '\x83') return 1;

	if (s[i] == '\xF8' && '\x80' <= s[i+1] && s[i+1] <= '\x87') return 1;

	if (s[i] == '\xF0' && '\x80' <= s[i+1] && s[i+1] <= '\x8F') return 1;

	if (i <= len - 3) {

		// 3-char sequences

		if (s[i] == '\xEF' && s[i+1] =='\xBB' && s[i+2] =='\xBF') return 1;

		if (s[i] == '\xE2' && s[i+1] =='\x81' && '\xAA' <= s[i+2] && s[i+2] <= '\xAF') return 1;

		if (s[i] == '\xE2' && s[i+1] =='\x80' && (('\x8C' <= s[i+2] && s[i+2] <= '\x8F') || ('\xAA' <= s[i+2] && s[i+2] <= '\xAE'))) return 1;

	}

}

return 0;

}

{% endhighlight %}

`contains_ignorable_sequence`函数将这些可被HFS+忽略的unicode序列集中了起来：

{% highlight c %}

\xFC[\x80-\x83]
\xF8[\x80-\x87]
\xF0[\x80-\x8F]
\xEF\xBB\xBF
\xE2\x81[\xAA-\xAF]
\xE2\x80[\x8C-\x8F\xAA-\xAE]

{% endhighlight %}

在`hfs_apple_module_fixups`函数中调用了`contains_ignorable_sequence`：

{% highlight c %}

if (contains_ignorable_sequence(r->filename)) {

	ap_log_rerror(APLOG_MARK, APLOG_NOERRNO|APLOG_ERR, 0, r,

			  "mod_hfs_apple: URI %s has ignorable character sequence. Denying access.",

			  r->filename);

	return HTTP_FORBIDDEN;

}

{% endhighlight %}

&nbsp;&nbsp;&nbsp;当URI中的filename存在这些字符序列时，则返回为这个请求是禁止的。更有意思的事情是，补丁虽然打了，但Apple还是没有将mod_hfs_apple模块设为默认加载，在虚拟机上安装了OS X Yosemite 10.10来看看。

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/01151244817cb1ad2440184c83afe16bba26039f.jpg">

&nbsp;&nbsp;&nbsp;直到去年有安全研究员提出了”osx apache没有默认加载mod_hfs_apple模块那么打补丁有什么意义？” CVE-2015-3675就这么诞生了2333。

&nbsp;&nbsp;&nbsp;The default configuration of the Apache HTTP Server on Apple OS X before 10.10.4 does not enable the mod_hfs_apple module, which allows remote attackers to bypass HTTP authentication via a crafted URL.

https://support.apple.com/zh-cn/HT204942

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/4444.png">

修补的方式就是默认加载了。我本机是OSX EI Caption 10.11.6，自带的Apache默认加载了这个模块：

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/6tt.png">

## 漏洞蔓延

&nbsp;&nbsp;&nbsp;根据服务+操作系统+第三方包指纹搜索，发现在公网架设Apache服务的Osx系统中，用集成包的要比用系统原生的多，MAMP这几个就犹如win下的php study等等第三方包一样流行，MAMP在Bitnami上也很流行，这就很尴尬了。

MAMP & MAMP Pro(2015-12-14):

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/011519390827084e57aa7c78373b8faef228197b.png">

XAMPP(2016-04-21):

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/011523524d67e25007dfc50ca6557b29820cfa3f.png">

AMPPS(2015-01-03):

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/011541228e7c5de5424f3967e04b1dbe1554c098.png">

Payload：

{% highlight c %}
%EF%BB%BF
%E2%81%AA
%E2%81%AB
%E2%81%AC
%E2%81%AD
%E2%81%AE
%E2%81%AF
%E2%80%AE
%E2%80%8C
{% endhighlight %}

MAMP & MAMP Pro：

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/1.png">

AMPPS：

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/2.png">

XAMPP:

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/3.png">

## 写在最后

&nbsp;&nbsp;&nbsp;虽然第三方包居多，但是想找得更全很难，毕竟是中间件的第三方包，很多站建起来就不好找是不是用这个第三方包搭建的。此漏洞完整的生命线就到这儿了，感兴趣的可以继续研究下补丁。