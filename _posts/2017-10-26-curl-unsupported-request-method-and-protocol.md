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

# Dig the trueth

I'm very curious is it a bug or by design. So I file [an issue](https://github.com/curl/curl/issues/2018) to seek the answer. In fact I'm using older(7.52) curl that does not support automatical tunneling. Quote from the reply of the issue:

```
Since 7.55.0, curl will enable tunneling automatically when you try to use SFTP over an HTTP proxy.
```

So I guess it should work with latest curl. Thus I downloaded and compiled the source code of the master branch of curl and try to execute my command. Unfortunately, I got another issue:

```
* STATE: INIT => CONNECT handle 0x1a42ec8; line 1425 (connection #-5000)
* Added connection 0. The cache now contains 1 members
*   Trying 172.17.18.84...
* TCP_NODELAY set
* STATE: CONNECT => WAITCONNECT handle 0x1a42ec8; line 1477 (connection #0)
* Connected to 172.17.18.84 (172.17.18.84) port 8080 (#0)
* STATE: WAITCONNECT => WAITPROXYCONNECT handle 0x1a42ec8; line 1594 (connection #0)
* Marked for [keep alive]: HTTP default
* allocate connect buffer!
* Establish HTTP proxy tunnel to 127.0.0.1:22
* Server auth using Basic with user 'sftptest'
> CONNECT 127.0.0.1:22 HTTP/1.1
> Host: 127.0.0.1:22
> User-Agent: curl/7.57.0-DEV
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 503 Service Unavailable
< Server: nps/2.3.1
< Mime-Version: 1.0
< Date: Thu, 26 Oct 2017 07:09:25 GMT
< Content-Type: text/html
< Content-Length: 4069
< X-Squid-Error: ERR_CONNECT_FAIL 111
< Vary: Accept-Language
< Content-Language: en
```

Notice that there's a `503 Service Unavailable` error from the response. Why this happened? Let's go through the output message to investigate this issue.

First of all, I found the proxy has been connected successfully:

```
*   Trying 172.17.18.84...
* TCP_NODELAY set
* STATE: CONNECT => WAITCONNECT handle 0x1a42ec8; line 1477 (connection #0)
* Connected to 172.17.18.84 (172.17.18.84) port 8080 (#0)
* STATE: WAITCONNECT => WAITPROXYCONNECT handle 0x1a42ec8; line 1594 (connection #0)
* Marked for [keep alive]: HTTP default
* allocate connect buffer!
* Establish HTTP proxy tunnel to 127.0.0.1:22
```

A tcp connection state change from `CONNECT => WAITCONNECT => WAITPROXYCONNECT => Establish HTTP proxy tunnel` does make sense.

Then it's going to connect to my sftp server:

```
* Server auth using Basic with user 'sftptest'
> CONNECT 127.0.0.1:22 HTTP/1.1
> Host: 127.0.0.1:22
> User-Agent: curl/7.57.0-DEV
> Proxy-Connection: Keep-Alive
```

However, the response is:
```
< HTTP/1.1 503 Service Unavailable
```

That means the sftp connection from the proxy server to my sftp server (`127.0.0.1:22`) failed - specifically, the service is unavailable. Now it's very clear to me: the proxy server is connecting `127.0.0.1:22`, which is proxy server itself, not my sftp server! If I change the ip address to the ethernet ip of my sftp server (`172.26.9.94`), it simply works:

```
* STATE: INIT => CONNECT handle 0x2238ec8; line 1425 (connection #-5000)
* Rebuilt URL to: sftp://sftptest@172.26.9.94/
* Added connection 0. The cache now contains 1 members
*   Trying 172.17.18.84...
* TCP_NODELAY set
* STATE: CONNECT => WAITCONNECT handle 0x2238ec8; line 1477 (connection #0)
* Connected to 172.17.18.84 (172.17.18.84) port 8080 (#0)
* STATE: WAITCONNECT => WAITPROXYCONNECT handle 0x2238ec8; line 1594 (connection #0)
* Marked for [keep alive]: HTTP default
* allocate connect buffer!
* Establish HTTP proxy tunnel to 172.26.9.94:22
* Server auth using Basic with user 'sftptest'
> CONNECT 172.26.9.94:22 HTTP/1.1
> Host: 172.26.9.94:22
> User-Agent: curl/7.57.0-DEV
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 200 Connection established
< 
* Proxy replied 200 to CONNECT request
* CONNECT phase completed!
* STATE: WAITPROXYCONNECT => SENDPROTOCONNECT handle 0x2238ec8; line 1573 (connection #0)
* CONNECT phase completed!
* SFTP 0x223f170 state change from SSH_STOP to SSH_INIT
* SFTP 0x223f170 state change from SSH_INIT to SSH_S_STARTUP
* STATE: SENDPROTOCONNECT => PROTOCONNECT handle 0x2238ec8; line 1608 (connection #0)
* SFTP 0x223f170 state change from SSH_S_STARTUP to SSH_HOSTKEY
* SSH MD5 fingerprint: 4cf03683e054a3398c91d76a16715e6b
* SSH host check: 2, key: <none>
* SFTP 0x223f170 state change from SSH_HOSTKEY to SSH_SESSION_FREE
* Marked for [closure]: SSH session free
* SFTP 0x223f170 state change from SSH_SESSION_FREE to SSH_STOP
* multi_done
* SSH DISCONNECT starts now
* SSH DISCONNECT is done
* Closing connection 0
* The cache now contains 0 members
```

# Conclusion

- Option 1 (prefered): Add your sftp ip address to `no_proxy` so that `curl` will never try to change the request protocol to http regardless of the version of curl.
- Option 2: If you cannot bypass your sftp ip address, you need to specify the ethernet ip address or domain name instead of 127.0.0.1/localhost, and make sure your curl version is 7.55+.