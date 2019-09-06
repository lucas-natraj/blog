---
layout: post
category : python
author: Lucas Natraj
tags: [quick, tutorial, debugging, cheat]
title: Python Cheat Sheet
---

### Check if a list of keys is present in a dictionary

```Python
mandatory_keys = ['source', 'id', 'target']
if all (key in obj for key in mandatory_keys):
    print "Yes"
```