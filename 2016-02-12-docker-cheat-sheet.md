---
layout: post
category : deployment
author: Chase Jenkins & Lucas Natraj
tags: [quick, tutorial, docker]
title: Docker Cheat Sheet
---

## Docker Tips

- Switching networks (home, work, wifi, lan, ...) may cause docker-machine to fail to pull images. A simple `docker-machine restart <machine-name>` 

## VirtualBox

### Port Forwarding
[VirtualBox Documentation](https://www.virtualbox.org/manual/ch06.html#network_nat)

```bash
# Forward tcp requests on the port 8040 (Host) to port 8030 (VirtualBox).
# VBoxManage controlvm <uuid|vmname> natpf<1-N> [<rulename>],tcp|udp,[<hostip>],<hostport>,[<guestip>],<guestport>
VBoxManage controlvm "default" natpf1 "my_service,tcp,,8040,,8030";
```

### List VMs / VM Info

```bash
# VM List
VBoxManage list vms

# VM Info
VBoxManage showvminfo default
```

### Removing all containers

```bash
docker ps -aq | xargs docker rm -f
```

### Quick Python Dev Environment

```bash
docker run -it -v ~/work/project/:/project  python /bin/bash
```
