---
layout: post
title: curl Unsupported Request Method and Protocol
date: 2017-10-26 10:03
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# A weird issue

Recenlty, I've met a weird issue when using `curl` to access a sftp server. I just execute following simple curl command:

```sh
curl -u sftptest:passwd sftp://127.0.0.1:22/home/sftptest/
```

I got a large html response. The key information is:

```
...
<title>ERROR: The requested URL could not be retrieved</title>
...
<p><b>Unsupported Request Method and Protocol</b></p>
```

I pretty much sure the `sftp` support has been enabled in `curl`. I just ran `curl -V` to confirm it.

Next I open the `verbose` option and found interesting messages:

```sh
*   Trying 172.17.10.80...
* TCP_NODELAY set
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to (nil) (172.17.18.84) port 8080 (#0)
* Server auth using Basic with user 'sftptest'
> GET sftp://sftptest:password@127.0.0.1/home/sftptest HTTP/1.1
> Host: 127.0.0.1:22
> Authorization: Basic c2Z0cHRlc3Q6MXEydzNlNHI=
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.52.1 OpenSSL/1.1.0e libssh2/1.7.0_DEV
> Accept: */*
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 400 Bad Request
< Server: nps/2.3.1
< Mime-Version: 1.0
< Date: Wed, 25 Oct 2017 10:03:23 GMT
< Content-Type: text/html
< Content-Length: 4205
< X-Squid-Error: ERR_INVALID_URL 0
< Vary: Accept-Language
< Content-Language: en
< X-Cache: MISS from netentsec-nps-172.17.18.84
< Connection: close
< 
{ [data not shown]
* Curl_http_done: called premature == 0

100  4205  100  4205    0     0  29450      0 --:--:-- --:--:-- --:--:-- 29822
* Closing connection 0
```

You'll notice that the curl will try to connect my proxy server first (172.17.10.80), then it recognize `sftp://home/sftptest` as `http` protocol! That's why we got a bad request(400) response and an error `Unsupported Request Method and Protocol`.

If we bypass the `127.0.0.1` from the proxy (e.g. `export no_proxy=127.0.0.1`), the request simply works.

That seems like a bad design of curl. It should never change the request protocol even behind a proxy.

# Conclusion

Add your sftp url to `no_proxy` so that `curl` will never try to change the request protocol to http.