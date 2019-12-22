---
layout: post
title:  "LXC full setup"
date:   2019-12-22 14:07:20 +0000
categories: lxc
---
Our goal is to setup functional unprivileged (non-root) lxc container with X11 and pulse audio support.
I assume that distribution on host is debian buster.

1. On host machine, install lxc and enable kernel support for non-root cloning containers: 
```
sudo apt-get install lxc
echo "kernel.unprivileged_userns_clone=1" > /etc/sysctl.d/80-lxc-userns.conf
sudo sysctl --system
```
2. Modify /etc/lxc/default.conf:
```
lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1
lxc.idmap = u 0 100000 65536
lxc.idmap = g 0 100000 65536
```
WARNING. With this setup apparmor throws error for me on debian buster. 
When it happen you can sacrafice a little piece of security 
with `lxc.apparmor.profile=unconfined`. 
This option disables apparmor protection for containers.

3. Create /etc/default/lxc-net:
```
USE_LXC_BRIDGE="true"
LXC_BRIDGE="lxcbr0"
LXC_ADDR="10.0.5.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.5.0/24"
LXC_DHCP_RANGE="10.0.5.10,10.0.5.30"
LXC_DHCP_MAX="20"
LXC_DHCP_CONFILE=""
LXC_DOMAIN=""
```
4. Create /etc/lxc/lxc-usernet:
```
user veth lxcbr0 10
```
5. Execute commands:
```
mkdir ~/.config/lxc
ln -s /etc/lxc/default.conf ~/.config/lxc/default.conf
chmod +x ~/.local/share/
```
6. Start services:
```
sudo systemctl enable lxc
sudo systemctl start lxc
sudo systemctl enable lxc-net
sudo systemctl start lxc-net
```
7. Execute: 
```
lxc-create -t download -n test_cont
``` 
You should see TUI with distributions and versions. Choose interesting distribution, version and arch and proceed. Alternatively you can skip TUI if you know what distro you want. 
E.g 
```
lxc-create -t download -n test_cont -- -d debian -v sid -a amd64
``` 
for x86_64 debian sid. 

8. Add this line on the bottom of container config. 
E.g ~/.local/share/lxc/test_cont/config:
```
lxc.mount.entry=/tmp/.X11-unix tmp/.X11-unix none bind,create=dir`
```

9. Start container and create user account:
```
lxc-start -n test_cont
lxc-attach -n test_cont
passwd
useradd -m -s /bin/bash -U user
passwd user
exit
```

10. On host machine install pulse audio (skip if it's already installed): 
```
sudo apt-get install pulseaudio
```

11. Uncomment or add in /etc/pulse/default.pa:
```
load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1;10.0.5.0/24
```

12. Log into container and install pulseaudio. Restart container and log on it. 
Then export this environmental variables:
```
export DISPLAY=:0.0
export PULSE_SERVER=10.0.5.1
```

13. And it's DONE. If something go wrong you can enable logging in lxc by adding `-l trace -o lxc.log`.
Additionaly you can start container in foreground by `-F` option.
E.g.
```
lxc-start -n test_cont -l trace -o test_cont.log
```
