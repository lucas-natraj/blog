---
layout: post
category : web
author: Lucas Natraj
tags: [quick, tutorial, debugging, cheat]
title: NancyFx Cheat Sheet
---

## [NancyFx](http://nancyfx.org)


### Read the Request as a string (useful for debugging)
**Note: The stream can ONLY be read once. Subsequent reads will be empty.**

```C#
public static class RequestStreamExtensions
{
    public static string ReadAsString(this RequestStream requestStream)
    {
        using (var reader = new StreamReader(requestStream))
        {
            return reader.ReadToEnd();
        }
    }
}
```