---
layout: post
category : deployment
author: Lucas Natraj
tags: [quick, tutorial, linux, unix, nix]
title: Shell Cheat Sheet
---

### Find Port Listener

```bash
# lsof -n -i[46][protocol][@hostname|hostaddr][:service|port] ...
lsof -n -i:$PORT | grep LISTEN
```

### Kill Port Listener

```bash
kill $(lsof -n -i:$PORT | grep LISTEN | awk '{print $2}')
```

### Display Available Space

```bash
# -a   Show all mount points, including those that were mounted with the MNT_IGNORE flag.
# -H   "Human-readable" output.
df -a -H
```

### Clear File Contents

```bash
cat /dev/null > logfile
truncate logfile --size 0
```

### Find File

```bash
find . -name filename
```

### Basic Authentication Base64 Encoding
```bash
 echo -n "Aladdin:OpenSesame" | base64
```

### GSUtil - Upload all files from a folder to a bucket
```bash
gsutil -m rsync -d -r srcFolder gs://zippo
```


### View Symbolic Link Path
```bash
# Option 1
readlink [path-to-link]

# Option 2
find [path-to-link] -ls
```