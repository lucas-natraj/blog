---
layout: post
category : security
author: Lucas Natraj
tags: [quick, tutorial, ssh]
title: Working with SSH Keys
---

## Generating RSA SSH Key Pairs
```
$ ssh-keygen -t rsa -b 4096 -C "me@example.com"
Generating public/private rsa key pair.
Enter file in which to save the key (~/.ssh/example_rsa): ~/.ssh/example_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in ~/.ssh/example_rsa.
Your public key has been saved in ~/.ssh/example_rsa.pub.
The key fingerprint is:
SHA256:cMvqGWlKRNxJsn8symj7LF5k1u/ilZ+SgZfPgNbJykg me@example.com
The key's randomart image is:
+---[RSA 4096]----+
|    . .          |
|   . = .         |
|    + + .        |
|   . o = .       |
|    = +=So       |
|   BE.+*O.       |
|  o.=+=o+*       |
| ..=.++=o.o.     |
| .oo+.+...o      |
+----[SHA256]-----+
```

## Adding SSH Key to ssh-agent
```
# start the ssh-agent in the background
$ eval "$(ssh-agent -s)"
Agent pid 59566

# add ssh key
$ ssh-add ~/.ssh/example_rsa
Enter passphrase for ~/.ssh/example_rsa: [passphrase] 
Identity added: ~/.ssh/example_rsa (~/.ssh/example_rsa)
```

## Fingerprint
```
# SHA256 Key
$ ssh-keygen -lf ~/.ssh/example_rsa.pub
4096 SHA256:cMvqGWlKRNxJsn8symj7LF5k1u/ilZ+SgZfPgNbJykg me@example.com (RSA)

# MD5
$ ssh-keygen -E md5 -lf ~/.ssh/example_rsa.pub
4096 MD5:ee:6f:60:43:16:c0:ee:37:c7:1c:a8:fe:1f:37:95:2c me@example.com (RSA)
```

## Disable SSH Host Verification
```
$ ssh -o "StrictHostKeyChecking no" git@github.com
```
