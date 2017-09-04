---
layout: post
title: Fatal The remote end hung up unexpectedly
date: 2017-09-04 17:26
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

Whenever you meet a `fatal: The remote end hung up unexpectedly` error during `git clone`, please check through following steps:

- Execute following command to better understand your error:
```sh
GIT_CURL_VERBOSE=1 GIT_TRACE_PACKET=1 GIT_TRACE=1 git clone http://xxx
```
> Usually, you'll get some results as following:
```
error: RPC failed; result = 18, HTP code = 200
```

- Try to increase the value of `http.postBuffer`:
```sh
git config --global http.postBuffer 524288000
```
> Note: if you're cloning a https repository, please increase the value of `https.postBuffer` .

- If your repository is hosting on `gitlab`, you may need to increase the `unicorn[worker_timeout]` in `/etc/gitlab/gitlab.rb`. [See this issue](https://gitlab.com/gitlab-com/support-forum/issues/246#note_1978829).