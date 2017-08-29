---
layout: post
title: Change NuGet global package folder path
date: 2017-08-29 11:21
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

It spent me 3 hours to find a way to change the default NuGet global package folder path. Finally I end up following solution:

- Add a `NuGet.Config` to the root of the project folder.
- Add following contents to the `NuGet.Config` file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <config>
     <add key="globalPackagesFolder" value="E:\nuget\packages" />
  </config>
</configuration>
```

The most important thing is use `globalPackagesFolder` instead of `repositoryPath`. 

References:

https://github.com/NuGet/Home/issues/4341#issuecomment-274224238
https://docs.microsoft.com/en-us/nuget/schema/nuget-config-file
