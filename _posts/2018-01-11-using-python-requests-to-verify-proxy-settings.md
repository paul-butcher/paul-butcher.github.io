---
layout: post
title: Using Python requests to verify proxy settings
categories: python
---

Using [requests.utils](http://docs.python-requests.org/en/master/_modules/requests/utils/), you can check whether a connection to a given URL will go through a proxy.

When configuring an environment, it is always possible that some settings may go awry, perhaps they have been assigned to a particular user, rather than globally;  maybe they are completely misconfigured.  If there is no validation or verification applied to a given value, then small errors could lead to big problems.

One such situation is when configuring an environment that accesses the internet through a proxy, but bypasses that proxy internally.  If you fail to set up a proxy, then it cannot access external APIs.  If you fail to configure `NO_PROXY` correctly, then it can place undue load on your proxy, or even completely fail to work.

[Requests](http://docs.python-requests.org/) offers ways to test some of your proxy configurations without making requests.  

You can examine the settings using `requests.utils.getproxies`, thus:

 ```
 >>> requests.utils.getproxies()
{
    'http': 'http://proxy.example.com/', 
    'https': 'https://proxy.example.com', 
    'no': 'localhost,127.0.0.1,.internal.example.com'
}
```

You can also check whether a given URL will go through the proxy, with `requests.utils.should_bypass_proxies`, thus:

```
>>> requests.utils.should_bypass_proxies('http://external.example.com', None)
True

>>> requests.utils.should_bypass_proxies('http://internal.example.com', None)
False
```


One thing to note is that `should_bypass_proxies` checks no_proxy independently of 
http_proxy and https_proxy.  It will return False as long as the URL is not matched by no_proxy, regardless of whether there is a proxy for it to bypass.

To perform the check in one step, you can use `select_proxy`, however, you will also need to pass in the proxy information as a dict, thus.  
```
>>> proxies = requests.utils.get_environ_proxies('http://example.com')
>>> requests.utils.select_proxy('http://example.com', proxies)
'http://proxy.example.com/'
>>> del proxies['http']
>>> requests.utils.get_environ_proxies()
```

This is ideal for a post-install check in such an application, perhaps to be exercised by an [ansible smoke test]({% post_url 2017-10-15-ansible-smoke-testing %})