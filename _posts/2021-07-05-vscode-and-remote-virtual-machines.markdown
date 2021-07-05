---
layout: post
title:  "SSHing into a VM hosted on a remote machine, i.e. how to use VSCode with a remotely hosted VM"
date:   2021-07-05 17:00 +0100
categories: jekyll update
---

Did you ever want to develop software in a VM that is hosted on a remote machine?  
If so, you probably didn't consider this being a problem when using an editor like vim, which you install right on site.
However, I try to use VSCode with the remote extension, and it expects to be able to directly connect via some SSH config into the target server.  
Specifically, my setup is as follows: I host a VM with [Vagrant](https://www.vagrantup.com/) (called `Y`) on a remote server (called `X`) and want to use VSCode on my local machine to connect to
`Y`.

Now, Vagrant in its default setting hosts a VM on `localhost:2222` which you can then  
ssh to via `$ vagrant ssh`. With `$ vagrant ssh-config` you will get the ssh config used to
connect to it. It looks something like this, which works well from the remote machine:
```config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /path/to/some/generated/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```
By pasting it into your `~/.ssh/config` you can then use `$ ssh default` to connect to it.
However, this only works on the remote machine where you set the VM up.
In order to connect to it from the outside, you need to somehow connect to the remote machine `X` at port
`2222`, which is likely closed to the outside.
The solution to this problem is a `ProxyJump`. `ProxyJump`s connect you to a remote machine via an intermediate "gateway" machine. The only unusual thing here is that the gateway machine is the same as the
target, only a different port.

So I tried something like this. Notice that I copied the private key to my local machine.
```config
# On local machine

Host X
  HostName X.com
  User nnolte
  ...

Host default
  HostName X.com
  User vagrant
  Port 2222
  ProxyJump X
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  IdentityFile path/to/copied/private_key
```

This did not work, gave me a connection refused, similar to when I tried directly connecting to port 2222.
No expert, but probably because this still counts as an external connection.
Funnily enough, swapping `HostName X.com` to `HostName localhost` turns out to be the solution:

```config
# On local machine

Host X
  HostName X.com
  User nnolte
  ...

Host default
  HostName localhost # NOT X.com
  User vagrant
  Port 2222
  ProxyJump X
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  IdentityFile path/to/copied/private_key
```
✨✨✨✨  
Cool, this worked. I am no expert on SSH config, but I did not expect `localhost` to be interpreted "relative" to the `ProxyJump`.

With this setup, you can connect to `default` from your local machine and use VSCode with the Remote extension as usual.
